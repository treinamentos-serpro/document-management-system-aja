# Especificação - Document Management System

> Especificação funcional e técnica para orientar a implementação incremental do
> Document Management System (DMS). Este documento não implementa nem altera os
> arquivos de backend ou frontend.

## 1. Objetivo

Entregar uma aplicação web que permita a um usuário enviar documentos, consultar os documentos sob sua responsabilidade e baixar cada arquivo, mantendo os arquivos no filesystem local e os metadados em memória.

## 2. Escopo

### Dentro do escopo

- Upload de um documento por requisição.
- Listagem dos documentos associados ao usuário da requisição.
- Download de um documento pelo identificador, restrito ao seu dono.
- Metadados mantidos em memória durante a execução do processo.
- Arquivos persistidos localmente em `backend/storage`, usando `multer` com `diskStorage`.
- Interface React para enviar, listar e baixar documentos, consumindo a API pelo prefixo `/api` configurado no proxy do Vite.
- Configuração operacional por variáveis de ambiente.

### Fora do escopo

- Armazenamento externo, em nuvem ou em banco de dados.
- Persistência dos metadados após reinício do processo.
- Autenticação, cadastro de usuários e gestão de credenciais.
- Versionamento, edição, exclusão, compartilhamento ou busca avançada de documentos.
- Upload múltiplo, pastas, permissões configuráveis ou auditoria.
- Restrições por tipo de arquivo nesta fase, além da validação de que o upload contém um arquivo válido.

## 3. Premissas e decisões

- A identidade do usuário deve estar disponível para controllers por meio do contexto da requisição. Para o contrato inicial, esse contexto é representado pelo cabeçalho `X-User-Id`; sua validação como identidade confiável depende de uma camada de autenticação futura. O cabeçalho, isoladamente, não é mecanismo de autenticação e o serviço não deve ser exposto como seguro em ambiente público sem essa camada.
- O valor de `X-User-Id` é obrigatório, não vazio e tratado como identificador opaco. A API não cria nem administra usuários.
- O nome original é metadado de apresentação, nunca caminho no filesystem. O nome físico do arquivo é gerado pelo servidor para evitar colisões, traversal e dependência de nomes fornecidos pelo cliente.
- Os documentos são ordenados por `uploadedAt` decrescente na listagem. Nesta fase, a lista é retornada completa, sem paginação.
- Uma reinicialização limpa os metadados em memória. Arquivos locais podem permanecer sem metadados; não há promessa de recuperação ou limpeza automática desses arquivos nesta fase.
- A interface permite operar sobre o usuário identificado pelo contexto configurado para a requisição; a tela não substitui autenticação.

## 4. Requisitos funcionais

| ID | Requisito | Critério de aceite |
| --- | --- | --- |
| RF-01 | O usuário pode enviar um documento. | Um arquivo válido enviado como multipart/form-data é gravado localmente e retorna metadados com identificador único. |
| RF-02 | O sistema exige a identidade do usuário nas operações de documentos. | Requisições sem `X-User-Id` válido são rejeitadas e não leem nem alteram documentos. |
| RF-03 | O usuário pode listar seus documentos. | A resposta contém somente documentos cujo `owner` corresponde ao usuário da requisição, ordenados do mais recente ao mais antigo. |
| RF-04 | O usuário pode baixar um documento pelo identificador. | O conteúdo binário do documento é devolvido somente ao dono; documento inexistente ou pertencente a outro usuário não é revelado. |
| RF-05 | O usuário recebe feedback de operações e erros. | A interface apresenta estado de carregamento, sucesso e falha para upload, listagem e download, sem perder a lista em caso de erro de upload. |
| RF-06 | O usuário pode selecionar um arquivo e enviar pela interface. | A interface envia um arquivo por vez, mostra seu nome e atualiza a listagem após upload concluído. |
| RF-07 | O usuário pode iniciar o download pela interface. | Cada item listado oferece uma ação de download que salva ou abre o arquivo recebido sem expor o caminho local do servidor. |
| RF-08 | A listagem vazia é tratada como estado normal. | A interface apresenta estado vazio quando a API retorna uma lista sem documentos. |

## 5. Requisitos não funcionais

