---
title: 'O cache do dataset Osync que se apaga sozinho'
description: 'Quando a consulta falha ou volta vazia, o onSync reconcilia contra nada e apaga o cache inteiro. Uma checagem de poucas linhas evita isso.'
pubDate: '2026-10-07'
heroImage: '../../assets/fluig-osync-cache-apagado.png'
---

Um dataset sincronizado (o que o Fluig chama de *Osync*: um dataset customizado com a
sincronização ligada) guarda uma cópia local dos dados de um sistema externo.
No nosso caso, os aprovadores por centro de custo, que vêm do Protheus. O formulário
consulta essa cópia, que é rápida, em vez de chamar o Protheus a cada tela.

A cada ciclo, o `onSync` faz uma reconciliação:

1. consulta o sistema de origem;
2. grava (`addOrUpdateRow`) cada linha que veio;
3. percorre o cache e apaga (`deleteRow`) o que **não veio** na consulta.

O passo 3 é o problema. Ele parte do princípio de que a consulta funcionou.

## O que acontece quando ela falha

Se o Protheus está fora do ar, a chamada devolve um dataset de erro ou nenhuma linha. O
`onSync` não sabe a diferença: para ele, "nada voltou" significa que **nada existe mais**.
O passo 3 apaga o cache inteiro, e o formulário passaria a mostrar a lista de aprovadores
vazia até o próximo ciclo bem-sucedido.

Nenhum erro aparece. A sincronização termina normalmente.

## A correção

Antes de reconciliar, valide a consulta. Se ela não parecer válida, **aborte e preserve o
cache**:

```js
function consultaValidaParaSync(query) {

	if (!query || query.getRowsCount() === 0) {
		return false;
	}

	// O dataset de erro tem uma única coluna ('error'); o de sucesso tem as esperadas
	var colunas = query.getColumnsName();
	if (!colunas || colunas.length !== init.columns.length) {
		return false;
	}

	// A chave primária da primeira linha precisa estar preenchida
	for (var i = 0; i < init.primaryKey.length; i++) {
		var valor = query.getValue(0, init.primaryKey[i]);
		if (valor === null || valor === undefined || String(valor) === '') {
			return false;
		}
	}

	return true;
}

function onSync(lastSyncDate) {
	var dataset = createStructure();
	var query = createDataset();

	if (!consultaValidaParaSync(query)) {
		log.error('onSync abortado: consulta inválida, com erro ou vazia. Cache preservado.');
		return dataset; // vazio: não adiciona nem remove linhas
	}

	// ... upsert e depois remoção do que saiu da origem
}
```

Retornar o dataset vazio é o que preserva o cache: sem `addOrUpdateRow` e sem `deleteRow`, o
Fluig não altera nada.

## O que fica de lição

- **Todo `onSync` que apaga precisa desconfiar da própria consulta.** Apagar é a parte
  perigosa da reconciliação.
- **Dataset de erro tem formato diferente do de sucesso.** Conferir o número e o nome das
  colunas pega o caso em que a falha volta como dataset com uma coluna `error`.
- **Registre no log quando abortar.** Sem isso, o cache "parado" parece um defeito.
- **Se vazio for um resultado legítimo** (por exemplo, a origem realmente ficou sem
  registros), a regra precisa de uma exceção explícita, como uma contagem mínima ou uma
  confirmação extra. Aqui, esse caso não existe: sempre há aprovadores.
