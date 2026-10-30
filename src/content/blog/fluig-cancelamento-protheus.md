---
title: 'Cancelar no Fluig sem deixar sujeira no Protheus'
description: 'Cancelar a solicitação no Fluig deixava pedido de compra e pré-nota vivos no Protheus, com o saldo do pedido já consumido. A correção foi uma matriz por etapa: quem cancela, o que excluir antes e quando o cancelamento deixa de ser permitido. Se a exclusão falha, o cancelamento é recusado.'
pubDate: '2026-10-30'
heroImage: '../../assets/fluig-cancelamento-protheus.png'
---

**Resumo:** num processo Fluig que cria documentos no Protheus, o botão **Cancelar** só
encerra a solicitação. O que já foi para o ERP fica lá. A correção tem três partes: uma
**matriz por etapa** (quem pode cancelar e o que precisa ser desfeito), a **exclusão no
Protheus antes** de aceitar o cancelamento, na ordem certa, e a **recusa** do cancelamento
sempre que a exclusão não acontece. No caminho apareceram sete armadilhas, e a primeira é
do próprio Fluig.

## O problema

O processo é o de pagamento de nota fiscal. Quando a nota não tem pedido de compra, o
Fluig **cria o pedido** no Protheus, espera a aprovação pela alçada do centro de custo e
depois **cria a pré-nota** de entrada. No fim, a área tributária classifica a nota.

Até então, cancelar a solicitação no meio desse caminho fazia exatamente o que o nome diz:
cancelava a solicitação. No Protheus ficavam:

- **o pedido de compra**, ocupando orçamento e aparecendo como pendente para o comprador;
- **a pré-nota**, que consome o saldo do pedido. O pedido aparecia como atendido por uma
  nota que ninguém ia pagar.

Ninguém dava baixa nesses documentos, porque ninguém sabia que eles existiam. O processo
que os criou já estava encerrado.

## Dois caminhos, e só um deles é nosso

O processo pergunta logo no início se a nota tem pedido de compra:

- **Caminho A, com pedido:** o solicitante escolhe um pedido que **já existe** no
  Protheus. O Fluig cria só a pré-nota.
- **Caminho B, sem pedido:** o Fluig **cria** o pedido, espera a alçada e depois cria a
  pré-nota.

Essa diferença virou a regra que vale acima de todas as outras: **pedido escolhido pelo
usuário nunca é excluído.** Ele é do Protheus, tem vida própria e pode estar atendendo
outras notas. O cancelamento só apaga o que o próprio processo criou.

## A matriz de cancelamento

Antes de escrever código, a regra foi para um documento aprovado pelas áreas envolvidas:
para cada etapa do fluxo, o que já existe no Protheus, quem pode cancelar e o que o
sistema faz.

| Etapa | O que já existe no Protheus | Quem cancela | O que o sistema faz |
|---|---|---|---|
| Início, análise de Compras, ajuste pelo solicitante | Nada | Qualquer participante | Só cancela |
| Integração do pedido, fila da alçada (alçada **pendente**) | Pedido | Só o gestor | Exclui o pedido (caminho B) |
| Integração do pedido, fila da alçada (alçada **aprovada**) | Pedido aprovado | Ninguém | Recusa |
| Integração da pré-nota, inclusão manual da pré-nota | Pedido e pré-nota | Só o gestor | Exclui a pré-nota e depois o pedido |
| Classificação, nota **ainda não classificada** | Pedido e pré-nota | Só o gestor | Exclui a pré-nota e depois o pedido |
| Classificação, nota **já classificada** | Nota fiscal e títulos a pagar | Ninguém | Recusa |
| Processo encerrado (fim aprovado ou reprovado) | — | Ninguém | Recusa |
| **Qualquer outra etapa** | — | Ninguém | Recusa |

A última linha é a mais importante: **o que não está na matriz é recusado**. Um gateway
novo no diagrama, uma etapa renomeada ou um estado que alguém esqueceu não cancelam em
silêncio.