| ID | Requisito |
| --- | --- |
| RNF-01 | Os arquivos enviados devem ser gravados no filesystem local da aplicação, em `backend/storage` por padrão, usando `multer` com `diskStorage`. Não utilizar serviços de armazenamento externos. |
| RNF-02 | Os metadados devem residir em memória e ser perdidos quando o processo backend reiniciar. |
| RNF-03 | A configuração deve seguir 12-Factor: porta, diretório de armazenamento e limite de tamanho devem ser configuráveis por variáveis de ambiente, com defaults documentados. |
| RNF-04 | O backend deve usar CommonJS e Express; o frontend deve usar React + Vite e JavaScript, respeitando as dependências já existentes. |
| RNF-05 | O backend deve separar rotas, controllers, services e repositories, seguindo o fluxo `routes -> controllers -> services -> repositories`. Camadas internas não devem depender de Express ou conhecer a camada HTTP. |
| RNF-06 | IDs e nomes físicos dos arquivos devem ser gerados no servidor. Dados fornecidos pelo cliente não podem determinar caminhos graváveis ou permitir acesso a arquivos fora do diretório de armazenamento. |
| RNF-07 | Erros devem ser convertidos em respostas HTTP previsíveis, sem expor stack traces, caminhos absolutos ou detalhes internos. |
| RNF-08 | A interface deve permanecer utilizável em telas estreitas e comunicar claramente estados de carregamento, vazio, sucesso e erro. |
| RNF-09 | Operações de arquivo devem tratar falhas de leitura e escrita. Se a persistência dos metadados falhar após o arquivo ser gravado, o arquivo recém-criado deve ser removido quando possível. |

### Configuração prevista

| Variável | Obrigatória | Default | Uso |
| --- | --- | --- | --- |
| `PORT` | Não | `3000` | Porta HTTP do backend. |
| `STORAGE_DIR` | Não | `backend/storage` | Diretório local para os arquivos enviados. O diretório deve ser criado se não existir. |
| `MAX_FILE_SIZE_BYTES` | Não | `10485760` (10 MiB) | Limite máximo do tamanho de um arquivo; valores inválidos devem impedir a inicialização ou usar validação documentada, sem ignorar o limite silenciosamente. |

## 6. Modelo de dados

### Metadados do documento

Os metadados são mantidos por um repository em memória. O caminho físico não é retornado na API.

| Campo | Tipo | Obrigatório | Descrição |
| --- | --- | --- | --- |
| `id` | string | Sim | Identificador único gerado pelo servidor. |
| `originalName` | string | Sim | Nome original informado pelo cliente, sanitizado para apresentação e para `Content-Disposition`. |
| `size` | number | Sim | Tamanho do arquivo em bytes. Deve ser maior que zero. |
| `uploadedAt` | string | Sim | Data e hora do upload em formato ISO 8601 UTC. |
| `owner` | string | Sim | Identificador do usuário obtido do contexto da requisição. |
| `mimeType` | string | Sim | Tipo MIME informado/detectado no upload; fallback `application/octet-stream` quando desconhecido. |
| `storageName` | string | Sim | Nome físico aleatório gerado pelo servidor e usado pelo repository para localizar o arquivo. Campo interno, não exposto pela API. |

Não há relacionamento persistente com uma entidade de usuário: `owner` é apenas um identificador associado ao documento. A estrutura exata de armazenamento em memória fica a cargo do repository, que deve oferecer operações para registrar, listar por dono e localizar por ID e dono.

## 7. Contratos de API

### Convenções

- As rotas de aplicação são expostas sob `/api` para o frontend; abaixo, os caminhos incluem esse prefixo.
- Todas as operações de documento exigem `X-User-Id` não vazio.
- Respostas JSON usam UTF-8. Erros seguem o formato `{ "error": { "code": "...", "message": "..." } }`.
- Campos internos (`storageName` e caminhos locais) nunca são retornados.

### `POST /api/upload`

Envia um documento do usuário.

- Cabeçalho: `X-User-Id: <identificador>`.
- Corpo: `multipart/form-data`, campo de arquivo `file`, exatamente um arquivo.
- Limite: `MAX_FILE_SIZE_BYTES`.
- Sucesso: `201 Created`.

```json
{
  "document": {
    "id": "<id>",
    "originalName": "relatorio.pdf",
    "size": 24576,
    "uploadedAt": "2026-09-29T12:00:00.000Z",
    "owner": "<identificador>",
    "mimeType": "application/pdf"
  }
}
```

Erros: `400` para identidade ausente ou arquivo ausente/inválido; `413` quando o limite de tamanho é excedido; `500` para falha inesperada de gravação ou registro. Em caso de falha depois da gravação, remover o arquivo parcial ou recém-criado quando possível.

### `GET /api/documents`

Lista os documentos do usuário, mais recentes primeiro.

- Cabeçalho: `X-User-Id: <identificador>`.
- Sucesso: `200 OK`; lista vazia é válida.

