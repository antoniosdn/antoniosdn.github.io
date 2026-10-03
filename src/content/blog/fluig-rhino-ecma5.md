---
title: 'Por que seu dataset Fluig quebra com let e arrow function'
description: 'Datasets e scripts de workflow do Fluig rodam no Rhino, limitado ao ECMA 5. Veja o que evitar e como substituir.'
pubDate: '2026-10-03'
---

Quem chega ao Fluig vindo do JavaScript moderno costuma tropeçar no mesmo ponto: o código
funciona no navegador e falha no servidor. O motivo é que **datasets** e **scripts de workflow**
rodam no **Rhino**, um motor JavaScript em Java que só aceita a sintaxe do **ECMA 5**.

## O que não funciona

| Evite (ES6+)                    | Use (ECMA 5)                              |
| ------------------------------- | ----------------------------------------- |
| `let` / `const`                 | `var`                                     |
| `() => {}`                      | `function () {}`                          |
| `` `Olá ${nome}` ``             | `"Olá " + nome`                           |
| `for (var x of lista)`          | `for (var i = 0; i < lista.length; i++)`  |
| `var { a } = obj`               | `var a = obj.a`                           |
| `class Foo {}`                  | funções construtoras                      |

## Exemplo

```js
function createDataset(fields, constraints, sortFields) {
	var dataset = DatasetBuilder.newDataset();
	dataset.addColumn("CODIGO");
	dataset.addColumn("DESCRICAO");

	var itens = [{ codigo: "01", descricao: "Matriz" }];
	for (var i = 0; i < itens.length; i++) {
		dataset.addRow([itens[i].codigo, itens[i].descricao]);
	}
	return dataset;
}
```

Os formulários (`forms/`) rodam no navegador e aceitam ES6+, exceto os eventos de servidor,
como `displayFields.js`. Na dúvida, escreva em ECMA 5 o que roda no servidor.
