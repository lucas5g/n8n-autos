# Spec: Atualizacao da criacao de chamados no Azure DevOps

## Objetivo

Atualizar o sub-workflow `tool-criar-chamado-azure` (`pjdK0m3bqs6aAPXF`) e o workflow chamador `GWhn18JSV7m4dw8F` para que a abertura de chamados passe a criar um `Epic` no projeto `CHAMADOS` do Azure DevOps, usando os novos campos do formulario.

## Escopo

Workflows envolvidos:

- Sub-workflow: `pjdK0m3bqs6aAPXF`
- Workflow pai/agente: `GWhn18JSV7m4dw8F`

Node relevante no sub-workflow:

- `criar-chamado-azure-devops`
- id: `b6e360c5-c711-4b5d-8e98-1bf1814b4585`

Node relevante no workflow pai:

- `tool-criar-chamado-azure`
- id: `b2766fff-3d59-4c55-9829-f01fb29f1700`
- tipo: `@n8n/n8n-nodes-langchain.toolWorkflow`

## Estado atual identificado

O sub-workflow ainda cria work item com esta configuracao:

- projeto Azure: `Teste Abrir Chamados`
- tipo Azure: `User Story`

URL atual:

```text
https://dev.azure.com/dpmg-sti/Teste%20Abrir%20Chamados/_apis/wit/workitems/%24User%20Story?api-version=7.1
```

O payload atual ainda usa o modelo antigo, com campos como:

- `regra_negocio_html`
- `criterios_aceite_html`
- `AcceptanceCriteria`

## Novo comportamento desejado

Criar `Epic` no projeto `CHAMADOS` com os campos abaixo:

- `Informations > Tipo de Chamado`
- `Informations > Sistema refente`
- `Description`
- `Demandante > Nome`
- `Demandante > Contato`
- `Demandante > Data Abertura`

Campos de entrada esperados no sub-workflow:

- `titulo`
- `descricao_html`
- `tipoChamado`
- `sistemaReferente`
- `nome`
- `contactEmail`
- `dataAbertura`

Confirmacoes feitas:

- `titulo` continua sendo o titulo do Epic
- `dataAbertura` deve ser preenchida automaticamente
- `contactEmail` deve alimentar o campo de contato
- `tipoChamado` deve usar exatamente estas opcoes:
  - `Correcao em sistema existente`
  - `Melhoria em sistema existente`
  - `Novo Sistema`

## Campos Azure confirmados

Campos confirmados por teste via API:

- `System.Title`
- `System.Description`
- `Custom.TipodeChamado`
- `Custom.Sistemarefente`
- `Custom.Contato`

Observacao:

- O campo visual esta escrito como `Sistema refente`, e o nome interno confirmado acompanha essa grafia: `Custom.Sistemarefente`.

## Campos ainda nao consolidados por referenceName

Apesar de a UI ter mostrado os ids internos do formulario, os nomes finais de API para estes campos ainda precisam ser confirmados antes de atualizar o workflow:

- `Data Abertura`
- `Nome`

Achados da interface:

- `32` => `Data Abertura`
- `50625150` => `Nome`

Exemplo capturado do formulario:

```json
[{"id":1625,"rev":3,"projectId":"af399ce3-b0b2-463f-b96c-1c5c62fd12ac","isDirty":true,"fields":{"32":"2026-03-31T19:38:00.000Z","50625150":"teste789"},"links":{},"multilineFieldsFormat":{}}]
```

Importante:

- Esses ids numericos parecem ser ids internos do formulario web.
- Ainda nao foi validado se a API do Work Item aceita esses numeros diretamente em `/fields/...`.
- Antes de implementar, confirmar se o Azure aceita:
  - `/fields/32`
  - `/fields/50625150`
- Se nao aceitar, descobrir o `referenceName` correspondente a cada campo.

## Problemas observados nos testes

### Nome

- O teste com `Custom.Nome` foi aceito pela API, mas o campo visual `Demandante > Nome` ficou vazio.
- Isso indica que `Custom.Nome` nao corresponde ao campo exibido no formulario.

