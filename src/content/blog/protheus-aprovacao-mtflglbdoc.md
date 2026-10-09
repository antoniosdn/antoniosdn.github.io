---
title: 'Aprovar documento no Protheus sem chamar a REST do próprio servidor'
description: 'Um workflow do Protheus que já roda dentro do AppServer não precisa de uma chamada HTTP ao próprio servidor para aprovar um documento. Chamar MTFlgLbDoc direto tira uma volta inteira de rede, serialização e pontos de falha.'
pubDate: '2026-10-09'
heroImage: '../../assets/protheus-aprovacao-mtflglbdoc.png'
---

Este post é sobre o **workflow do próprio Protheus**, aquele que envia o pedido de compras
por e-mail em HTML para o aprovador responder (`TWFProcess`), e não sobre um processo Fluig.

Quando o aprovador respondia, o retorno do workflow chegava ao Protheus e, para efetivar a
aprovação, fazia uma chamada HTTP `PUT` na API REST padrão de aprovação em lote, **no
próprio servidor onde já estava rodando**.

Funcionava. Mas era uma volta longa para chegar a uma função que estava ali do lado.

## O custo da volta pela REST

O código já está dentro do AppServer, com ambiente aberto, tabelas acessíveis e o registro
da alçada posicionado. Ainda assim, para aprovar, ele:

1. monta um JSON com os dados do documento;
2. abre uma conexão HTTP (com TLS) contra o próprio servidor;
3. ocupa uma thread do serviço REST, que prepara o ambiente de novo para atender a requisição;
4. desserializa o JSON, localiza o documento outra vez e chama a rotina de liberação;
5. serializa a resposta, que volta pela rede e é interpretada pelo workflow.

Tudo isso para executar algo que o workflow poderia chamar com uma linha. Além do tempo de
cada etapa, a volta traz dependências que não precisavam existir:

- o **serviço REST** precisa estar no ar e com thread livre no momento da resposta;
- **timeout** e erro de rede viram motivo de falha de uma aprovação;
- a **URL** e a **autenticação** do endpoint precisam estar configuradas e corretas em cada ambiente;
- o erro, quando acontece, fica dividido entre o log do workflow e o log do serviço REST.

## Chamar a rotina direto

O caminho `workflow → HTTP → serviço REST → liberação do documento` pode ser
`workflow → liberação do documento`.

A função de liberação de documento de alçada do Compras é a `MTFlgLbDoc`:

```advpl
MTFlgLbDoc(cNum, cUser, cAprov, cTipo, cFluig, cParecer)
// cAprov: "1" aprova, "2" rejeita
```

A aprovação passa a rodar na mesma thread do workflow, sem rede, sem serialização e sem
depender de outro serviço. O que sobra para tratar é só o resultado da própria função.

## Dois cuidados que a chamada direta exige

**1. Gravar a marca do documento antes (`CR_FLUIG`).** A `MTFlgLbDoc` localiza a linha da
alçada na SCR comparando `CR_FLUIG` com o identificador recebido. Sem essa marca, se o
mesmo usuário tem mais de uma alçada no mesmo pedido, ela pega a primeira que estiver
pendente. A correção é localizar a linha pela chave completa (filial, tipo, número,
aprovador, grupo e item do grupo), gravar a marca e só então chamar:

```advpl
cMarca := AllTrim(Str(SCR->(RecNo())))

RecLock("SCR", .F.)
SCR->CR_FLUIG := PadR(cMarca, TamSX3("CR_FLUIG")[1])
SCR->(MsUnlock())

lRet := MTFlgLbDoc(PadR(cNumero, TamSX3("CR_NUM")[1]), ;
                   AllTrim(SCR->CR_USER), cAprov, AllTrim(SCR->CR_TIPO), ;
                   cMarca, cParecer)
```

**2. Acertar a filial (`cFilAnt`).** A `MTFlgLbDoc` resolve a filial por `xFilial("SCR")`, e
a thread do workflow não garante a filial do documento. O padrão é guardar o valor,
atribuir a filial do pedido, chamar e restaurar num único ponto de saída.

Para que um erro de execução no meio do caminho também passe pelo `Recover`, e a filial seja
restaurada de qualquer jeito, o `Begin Sequence` precisa de um `ErrorBlock` que converta o
erro em `Break`. Sem ele, o erro vai direto para o tratador padrão e o `Recover` nunca roda:

```advpl
Local bErroAnt := ErrorBlock({|e| Break(e)})
Local cFilBkp  := cFilAnt
Local cMsg     := ""
Local oError   := Nil

cFilAnt := cFilialDoc

Begin Sequence
	// localiza a SCR, grava a marca e chama MTFlgLbDoc
	// em validação de negócio: cMsg := "..." e Break (sem argumento)
Recover Using oError
	lRet := .F.
	If oError <> Nil
		cMsg := oError:Description
	EndIf
	FwLogMsg("ERROR",,"APROVDOC","fAprovDoc","","01",cMsg,0,0,{})
End Sequence

ErrorBlock(bErroAnt)
cFilAnt := cFilBkp
```

Repare no `If oError <> Nil`: um `Break` sem argumento, usado para sair por regra de
negócio, chega ao `Recover` com `oError` vazio. Ler `oError:Description` direto ali estoura.

## Vale para o tipo de documento?

Antes de trocar, confirme no fonte padrão que a função trata o seu tipo de documento. No
nosso caso (item de pedido, `IP`), a aprovação cai no ramo que chama `A097ProcLib`, que já
trata o tipo, libera os itens e dispara o e-mail ao comprador. A rejeição segue para
`MaAlcDoc`. Vale confirmar a rejeição em homologação, porque ela usa um caminho sem chave
alternativa.

## O que fica de lição

- **Se o código já está dentro do Protheus, não chame a REST do Protheus.** A REST é a porta
  para quem está fora.
- **Cada camada a mais é tempo e um ponto de falha a mais**: rede, TLS, serviço REST,
  serialização e configuração por ambiente.
- **Ao chamar rotina padrão diretamente, replique o contexto que ela espera**: a filial
  correta e a marca que ela usa para achar o registro.
- **`Recover Using` sem `ErrorBlock` só pega `Break`.** Para capturar erro de execução,
  defina o `ErrorBlock` antes e restaure depois.
- **Não remova o endpoint REST por isso.** Outros consumidores, de fora do Protheus, podem
  continuar precisando dele.
