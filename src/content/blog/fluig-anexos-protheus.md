---
title: 'Levando anexos do Fluig para a base de conhecimento do Protheus'
description: 'URL de download que dá 404, base64 grande demais para uma chamada e retentativas que duplicam arquivo: como resolvemos o envio de anexos de um processo Fluig para a pré-nota no Protheus.'
pubDate: '2026-10-03'
---

Num processo de notas fiscais, o fornecedor anexa a **Nota Fiscal** e o **Boleto** no Fluig.
A pré-nota é gerada no Protheus, e o time de Compras precisava ver esses arquivos lá, na
**base de conhecimento** da nota, sem baixar do Fluig e subir de novo à mão.

Parece simples: pegar o arquivo, mandar por REST, gravar. Na prática, apareceram quatro
problemas.

## 1. A URL de download responde 404

O caminho mais comum para ler um documento no servidor é `getDownloadURL` e abrir a URL com
`java.net.URL`. Funciona para documentos do GED. Já o **anexo de solicitação** fica em
`/volume/stream/private`, e essa URL aberta sem sessão de usuário responde **404**.

A saída foi ler os bytes direto pelo SDK, sem passar por HTTP:

```js
var bytes = fluigAPI.getDocumentService()
	.getDocumentContentAsBytes(parseInt(idDocumento, 10));
var base64 = String(java.util.Base64.getEncoder().encodeToString(bytes));
```

## 2. Arquivo grande demais para uma chamada só

Um PDF de 10 MB vira uns 13 MB de base64. Um JSON desse tamanho numa única requisição
esbarra no limite de corpo e no timeout e, do lado Protheus, no `MaxStringSize` do
`appserver.ini`, que limita o tamanho de uma string em ADVPL/TLPP. O body inteiro, o
base64 e o arquivo decodificado viram strings na memória do AppServer. Então o script de
serviço envia **em partes**:

- até 4 milhões de caracteres de base64 (cerca de 3 MB de arquivo), vai inteiro;
- acima disso, vai em partes de 4 milhões de caracteres, em ordem.

O tamanho da parte é **múltiplo de 4** de propósito: cada 4 caracteres de base64 viram 3
bytes, então cada parte decodifica sozinha, sem depender da próxima. O Protheus faz
`Decode64` da parte e acrescenta os bytes num arquivo `.PART`.

```js
var totalPartes = Math.ceil(base64.length / TAM_PARTE);
for (var parte = 1; parte <= totalPartes; parte++) {
	params.parte = String(parte);
	params.totalpartes = String(totalPartes);
	params.rawfile = base64.substring((parte - 1) * TAM_PARTE, parte * TAM_PARTE);
	ret = chamaProtheus(params);
	// ...
}
```

Junto vai o `tamanhototal` em bytes. Na última parte, o Protheus confere se o `.PART` tem
exatamente esse tamanho antes de renomear para o nome final. Se não bater, descarta e pede
para recomeçar.

## 3. Retentativa não pode duplicar anexo

A tarefa de serviço do Fluig tenta de novo quando dá erro. Se a primeira tentativa gravou o
arquivo e caiu antes de responder, a segunda gravaria outra cópia.

A solução foi tornar o endpoint **idempotente**. O nome do objeto na base de conhecimento é
montado a partir do número da solicitação e do ID do documento no Fluig:

```
FLG<solicitação>_<id do documento>_<nome do arquivo>.<ext>
```

Antes de gravar, o Protheus procura esse nome na `ACB`. Se já existir, só garante o vínculo
na `AC9` e responde `ja_anexado`. Reenviar a tarefa inteira não cria nada repetido.

O envio em partes segue a mesma ideia:

- a parte que já está gravada (o `.PART` já tem aquele tamanho) é aceita sem escrever de novo;
- a parte que chega fora de ordem recebe **409**, e o Fluig recomeça uma vez a partir da parte 1;
- `.PART` abandonado há mais de um dia é apagado no próximo envio.

## 4. Achar a pré-nota certa

A pré-nota pode nascer de dois jeitos: pela integração automática, que já grava o número da
solicitação num campo customizado da `SF1`, ou incluída manualmente por alguém de Compras,
quando esse campo fica vazio.

O endpoint procura primeiro pelo número da solicitação e, se não achar, pela chave do
documento (número, série, fornecedor e loja). Quando acha pelo documento, grava o número da
solicitação na `SF1`, e a pré-nota manual passa a ficar ligada ao processo também.

## Como ficou o processo

```
Pré-nota (automática ou manual)
        │
        ▼
Integração Protheus – Anexos  ──falha após 3 tentativas──►  Erro na integração – Anexos
        │                                                    ├─ Reenviar anexos
        ▼                                                    └─ Seguir sem anexos
Classificação da nota
```

- A tarefa de serviço envia só os anexos que interessam (descrição "Nota Fiscal" ou "Boleto").
- O resultado de cada arquivo vai para o histórico da solicitação: "anexado" ou "já existia no Protheus".
- Se falhar nas três tentativas, a solicitação vai para uma atividade de tratamento com
  Compras, que escolhe entre reenviar ou seguir sem anexos. O processo não trava.
- O formulário só aceita PDF, XML, JPG, JPEG e PNG até 50 MB, a mesma regra que o Protheus
  valida do lado dele (com as extensões num parâmetro `MV_`).

## O que fica de lição

- **Anexo de solicitação não é documento do GED.** Para ler no servidor, use
  `getDocumentContentAsBytes`, não a URL de download.
- **Divida o base64 em múltiplos de 4.** Cada parte decodifica sozinha e o servidor só
  precisa concatenar bytes.
- **Integração chamada por tarefa de serviço precisa ser idempotente.** Use uma chave que
  não muda entre tentativas (solicitação + ID do documento) e deixe o servidor responder
  "já tenho".
- **Planeje a falha no diagrama.** Um evento de erro levando a uma atividade humana evita
  solicitação parada com erro no meio do fluxo.

Um detalhe do lado Protheus: o endpoint grava o arquivo no diretório da base de
conhecimento (`MsDocPath()`), então exige `MV_MODEDOC = 1`, ou seja, base de conhecimento em
file system. Com o conteúdo guardado no banco de dados, a gravação seria outra.