Nota classificada fica bloqueada de propósito. A partir dali existe nota fiscal de verdade
e título a pagar, e desfazer isso é estorno fiscal, trabalho da área tributária dentro do
Protheus. O Fluig não deve fazer isso por conta própria.

### A ordem importa: pré-nota antes do pedido

A pré-nota reserva o saldo do pedido, e enquanto ela existir o Protheus **não deixa
excluir o pedido**. Por isso a sequência é sempre a mesma:

1. Excluir a pré-nota.
2. Excluir o pedido (só no caminho B).
3. Deixar o Fluig concluir o cancelamento.

Se o passo 1 ou o 2 falhar, o 3 não acontece. O processo continua aberto e o usuário vê a
mensagem que o Protheus devolveu.

## Onde a regra mora: `beforeCancelProcess`

O evento `beforeCancelProcess` do processo roda antes do cancelamento. Se ele lança erro,
o Fluig recusa o cancelamento e mostra o erro na tela. É o lugar natural para a matriz:

```js
function beforeCancelProcess(colleagueId, processId) {
	var state = parseInt(getValue("WKNumState"), 10);
	var numfluig = getValue("WKNumProces");

	// Processo encerrado
	if (state == ESTADO_FIM_APROVADO || state == ESTADO_FIM_REPROVADO) {
		throw "nogestorcancel";
	}

	// Nada foi para o Protheus: qualquer participante cancela
	if (ESTADOS_LIVRES.indexOf(state) >= 0) {
		return;
	}

	if (!ehGestorProcesso(getValue("WKUser"))) {
		throw "nogestorcancel";
	}

	var campos = hAPI.getCardData(numfluig);
	var caminhoB = valor(campos, "pedido") == "nao";

	// Pedido criado, ainda sem pré-nota: janela da alçada
	if (ESTADOS_ALCADA.indexOf(state) >= 0) {
		if (valor(campos, "aprovado") == "sim") {
			throw "pedidoaprovado";
		}
		if (caminhoB) {
			excluiPedido(campos, numfluig);
		}
		return;
	}

	// Pré-nota existe (ou está sendo criada)
	if (ESTADOS_PRENOTA.indexOf(state) >= 0) {
		excluiPreNota(campos, numfluig); // recusa se a NF já foi classificada
		if (caminhoB) {
			excluiPedido(campos, numfluig);
		}
		return;
	}

	// Fora da matriz: recusa
	throw "nogestorcancel";
}
```

É ECMA 5, porque o script roda no Rhino (`indexOf` em array funciona). Os campos do
formulário vêm de `hAPI.getCardData`, o mesmo padrão das tarefas de serviço. Neste evento
não há formulário carregado, e `hAPI.getCardValue` não é confiável.

## As armadilhas

### 1. `WKManagerMode` vem sempre falso no `beforeCancelProcess`

A primeira ideia foi usar a variável que o Fluig oferece para dizer se o usuário é gestor
do processo. No teste, ela voltou **`false` em todas as chamadas**, inclusive para quem
abria a tela de cancelamento justamente como gestor. Nesse evento a variável não é
preenchida.

A decisão passou a vir de um **papel** do Fluig, consultado pelo dataset interno
`workflowColleagueRole`. A variável ficou só como atalho, caso algum dia venha verdadeira:

```js
function ehGestorProcesso(usuario) {
	var gm = String(getValue("WKManagerMode")).toLowerCase();
	if (gm == "true") {
		return true;
	}

	var c1 = DatasetFactory.createConstraint("workflowColleagueRolePK.roleId",
		PAPEL_GESTOR, PAPEL_GESTOR, ConstraintType.MUST);
	var c2 = DatasetFactory.createConstraint("workflowColleagueRolePK.colleagueId",
		usuario, usuario, ConstraintType.MUST);
	var ds = DatasetFactory.getDataset("workflowColleagueRole", null, [c1, c2], null);

	return ds != null && ds.rowsCount > 0;
}
```