### Data Abertura

- O teste com `Microsoft.VSTS.Scheduling.StartDate` preencheu `Planejamento > Start Date`, nao `Demandante > Data Abertura`.
- Portanto, `StartDate` nao deve ser usado como substituto de `Data Abertura`.

### Fuso horario

- Quando `StartDate` foi usado, a UI exibiu dia/hora deslocados por timezone.
- Isso reforca que o campo correto para `Data Abertura` deve ser o proprio campo customizado do formulario.

## Payload final esperado

O payload final do sub-workflow deve ficar conceitualmente assim:

```json
[
  { "op": "add", "path": "/fields/System.Title", "value": "<titulo>" },
  { "op": "add", "path": "/fields/System.Description", "value": "<descricao_html>" },
  { "op": "add", "path": "/fields/Custom.TipodeChamado", "value": "<tipoChamado>" },
  { "op": "add", "path": "/fields/Custom.Sistemarefente", "value": "<sistemaReferente>" },
  { "op": "add", "path": "<campo_nome>", "value": "<nome>" },
  { "op": "add", "path": "/fields/Custom.Contato", "value": "<contactEmail>" },
  { "op": "add", "path": "<campo_data_abertura>", "value": "<dataAbertura>" }
]
```

Onde:

- `<campo_nome>` ainda precisa ser confirmado
- `<campo_data_abertura>` ainda precisa ser confirmado

## URL final desejada no Azure

```text
https://dev.azure.com/dpmg-sti/CHAMADOS/_apis/wit/workitems/%24Epic?api-version=7.1
```

## Mudancas necessarias no sub-workflow `pjdK0m3bqs6aAPXF`

1. Alterar a URL do node `criar-chamado-azure-devops` para `CHAMADOS` + `%24Epic`.
2. Remover do `jsonBody` os campos antigos:
   - `regra_negocio_html`
   - `criterios_aceite_html`
   - `Custom.58212133-dce5-4aa5-a851-6231e70a9d68`
   - `Microsoft.VSTS.Common.AcceptanceCriteria`
3. Passar a enviar:
   - `System.Title`
   - `System.Description`
   - `Custom.TipodeChamado`
   - `Custom.Sistemarefente`
   - campo correto de `Nome`
   - `Custom.Contato`
   - campo correto de `Data Abertura`
4. Atualizar o `jsonExample` do trigger `When Executed by Another Workflow` para refletir os novos inputs.
5. Revisar o node de e-mail se for necessario incluir os novos campos no corpo da notificacao.

## Mudancas necessarias no workflow pai `GWhn18JSV7m4dw8F`

No node `tool-criar-chamado-azure`:

1. Remover inputs antigos:
   - `regra_negocio_html`
   - `criterios_aceite_html`
2. Passar a enviar os novos campos:
   - `titulo`
   - `descricao_html`
   - `tipoChamado`
   - `sistemaReferente`
   - `nome`
   - `contactEmail`
   - `dataAbertura`
3. Atualizar o schema do node tool para refletir os novos campos.
4. Ajustar o agente para produzir:
   - `titulo`
   - `descricao_html`
   - `tipoChamado`
   - `sistemaReferente`
5. Preencher automaticamente:
   - `nome` a partir dos dados do Chatwoot
   - `contactEmail` com o valor ja existente
   - `dataAbertura` com a data/hora atual no momento da execucao

## Origem atual dos dados no workflow pai

Ja identificado:

- `contactEmail` vem de `chatwoot-dados`
- atualmente o tool envia:
  - `titulo`
  - `descricao_html`
  - `regra_negocio_html`
  - `criterios_aceite_html`
  - `contactEmail`

Pendente de incluir no fluxo pai:

- `tipoChamado`
- `sistemaReferente`
- `nome`
- `dataAbertura`

## Testes ja executados

### Validacoes bem-sucedidas

Foi validado com sucesso que estes campos sao aceitos pela API do Epic em `CHAMADOS`:

- `System.Title`
- `System.Description`
- `Custom.TipodeChamado`
- `Custom.Sistemarefente`
- `Custom.Contato`

### Validacoes que nao servem para o formulario correto

- `Custom.Nome`
  - aceito, mas nao preencheu o campo visual `Demandante > Nome`
- `Microsoft.VSTS.Scheduling.StartDate`
  - aceito, mas preencheu `Planejamento > Start Date`

### Validacoes que falharam

- `Custom.SistemaReferente`
  - erro: campo nao encontrado
- `Custom.DataAbertura`
  - erro: campo nao encontrado

## Proximos passos recomendados

1. Confirmar o nome de API do campo `Nome`.
2. Confirmar o nome de API do campo `Data Abertura`.
3. Validar se a API aceita os ids numericos do formulario:
   - `/fields/32`
   - `/fields/50625150`
4. Se nao aceitar, localizar o `referenceName` correto desses dois campos.
5. Atualizar o sub-workflow `pjdK0m3bqs6aAPXF`.
6. Atualizar o workflow pai `GWhn18JSV7m4dw8F`.
7. Executar um teste real de criacao de Epic.
8. Confirmar na UI se todos os campos foram preenchidos nos lugares corretos.

## Curls uteis para retomar

### Validar criacao sem salvar

```bash
curl --request POST \
  --url "https://dev.azure.com/dpmg-sti/CHAMADOS/_apis/wit/workitems/\$Epic?api-version=7.1&validateOnly=true" \
  --user ":SEU_PAT" \
  --header "Content-Type: application/json-patch+json" \
  --data '[
    { "op": "add", "path": "/fields/System.Title", "value": "Solicitacao curta e objetiva" },
    { "op": "add", "path": "/fields/System.Description", "value": "<p>Descricao objetiva do chamado</p>" },
    { "op": "add", "path": "/fields/Custom.TipodeChamado", "value": "Novo Sistema" },
    { "op": "add", "path": "/fields/Custom.Sistemarefente", "value": "Sistema X" },
    { "op": "add", "path": "/fields/Custom.Contato", "value": "usuario@exemplo.com" }
  ]'
```

### Criar de fato

```bash
curl --request POST \
  --url "https://dev.azure.com/dpmg-sti/CHAMADOS/_apis/wit/workitems/\$Epic?api-version=7.1" \
  --user ":SEU_PAT" \
  --header "Content-Type: application/json-patch+json" \
  --data '[
    { "op": "add", "path": "/fields/System.Title", "value": "Solicitacao curta e objetiva" },
    { "op": "add", "path": "/fields/System.Description", "value": "<p>Descricao objetiva do chamado</p>" },
    { "op": "add", "path": "/fields/Custom.TipodeChamado", "value": "Novo Sistema" },
    { "op": "add", "path": "/fields/Custom.Sistemarefente", "value": "Sistema X" },
    { "op": "add", "path": "/fields/Custom.Contato", "value": "usuario@exemplo.com" }
  ]'
```

### Teste para os ids numericos descobertos na UI

```bash
curl --request POST \
  --url "https://dev.azure.com/dpmg-sti/CHAMADOS/_apis/wit/workitems/\$Epic?api-version=7.1&validateOnly=true" \
  --user ":SEU_PAT" \
  --header "Content-Type: application/json-patch+json" \
  --data '[
    { "op": "add", "path": "/fields/System.Title", "value": "Teste" },
    { "op": "add", "path": "/fields/System.Description", "value": "<p>Teste</p>" },
    { "op": "add", "path": "/fields/32", "value": "2026-03-31T19:38:00.000Z" },
    { "op": "add", "path": "/fields/50625150", "value": "teste789" }
  ]'
```

## Resultado esperado apos implementacao

Ao acionar o workflow pai, o agente deve abrir um Epic no Azure DevOps com:

- titulo preenchido
- descricao preenchida
- tipo de chamado preenchido
- sistema refente preenchido
- nome do demandante preenchido no campo correto
- contato preenchido
- data de abertura preenchida no campo correto