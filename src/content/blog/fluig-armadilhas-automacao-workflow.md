---
title: 'Cinco armadilhas ao automatizar a movimentação de solicitações no Fluig'
description: 'Formulário apagado pelo saveAndSendTask, dataset que devolve a versão antiga do card, recusa que não lança exceção, atividade atual que o processTask não informa e acentuação dobrada: o que descobrimos ao movimentar solicitações por um dataset agendado.'
pubDate: '2026-10-04'
heroImage: '../../assets/fluig-armadilhas-automacao-workflow.png'
---

Num processo de elaboração de contratos, a minuta sai do Fluig para uma plataforma externa
de assinatura. Quando o contrato volta assinado, o Protheus recebe o retorno e grava os
dados numa tabela de integração. Faltava fechar o ciclo: levar o contrato e os dados de
volta para a solicitação e movimentar o processo.

O primeiro desenho era um laço com temporizador dentro do próprio processo. Ficou difícil
de manter e virou um **dataset agendado** (sincronização a cada 30 minutos) que faz o
papel de orquestrador:

- lista as solicitações paradas na atividade de espera;
- consulta no Protheus se o contrato daquela solicitação já voltou;
- preenche o formulário, anexa o contrato e movimenta com `saveAndSendTask`;
- dá baixa no registro do Protheus.

O código passou na revisão. Mesmo assim, os cinco problemas abaixo só apareceram quando
rodamos no ambiente.

## 1. O `saveAndSendTask` apaga o formulário

A primeira versão enviava no `cardData` só os campos que o retorno preenchia: vigência,
fornecedor, tipo de contrato. A solicitação andou e **todos os outros campos do formulário
voltaram vazios**.

O `saveAndSendTask` não faz merge. O `cardData` substitui o formulário inteiro, e o
campo que não vai nele é gravado em branco.

A correção foi ler o card completo, copiar todos os campos e só depois sobrescrever os que
mudam:

```js
var valores = {};

for (nome in camposAtuais) {
	if (camposAtuais.hasOwnProperty(nome)) {
		valores[nome] = camposAtuais[nome];
	}
}

valores["vigencia"] = dataFim;
valores["dtIniContrato"] = dataInicio;
// ... monta o cardData a partir de "valores"
```

Na leitura, ficam de fora as colunas de controle do Fluig: tudo que tem `#` no nome
(`metadata#id`, `metadata#version`...) e campos como `documentid`, `cardid`, `version` e
`companyid`.

## 2. O dataset do formulário devolve a versão errada do card

Para ler o card, consultamos o dataset do formulário filtrando por `documentid` e lemos
`getValue(0, ...)`. Veio o ID do documento externo **errado**, sem erro nenhum.

O dataset do formulário traz **uma linha por versão do card**. Aquele card tinha duas
versões com valores diferentes, e a linha zero era a mais antiga.

A correção tem duas partes: filtrar pela versão ativa e, por segurança, escolher a linha
certa mesmo que o filtro seja ignorado:

```js
constraints.push(DatasetFactory.createConstraint(
	"metadata#active", "true", "true", ConstraintType.MUST));

function linhaAtiva(ds) {
	var indice = 0;
	var maiorVersao = -1;

	for (var i = 0; i < ds.rowsCount; i++) {
		if (("" + ds.getValue(i, "metadata#active")).toLowerCase() == "true") {
			return i;
		}
		var versao = parseInt("" + ds.getValue(i, "metadata#version"), 10);
		if (!isNaN(versao) && versao > maiorVersao) {
			maiorVersao = versao;
			indice = i;
		}
	}
	return indice;
}
```

Vale revisar outros datasets que leem o formulário com `getValue(0, ...)`. Achamos o mesmo
defeito latente em mais dois.

## 3. A recusa vem no corpo, não como exceção

Em um dos testes, a movimentação foi recusada por inteiro: a solicitação não andou, nada
foi anexado e nada foi gravado. Mesmo assim, o orquestrador registrou sucesso e **deu
baixa no Protheus**. A partir daí, aquele contrato não seria mais processado.