Se a consulta do papel falhar, a função devolve falso. Na dúvida, o cancelamento é
recusado, não liberado.

### 2. "Pedido aprovado" não pode valer em todas as etapas

A decisão de negócio foi bloquear o cancelamento de pedido **já aprovado na alçada**. A
primeira versão da regra testava `aprovado == "sim"` logo depois de conferir o gestor.

Só que, pelo desenho do fluxo, **toda** solicitação que chega à pré-nota já passou pela
alçada. O teste solto bloqueava também as etapas de pré-nota e classificação, e metade da
matriz ficava inalcançável.

O bloqueio vale só na **janela entre a aprovação e a pré-nota**. Daí para frente, o
critério é outro: a nota já foi classificada ou não. Isso só apareceu porque a matriz foi
escrita antes do código. Em tabela, a contradição fica à vista. Num `if` ela passa.

### 3. O Fluig não lê o banco do Protheus, então a exclusão é a consulta

O plano original tinha um script que consultava a tabela de notas para saber se a nota já
estava classificada. Só que o Fluig desse ambiente **não tem acesso ao banco do
Protheus**: toda leitura passa por REST.

Em vez de criar um endpoint de consulta e outro de exclusão, o próprio endpoint de
exclusão **é a consulta**. Ele localiza a pré-nota e, se ela já estiver classificada,
recusa sem excluir nada:

```advpl
// NF classificada: recusa e devolve um marcador próprio
If !Empty((cAlias)->F1_STATUS)
	oRet["Code"]         := 400
	oRet["Classificada"] := .T.
	oRet["Message"]      := "Nota fiscal ja classificada. Exclusao nao permitida."
	Break
EndIf
```

Assim, o cancelamento faz uma chamada REST a menos. E a leitura e a ação acontecem na
mesma chamada, sem chance de a nota ser classificada entre uma e outra.

O Fluig testa o campo **`Classificada`**, nunca o texto da mensagem. Mensagem é para gente
ler, e alguém vai mudar o texto um dia. Um marcador dedicado no JSON é contrato.

### 4. Processos antigos não têm o vínculo

Para achar a pré-nota de uma solicitação, a inclusão passou a gravar o **número da
solicitação Fluig** num campo customizado da SF1. O valor já chegava no endpoint de
inclusão, mas era descartado.

O problema são as solicitações que **já estavam abertas** antes da mudança. As
pré-notas delas não têm o vínculo, e isso causaria dois defeitos:

1. a exclusão não acharia a pré-nota e responderia "já não existe". O passo seguinte
   tentaria excluir um pedido com saldo consumido, e o Protheus recusaria;
2. pior: a verificação de classificação daria **falso negativo**. Nota classificada
   passaria por não classificada, e a única regra que não pode falhar falharia.

A saída foi um **fallback pela chave do documento**: se não achar pelo número Fluig,
procura por filial, número, série, fornecedor e loja, que o formulário já tem desde a
inclusão.

```advpl
// 1) Pelo vínculo com o Fluig (solicitações novas)
BeginSQL Alias cAlias
	SELECT TOP 1 F1_DOC, F1_SERIE, F1_FORNECE, F1_LOJA, F1_STATUS
	FROM %table:SF1% SF1
	WHERE SF1.F1_FILIAL = %xFilial:SF1%
	  AND SF1.F1_XFLUIG = %exp:cIdFluig%
	  AND SF1.%notDel%
EndSQL

// 2) Se não achou, pela chave do documento (solicitações anteriores à mudança)
If (cAlias)->(Eof())
	(cAlias)->(DbCloseArea())
	cAlias := GetNextAlias()
	BeginSQL Alias cAlias
		SELECT TOP 1 F1_DOC, F1_SERIE, F1_FORNECE, F1_LOJA, F1_STATUS
		FROM %table:SF1% SF1
		WHERE SF1.F1_FILIAL  = %xFilial:SF1%
		  AND SF1.F1_DOC     = %exp:cNumNf%
		  AND SF1.F1_SERIE   = %exp:cSerie%
		  AND SF1.F1_FORNECE = %exp:cFornece%
		  AND SF1.F1_LOJA    = %exp:cLoja%
		  AND SF1.%notDel%
	EndSQL
EndIf
```

