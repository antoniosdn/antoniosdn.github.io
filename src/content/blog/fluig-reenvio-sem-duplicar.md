---
title: 'Reenvio sem duplicar: idempotência nas tarefas de serviço do Fluig com o Protheus'
description: 'Um HTTP 500 no meio da integração mandava a solicitação para o tratamento de erro, e o reenvio incluía de novo o que já tinha entrado no Protheus. A correção é conferir antes de incluir e nunca incluir sem ter conferido.'
pubDate: '2026-10-14'
heroImage: '../../assets/fluig-reenvio-sem-duplicar.png'
---

Dois processos Fluig integram com o Protheus por uma **tarefa de serviço**. Um inclui uma
solicitação de compra. O outro inclui um título a pagar parcelado, com uma chamada REST
por parcela.

Se o script da tarefa lança erro, a solicitação vai para uma atividade **Tratar erro**.
Quem atende confere o problema e reenvia, e o script roda de novo do começo.

Com falhas intermitentes do Protheus (erro de licença no AppServer, HTTP 500), esse
reenvio **incluía de novo o que já tinha entrado**.

## Como a duplicidade acontecia

**1. O Protheus grava, e a resposta se perde.** Quando a thread do REST cai, o Fluig
recebe uma resposta vazia e trata como erro. Mas a inclusão pode ter sido gravada antes da
queda. No reenvio, entra uma segunda solicitação de compra para o mesmo pedido.

**2. O laço para no meio.** No título parcelado, a parcela 01 entrou e a 02 voltou com
HTTP 500. No reenvio, o laço recomeçou da parcela 01, que já existia: título duplicado ou
erro de chave, com a solicitação presa no tratamento de erro.

Nos dois casos, o script trata cada execução como se fosse a primeira.

## A solução: conferir antes de incluir

Antes de incluir, a tarefa pergunta ao Protheus se aquilo já foi feito. A chave é o
**número da solicitação Fluig**, que a integração já grava no registro do Protheus.

```js
function servicetask31(attempt, message) {
	var numproc = getValue("WKNumProces");

	// Se a SC já foi incluída, reaproveita o número em vez de incluir de novo
	var numSolExistente = buscaSolicitacaoExistente(numproc);

	if (numSolExistente != "") {
		hAPI.setCardValue("numSolCompra", numSolExistente);
		return;
	}

	// ... inclui normalmente
}
```

Se a SC já existe, a tarefa termina com sucesso e grava no formulário o número que já
estava no Protheus.

A consulta só considera registros emitidos a partir da abertura da solicitação. Sem isso,
um registro antigo com o mesmo número (comum numa base de teste copiada da produção) faria
a tarefa pular uma inclusão que precisava acontecer.

## Se a consulta falhar, não inclua

A consulta também vai ao Protheus, que é justamente o que está instável. Se ela falhar e o
código entender "não veio nada" como "não existe", a proteção some quando mais precisa
dela. Por isso, a busca **lança erro** sempre que não consegue conferir:

```js
function buscaSolicitacaoExistente(numproc) {
	var emissaoDe = (hAPI.getCardValue("dataSolicitacao") + "").replace(/-/g, "");

	var constraints = [
		DatasetFactory.createConstraint("numfluig", numproc + "", numproc + "", ConstraintType.MUST),
		DatasetFactory.createConstraint("emissaode", emissaoDe, emissaoDe, ConstraintType.MUST)
	];

	var ds = DatasetFactory.getDataset("ds_sc_por_numfluig", null, constraints, null);

	// Sem retorno: não dá para saber, então não inclui
	if (ds == null) {
		throw "Falha ao consultar SC existente";
	}

	// Consulta feita, nada encontrado: pode incluir
	if (ds.rowsCount == 0) {
		return "";
	}

	// Veio linha sem o número: é o dataset de erro
	var numSol = ds.getValue(0, "NUMSOL");
	if (numSol == null || (numSol + "") == "") {
		throw "Falha ao consultar SC existente: " + ds.getValue(0, "error");
	}

	return numSol + "";
}
```

Na pior hipótese, a solicitação vai para o tratamento de erro, onde ela já estaria sem a
correção, mas sem duplicar nada.

## No laço: pular as parcelas que já entraram

No título parcelado, a consulta roda uma vez antes do laço e devolve as parcelas que já
existem. Dentro do laço, a parcela incluída é pulada:

```js
var parcelasIncluidas = buscaParcelasIncluidas(numSolicitacao); // { "01": true }

while (p <= totalParcelas) {
	var parcela = padLeft(p, 2);

	if (parcelasIncluidas[parcela] === true) {
		p++;
		continue;
	}

	// ... inclui só a parcela que falta
	p++;
}
```

No caso da parcela 02, o reenvio encontrou a 01, pulou e incluiu só a 02.

Com o script seguro para rodar de novo, deu para ligar as tentativas automáticas da tarefa
de serviço (três, com alguns minutos de intervalo). Na ordem inversa, cada tentativa seria
mais uma duplicação.

## O que fica de lição

- **Todo script de integração vai rodar mais de uma vez.** Escreva como se a execução
  anterior pudesse ter feito parte do trabalho.
- **Grave a chave do Fluig no registro do Protheus.** Sem ela, não há como conferir.
- **Se a conferência falhar, não inclua.** Erro na consulta não é o mesmo que "não existe".