O `saveAndSendTask` não lança exceção quando recusa. A chamada retorna normalmente, e o
motivo vem no texto da resposta. Por isso, a resposta precisa ser conferida antes de
seguir:

```js
var retorno = leResposta(service.saveAndSendTask(/* ... */));

if (respostaComErro(retorno)) {
	throw "Movimentação recusada para a solicitação " + processo + ": " + retorno;
}

function respostaComErro(resposta) {
	var texto = ("" + resposta).toUpperCase();
	return texto.indexOf("ERROR") >= 0 || texto.indexOf("PERMISS") >= 0;
}
```

A baixa no Protheus fica **depois** dessa checagem. Se a movimentação falhar, o registro
continua pendente e a próxima execução tenta de novo.

## 4. O `processTask` não informa onde a solicitação está

Para saber se a solicitação estava na atividade de espera, a primeira versão usava o
`processTask` e pegava o `choosedSequence` da última tarefa.

Funcionou até a solicitação voltar para a espera vinda de outra atividade. Aí deixou de ser
reconhecida.

O motivo é que o `choosedSequence` só é preenchido quando a tarefa é **concluída** e indica
o destino escolhido. A tarefa que está aberta vem com `choosedSequence = 0`. Quando a
solicitação volta para a espera, o último `choosedSequence` preenchido ainda aponta para o
destino antigo.

O dado certo está no `processHistory`: a linha com `active = true` e o maior
`movementSequence` tem, em `stateSequence`, a atividade em que a solicitação está parada.

```js
for (var i = 0; i < ds.rowsCount; i++) {
	if (("" + ds.getValue(i, "active")).toLowerCase() != "true") continue;

	var estado = "" + ds.getValue(i, "stateSequence");
	var movimento = parseInt("" + ds.getValue(i, "processHistoryPK.movementSequence"), 10);

	if (estado != "0" && movimento >= maiorMovimento) {
		maiorMovimento = movimento;
		estadoAtual = estado;
	}
}
```

O `processTask` continua como alternativa quando o histórico não traz uma resposta única.

## 5. Acentuação dobrada entre a API externa e o Protheus

A plataforma de assinatura envia UTF-8, e o Protheus grava em CP1252. Sem conversão, cada
caractere acentuado virava dois na tabela de integração, e o texto chegava corrompido ao
formulário.

A correção ficou no endpoint do Protheus, com `DecodeUTF8` na gravação. Atenção: registros
gravados antes da correção **continuam corrompidos** e precisam ser reenviados.

## Como ficou o orquestrador

```
Dataset agendado (a cada 30 min)
        │
        ▼
Solicitações na atividade de espera  ◄── processHistory (active = true)
        │
        ▼
Contrato já voltou no Protheus? ──não──► tenta na próxima execução
        │ sim
        ▼
Lê o card inteiro (versão ativa) + sobrescreve os campos do retorno
        │
        ▼
saveAndSendTask (formulário completo + contrato anexado)
        │
        ├─ resposta com ERROR ──► não dá baixa, registra no log
        ▼
Baixa do registro no Protheus
```

- O contrato vira **anexo da solicitação**, não documento do GED.
- O orquestrador tem um modo de simulação, que só lê e registra o que faria, sem
  movimentar. Ele foi útil para validar a leitura do card e da atividade antes de ligar a
  movimentação.
- A espera na atividade não tem prazo de expiração, porque quem controla o tempo agora é o
  agendamento.

## O que fica de lição

- **O `cardData` substitui, não complementa.** Leia o card inteiro e reenvie tudo.
- **O dataset do formulário tem uma linha por versão.** Filtre por `metadata#active` e
  nunca confie na linha zero.
- **Confira o texto da resposta do `saveAndSendTask`.** A ausência de exceção não significa
  sucesso. Dê baixa na origem só depois da checagem.
- **A atividade atual está no `processHistory`, não no `processTask`.**
- **Teste no ambiente, com solicitações reais de homologação.** Nenhum desses cinco
  problemas apareceu na revisão de código.