Funciona para trás sem script de correção no banco do ERP. Um backfill do vínculo nas
solicitações abertas é bem-vindo, mas não é pré-requisito.

### 5. Mesmo com o número do pedido, confira de quem ele é

No caminho B o Fluig sabe o número do pedido que criou e manda no corpo da exclusão. Seria
fácil excluir direto por esse número.

O endpoint não confia. Se o número vier, ele posiciona o pedido e confere se o **número
Fluig gravado no pedido** é o da solicitação que está pedindo a exclusão. Se não for,
recusa:

```advpl
If SC7->(dbSeek(xFilial("SC7") + PadR(cNumPed, TamSX3("C7_NUM")[1])))
	If AllTrim(SC7->C7_XFLUIG) <> cNumFluig
		oRet["Code"]    := 400
		oRet["Message"] := "Pedido informado nao pertence ao processo Fluig " + cNumFluig
		Break
	EndIf
EndIf
```

Um campo de formulário editado por engano, ou um card copiado de outra solicitação, não
pode apagar o pedido de outra pessoa. A regra "pedido do caminho A nunca é excluído" fica
garantida em dois lugares: no Fluig, pelo caminho escolhido, e no Protheus, pelo vínculo.

### 6. "Já não existe" é sucesso

O cancelamento chama o Protheus, e o Protheus pode excluir o documento e a resposta se
perder no caminho (timeout, queda da thread do REST). O Fluig recebe erro, o cancelamento
é recusado e o usuário tenta de novo.

Na segunda tentativa, a pré-nota já não existe. Se o endpoint respondesse "não
encontrado" como erro, a solicitação nunca mais seria cancelada. Por isso, documento
inexistente devolve **200**: o objetivo da chamada era que ele não existisse, e ele não
existe.

É a mesma ideia do [post sobre reenvio sem duplicar](/blog/fluig-reenvio-sem-duplicar/):
toda chamada de integração vai rodar mais de uma vez, então ela precisa dar o mesmo
resultado na segunda.

### 7. `MSExecAuto` de exclusão precisa do registro posicionado

Para excluir pela rotina padrão (`MATA120` para o pedido, `MATA140` para a pré-nota, com
opção 5), informar o número no cabeçalho não basta. O ExecAuto de exclusão trabalha sobre
o **registro posicionado**, então o endpoint faz o `DbSeek` antes:

```advpl
SC7->(dbSetOrder(1)) // C7_FILIAL + C7_NUM + C7_ITEM + C7_SEQUEN
If !SC7->(dbSeek(xFilial("SC7") + PadR(cNumPed, TamSX3("C7_NUM")[1])))
	oRet["Code"]    := 200
	oRet["Message"] := "Pedido ja inexistente"
	Break
EndIf

aAdd(aCabec, {"C7_NUM", SC7->C7_NUM, Nil})

lMsErroAuto := .F.
MSExecAuto({|x, y, z| Mata120(1, x, y, z)}, aCabec, aItens, 5)

If lMsErroAuto
	cMsg := MostraErro("/temp", "pedcompras_exclusao.log")
	oRet["Code"]    := 404
	oRet["Message"] := cMsg
	Break
EndIf
```

Usar a rotina padrão em vez de um `RecLock` com `DbDelete` é o que garante que o Protheus
desfaça o que ele mesmo amarrou ao documento: saldo, alçada, itens. Mesmo assim, vale
conferir em homologação se a exclusão do pedido limpa os registros de aprovação pendentes
(SCR) do seu ambiente.

## A mensagem que o usuário vê