```json
{
  "documents": [
    {
      "id": "<id>",
      "originalName": "relatorio.pdf",
      "size": 24576,
      "uploadedAt": "2026-09-29T12:00:00.000Z",
      "owner": "<identificador>",
      "mimeType": "application/pdf"
    }
  ]
}
```

Erros: `400` para identidade ausente; `500` para falha inesperada ao consultar o repository.

### `GET /api/documents/:id/download`

Transfere o conteúdo binário de um documento pertencente ao usuário.

- Cabeçalho: `X-User-Id: <identificador>`.
- Sucesso: `200 OK`, corpo binário, `Content-Type` conforme o metadado e `Content-Disposition: attachment` com o nome original sanitizado.
- Se o ID não existir, pertencer a outro usuário ou não houver metadado correspondente, responder `404 Not Found` com mensagem genérica.
- Erros: `400` para identidade ausente ou ID inválido; `404` para documento inexistente/não autorizado; `500` para erro inesperado ao ler o arquivo.

O backend não deve aceitar caminho de arquivo fornecido pelo cliente. O repository resolve o arquivo usando somente metadados internos e o diretório de armazenamento configurado.

### Erros comuns

```json
{
  "error": {
    "code": "FILE_TOO_LARGE",
    "message": "O arquivo excede o tamanho máximo permitido."
  }
}
```

Códigos previstos: `USER_REQUIRED`, `FILE_REQUIRED`, `INVALID_FILE`, `FILE_TOO_LARGE`, `DOCUMENT_NOT_FOUND` e `INTERNAL_ERROR`. A mensagem pode ser localizada em português; clientes devem depender do `code`, não do texto.

## 8. Decisões arquiteturais

### Backend

- `routes/`: declara os endpoints e associa middleware de upload e controllers.
- `controllers/`: valida entrada HTTP, obtém identidade e traduz resultados/erros para status e respostas HTTP.
- `services/`: implementa regras de negócio, incluindo associação ao dono e autorização de acesso aos documentos.
- `repositories/`: encapsula gravação/leitura local de arquivos e metadados em memória.
- `multer` com `diskStorage` grava diretamente no diretório local configurado; storage filename é gerado no servidor.
- O controller coordena HTTP, mas não deve implementar regras de negócio ou acessar diretamente o filesystem.
- Não adicionar banco de dados nem dependência de armazenamento remoto.

### Frontend

- Componentes funcionais React organizados em `components/`, `pages/` e `services/`, de acordo com a estrutura existente.
- A camada de serviço encapsula `fetch` e usa `/api` como prefixo, conforme o proxy do Vite.
- A interface oferece envio de um arquivo, listagem dos metadados públicos, download e estados de feedback definidos nos requisitos funcionais.
- A identidade enviada em `X-User-Id` deve vir de configuração/contexto do cliente para o modo inicial; isso não representa autenticação segura.

## 9. Plano de execução

As etapas abaixo são um roteiro de implementação futuro. Nesta entrega, somente este documento de especificação deve ser criado; nenhum arquivo do backend ou frontend faz parte da execução atual.

1. Revisar e aprovar escopo, cabeçalho de identidade, limite de upload e formato dos erros antes de implementar.
2. Implementar no backend a configuração, o armazenamento local via `multer`/`diskStorage`, o repository de arquivos e o repository em memória, preservando as camadas definidas.
3. Implementar serviços, controllers e rotas de upload, listagem e download, incluindo validação de dono, respostas de erro e limpeza de arquivo em falhas.
4. Cobrir contratos e regras do backend com testes do runner nativo `node:test`, incluindo isolamento entre usuários, limites, falhas e acesso a arquivos.
5. Implementar no frontend o serviço HTTP e a experiência de upload, listagem, estados vazio/erro e download, seguindo o proxy `/api`.
6. Verificar o build do frontend e executar testes automatizados do backend; fazer uma validação integrada local dos fluxos completos.
7. Documentar configuração e execução local no README após a implementação e revisar os critérios de aceite desta especificação.

## 10. Critérios de aceite do produto

- Um upload válido gera arquivo no diretório local configurado e retorna metadados sem expor caminho interno.
- Um arquivo acima do limite configurado é rejeitado com `413` e não fica registrado como documento.
- A listagem retorna somente documentos do usuário requisitante e ordena por data decrescente.
- Download do próprio documento retorna os bytes e nome de arquivo apropriado; tentativa de baixar documento alheio resulta em `404`.
- Operações sem identidade são rejeitadas.
- A interface permite upload, exibe a listagem resultante e inicia downloads, apresentando estado vazio e mensagens de erro adequadas.
- O backend mantém a separação arquitetural e os metadados desaparecem após reinício, conforme as limitações declaradas.