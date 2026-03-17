# n8n-autos

Servidor MCP local para consultar a API publica do n8n usando `stdio`.

## Requisitos

- Node.js 18+
- um arquivo `.env` com `N8N_BASE_URL` e `N8N_API_KEY`

## Instalar

```bash
npm install
```

## Rodar localmente

```bash
npm start
```

## Ferramentas MCP disponiveis

- `n8n_health_check`
- `n8n_list_workflows`
- `n8n_get_workflow`
- `n8n_list_executions`
- `n8n_get_execution`

## Exemplo de configuracao em cliente MCP

Use o comando abaixo no cliente que aceite servidores MCP via `stdio`:

```json
{
  "mcpServers": {
    "n8n-local": {
      "command": "node",
      "args": ["/home/lucas/projects/n8n-auto/src/index.js"],
      "cwd": "/home/lucas/projects/n8n-auto"
    }
  }
}
```