O `throw` do `beforeCancelProcess` aparece num modal do Fluig, com o texto cru do erro.
Para não mostrar `nogestorcancel` para o usuário, o script lança **tokens** curtos, e o
formulário troca o texto do modal por uma mensagem legível:

| Token | Mensagem na tela |
|---|---|
| `nogestorcancel` | Nesta etapa o cancelamento só pode ser feito pelo gestor do processo. |
| `pedidoaprovado` | Não é permitido cancelar: o pedido já foi aprovado na alçada do centro de custo. |
| `nfclassificada` | Não é permitido cancelar: a nota fiscal já foi classificada no Protheus. Solicite o estorno à área tributária. |
| `proterr:<texto>` | Não foi possível cancelar. O Protheus retornou: \<texto\> |

O modal é criado na página pai depois que o erro volta, então o formulário observa o DOM
com um `MutationObserver`:

```js
var MSG_CANCEL = {
	nogestorcancel: "Nesta etapa o cancelamento só pode ser feito pelo gestor do processo.",
	pedidoaprovado: "Não é permitido cancelar: o pedido já foi aprovado na alçada do centro de custo.",
	nfclassificada: "Não é permitido cancelar: a nota fiscal já foi classificada no Protheus. Solicite o estorno à área tributária."
};

var observer = new MutationObserver(function () {
	var $msg = parent.$(".modal-content > .modal-body > .message");
	var html = $msg.html();

	// O modal pode existir sem .message: html() devolve null
	if (html == null || typeof html !== "string") {
		return;
	}

	var texto = null;
	var idx = html.indexOf("proterr:");

	if (idx >= 0) {
		var detalhe = html.substring(idx + 8).replace(/<[^>]*>/g, "").trim();
		texto = "Não foi possível cancelar. O Protheus retornou: " + detalhe;
	} else {
		for (var token in MSG_CANCEL) {
			if (html.indexOf(token) >= 0) {
				texto = MSG_CANCEL[token];
				break;
			}
		}
	}

	if (texto != null) {
		$msg.html("<strong>" + texto + "</strong>");
	}
});

observer.observe(parent.document.body, { childList: true, subtree: true });
```

A versão anterior do observer tratava um token só e quebrava quando `html()` voltava
`null`. Um erro no observer não aparece para ninguém: o usuário só vê o token cru e acha
que o sistema está com defeito.

## O que ainda fica em aberto

- **Cancelar durante uma integração automática.** As tarefas que criam o pedido e a
  pré-nota têm tentativas automáticas. Cancelar exatamente durante uma delas é raro, mas
  possível. Como a exclusão é idempotente e o vínculo é conferido, o pior caso é um
  documento criado depois da exclusão. Vale um relatório que cruze documentos com vínculo
  Fluig e solicitações canceladas.
- **Alçada.** A exclusão pela rotina padrão deveria levar junto os registros de aprovação.
  É a primeira coisa a conferir em homologação.

## O que fica de lição

- **Cancelar é uma integração, não um botão.** Se o processo criou algo fora do Fluig, o
  cancelamento precisa desfazer, ou recusar.
- **Escreva a matriz antes do código.** Etapa, o que existe no ERP, quem cancela, o que
  acontece. Contradições de regra aparecem na tabela e somem num `if`.
- **O que não está na matriz é recusado.** Etapa nova não pode ganhar cancelamento livre
  por esquecimento.
- **Desfaça na ordem inversa da criação.** Pré-nota antes do pedido. E se um passo falhar,
  não conclua o cancelamento.
- **Só apague o que é seu, e confira no destino.** O vínculo com o número Fluig no
  registro do Protheus é o que separa "pedido criado por este processo" de "pedido de
  outra pessoa".
- **"Já não existe" é sucesso.** Sem isso, uma resposta perdida trava o cancelamento para
  sempre.
- **Não confie em variável de contexto sem testar no evento.** `WKManagerMode` existe,
  mas no `beforeCancelProcess` ela não diz nada.
