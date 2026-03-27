# Spec - Integracao Chatwoot + n8n para atendimento com widget e controle de excecao

## Contexto

Hoje existe um workflow n8n com ID `TdG6aBvW8qUe62M9` e nome `agente-chamados-v0.2` que usa `Chat Trigger` nativo do n8n para conversar com usuarios e, quando apropriado, abrir chamados no Azure DevOps por meio da tool `criar_chamado_azure`.

A proposta desta spec e substituir a camada de interface do `Chat Trigger` por `Chatwoot`, mantendo o n8n como motor de orquestracao e IA. O principal motivo da mudanca e ganhar uma experiencia visual melhor no site, simplificar instalacao em multiplos sites e preparar a base para futura operacao humana, sem depender disso na primeira versao.

## Configuracao conhecida do Chatwoot

- `base_url`: `https://dev.chatwoot.mg.def.br`
- `tipo_instancia`: `self-hosted`
- `account_id`: `1`
- `inbox_id`: `12`
- `website_token`: `mew1EgJVcZToPgE3ZvLhZ4uB`
- `api_access_token`: configurado e disponivel para integracao

## Objetivo

Implementar uma integracao em que:

- o `Chatwoot` seja o widget oficial dos sites;
- o `n8n` processe as mensagens recebidas e gere as respostas do agente;
- o fluxo continue apto a abrir chamados no Azure DevOps;
- exista caminho claro para evoluir depois para atendimento humano sem trocar de widget;
- a conversa preserve contexto basico do visitante e estado operacional da automacao.

## Resultado esperado

- visitante conversa pelo widget do `Chatwoot` no site;
- mensagens do visitante chegam ao `n8n` por webhook;
- o `n8n` decide entre responder com IA, pedir mais contexto, abrir chamado ou bloquear a automacao por excecao;
- respostas do agente sao publicadas na mesma conversa do `Chatwoot`;
- quando houver excecao, a conversa fica marcada para revisao sem que o bot continue respondendo.

## Escopo da primeira versao

### Incluido

- widget do `Chatwoot` instalado nos sites;
- inbox de website no `Chatwoot`;
- webhook do `Chatwoot` para notificar `message_created` no `n8n`;
- workflow n8n para processar mensagens recebidas;
- resposta automatica do agente na conversa original;
- persistencia minima de identificadores da conversa;
- regra de controle de excecao por labels;
- continuidade da integracao com Azure DevOps.

### Fora do escopo da primeira versao

- omnichannel completo com WhatsApp, Instagram ou email;
- roteamento multidepartamento;
- analytics avancado e dashboards executivos;
- autenticao SSO do visitante no widget;
- memoria de longo prazo compartilhada fora da propria conversa.

## Arquitetura proposta

### Componentes

- `Site`: hospeda o snippet do widget do `Chatwoot`.
- `Chatwoot`: interface conversacional, inbox, historico e futura operacao humana.
- `n8n`: recebe eventos, orquestra IA, regras de negocio e integracoes.
- `Workflow atual`: base logica do agente de chamados, adaptada para entrada e saida via `Chatwoot`.
- `Azure DevOps`: destino final para criacao do chamado.

### Fluxo de alto nivel

1. Usuario abre o widget do `Chatwoot` no site.
2. Usuario envia mensagem.
3. `Chatwoot` dispara webhook para um `Webhook Trigger` do `n8n`.
4. O `n8n` valida assinatura do webhook, filtra eventos irrelevantes e carrega contexto da conversa.
5. O `n8n` decide se deve:
   - responder automaticamente;
   - solicitar mais informacoes;
   - abrir chamado no Azure DevOps;
   - bloquear a automacao por excecao.
6. Quando houver resposta automatica, o `n8n` envia a mensagem de volta para a API do `Chatwoot`.
7. Quando houver excecao, o `n8n` marca a conversa com labels operacionais e interrompe novas respostas automaticas.

## Modelo operacional

### Canal oficial

- usar `Website Inbox` do `Chatwoot` como canal oficial do widget;
- embed feito pelo snippet nativo do `Chatwoot` ou por GTM, dependendo do site;
- nesta primeira versao, nao habilitar identificacao autenticada do usuario.

### Papel do n8n

- atuar como backend de automacao e inteligencia;
- centralizar regras de negocio do agente;
- encapsular a integracao com Azure DevOps;
- decidir quando permanecer no autoatendimento e quando interromper a automacao por excecao.

### Papel do Chatwoot

- ser a interface de conversa;
- armazenar historico da interacao;
- permitir acompanhamento operacional das conversas;
- concentrar a base para futura evolucao de suporte humano.

## Eventos e integracoes

### Entrada principal

Evento principal inicial: `message_created` do `Chatwoot`.

Filtrar apenas mensagens:

- `incoming`;
- nao privadas;
- originadas do contato, nao do agente;
- associadas ao inbox do assistente.

### Eventos futuros recomendados

- `conversation_created` para inicializacao de metadados;
- `conversation_status_changed` para sincronizar estados;
- `webwidget_triggered` para registrar abertura sem mensagem;
- `message_updated` se houver edicao relevante.

### Integracoes de saida

- API do `Chatwoot` para publicar mensagem na conversa;
- API do `Chatwoot` para atualizar status, labels e, futuramente, atributos adicionais;
- workflow `criar_chamado_azure` para abertura do chamado.

## Adaptacao do workflow atual

O workflow `TdG6aBvW8qUe62M9` hoje depende do `Chat Trigger` do n8n. Para suportar `Chatwoot`, a logica precisa ser desacoplada da interface de entrada.

### Mudancas estruturais

- trocar o gatilho de `Chat Trigger` por `Webhook Trigger` para eventos do `Chatwoot`;
- extrair a logica do agente para um fluxo reutilizavel que receba texto, identificadores e contexto da conversa;
- manter a ferramenta `criar_chamado_azure` como parte do fluxo de decisao;
- substituir a memoria nativa do chat por memoria baseada em historico do `Chatwoot`.

### Entrada canonica sugerida para o agente

O agente deve receber, no minimo:

- `conversation_id`
- `contact_id`
- `message_id`
- `chat_input`
- `contact_name`
- `contact_email` quando existir
- `browser_context` quando vier do webhook
- `labels_atuais`
- `status_conversa`
- `site_origem`

### Saida canonica sugerida do agente

O agente deve devolver um objeto estruturado com:

- `acao`: `responder`, `criar_chamado`, `marcar_excecao`, `ignorar`
- `texto_resposta`
- `nota_interna`
- `dados_chamado`
- `atualizacoes_chatwoot`

## Controle de excecao

### Objetivo

Permitir que a automacao pare de responder uma conversa de forma controlada, rastreavel e visivel no Chatwoot sempre que surgir uma condicao que exija revisao.

### Gatilhos recomendados para excecao

- agente detectar baixa confianca ou ambiguidade alta;
- falha tecnica na integracao com Azure DevOps;
- topico fora de escopo do agente automatizado;
- excesso de voltas sem resolucao.

### Acoes de excecao no Chatwoot

- adicionar label `precisa_revisao`;
- alterar status da conversa para `open`;
- registrar nota interna com resumo do contexto coletado e do motivo do bloqueio;
- interromper novas respostas automaticas enquanto a conversa estiver marcada com a label de excecao.

### Regra anti-conflito

Depois da excecao:

- o bot nao responde mais naquela conversa enquanto a label `precisa_revisao` existir;
- a remocao manual da label reativa a automacao;
- labels e nota interna definem o estado operacional da conversa.

## Estado da conversa

Sugestao de estados logicos controlados por labels no `Chatwoot`:

- `precisa_revisao`
- `chamado_criado`

Regras da v1:

- ausencia da label `precisa_revisao` significa que o bot pode operar;
- presenca da label `precisa_revisao` significa que o bot deve permanecer silencioso;
- a label `chamado_criado` pode ser usada para visibilidade operacional apos sucesso no Azure DevOps.

Essas labels nao substituem o status nativo do `Chatwoot`; elas complementam o controle da automacao.

## Persistencia e memoria

### Abordagem recomendada para v1

Usar o historico da propria conversa no `Chatwoot` como fonte principal de contexto e complementar com labels e notas internas.

### Opcoes de apoio

- `labels` para roteamento e observabilidade;
- armazenamento externo futuro se houver necessidade de memoria longa ou busca semantica.

### Decisao para v1

Nao introduzir banco adicional na primeira entrega, a menos que a latencia ou limite de contexto se tornem problema real.

## Seguranca

- validar assinatura HMAC dos webhooks do `Chatwoot`;
- restringir o webhook do n8n a eventos do inbox esperado;
- proteger tokens da API do `Chatwoot` e credenciais do Azure no n8n;
- nao expor links internos do Azure ao usuario final;
- registrar erros tecnicos sem vazar credenciais ou payloads sensiveis;
- como a v1 nao possui autenticacao do usuario no site, nao usar validacao de identidade nesta etapa.

## Requisitos funcionais

### RF01 - Widget

O sistema deve disponibilizar um widget de chat embutivel em sites por meio do `Chatwoot`.

### RF02 - Recepcao de mensagens

O sistema deve receber mensagens de visitantes enviadas pelo widget e encaminha-las ao fluxo do agente no `n8n`.

### RF03 - Resposta automatica

O sistema deve publicar respostas do agente na mesma conversa do `Chatwoot`.

### RF04 - Abertura de chamado

O sistema deve preservar a capacidade de abrir chamado no Azure DevOps apos coleta de contexto e confirmacao explicita do usuario.

### RF05 - Controle de excecao

O sistema deve permitir interromper a automacao da conversa por meio de labels operacionais, mantendo historico e contexto.

### RF06 - Bloqueio de resposta indevida

O sistema deve evitar que o bot continue respondendo conversas marcadas com label de excecao.

### RF07 - Rastreabilidade

O sistema deve registrar os identificadores essenciais da conversa, do contato e do chamado criado.

## Requisitos nao funcionais

### RNF01 - Simplicidade de instalacao

O widget deve poder ser instalado por snippet ou gerenciador de tags, sem dependencia do `Chat Trigger` do n8n.

### RNF02 - Baixo acoplamento

A logica do agente deve ficar desacoplada da tecnologia de widget.

### RNF03 - Observabilidade

Deve ser possivel identificar falhas por etapa: recepcao do webhook, processamento no n8n, envio ao Chatwoot e abertura no Azure.

### RNF04 - Evolucao gradual

O desenho deve permitir iniciar com automacao pura e evoluir depois para operacao hibrida com humano.

## API e dados minimos da integracao

### Do Chatwoot para o n8n

Campos minimos a mapear do webhook:

- `event`
- `id` da mensagem
- `content`
- `message_type`
- `conversation.id`
- `conversation.status`
- `contact.id`
- `contact.name`
- `sender.type`
- `inbox.id`
- `account.id`
- `conversation.additional_attributes.referer`

### Do n8n para o Chatwoot

Operacoes minimas:

- enviar mensagem publica na conversa;
- adicionar ou remover labels;
- registrar nota interna;
- opcionalmente alterar status.

## Fluxos principais

### Fluxo A - Atendimento automatizado normal

1. usuario envia mensagem;
2. webhook chega ao n8n;
3. n8n monta contexto;
4. agente responde;
5. resposta e publicada no Chatwoot;
6. conversa segue em modo bot.

### Fluxo B - Abertura de chamado

1. usuario descreve demanda;
2. agente coleta contexto aos poucos;
3. agente apresenta resumo estruturado;
4. usuario confirma explicitamente;
5. n8n chama `criar_chamado_azure`;
6. n8n responde no Chatwoot com o ID do chamado.

### Fluxo C - Excecao operacional

1. o bot detecta baixa confianca, erro tecnico, fora de escopo ou repeticao sem progresso;
2. n8n registra nota interna com contexto e motivo;
3. n8n marca a label `precisa_revisao` e mantem a conversa `open`;
4. bot deixa de responder;
5. a conversa fica visivel para revisao manual futura, sem continuidade automatica.

## Regras de negocio iniciais

- o agente automatizado atende somente assuntos relacionados ao sistema GERAIS;
- o agente nao deve afirmar que criou chamado sem sucesso real do Azure DevOps;
- o `id` numerico do Azure e o identificador oficial apresentado ao usuario;
- links internos ou IDs tecnicos auxiliares nao devem ser exibidos ao usuario;
- para login e acesso, o agente nao deve perguntar modulo antes do proprio acesso ao sistema;
- quando a conversa estiver com a label `precisa_revisao`, o bot deve ficar silencioso.

## Riscos e mitigacoes

### Risco 1 - Resposta duplicada

Bot responder apos a conversa entrar em estado de excecao.

Mitigacao:

- usar a label `precisa_revisao` como bloqueio;
- checar estado antes de cada resposta automatica.

### Risco 2 - Loops de webhook

Mensagem enviada pelo bot disparar nova automacao.

Mitigacao:

- processar apenas mensagens `incoming` do contato;
- ignorar mensagens de agente, sistema ou nota privada.

### Risco 3 - Perda de contexto

Memoria do fluxo ficar inferior a conversa real.

Mitigacao:

- reconstruir contexto a partir do historico relevante do `Chatwoot`;
- resumir contexto em atributos quando necessario.

### Risco 4 - Acoplamento excessivo ao Chat Trigger atual

Partes da logica atual dependerem do formato nativo do `Chat Trigger`.

Mitigacao:

- criar uma camada de entrada canonica;
- separar interface, orquestracao e regras do agente.

## Criterios de aceite da funcionalidade

- e possivel instalar o widget em um site usando `Chatwoot`;
- uma mensagem enviada no widget chega ao `n8n` com os metadados minimos esperados;
- o `n8n` consegue responder na mesma conversa;
- o fluxo de abertura de chamado continua funcional;
- a label `precisa_revisao` bloqueia novas respostas automaticas;
- a remocao manual da label permite reativar a automacao na mesma thread.

## Plano sugerido de implementacao

### Fase 1 - Fundacao

- validar acesso ao inbox de website no `Chatwoot`;
- configurar widget e instalacao no site com o `website_token` conhecido;
- criar credenciais de API e webhook;
- definir labels operacionais.

### Fase 2 - Integracao basica

- criar workflow n8n de entrada por webhook;
- validar assinatura do `Chatwoot`;
- filtrar eventos;
- enviar resposta simples de teste para a conversa.

### Fase 3 - Adaptacao do agente

- adaptar o fluxo atual para entrada canonica;
- plugar memoria e regras do agente;
- manter integracao com `criar_chamado_azure`.

### Fase 4 - Controle de excecao

- definir gatilhos de bloqueio;
- atualizar labels, nota interna e status;
- bloquear respostas do bot em conversas com excecao.

### Fase 5 - Hardening

- observabilidade;
- testes de erro;
- controles de idempotencia;
- refinamento de prompt e operacao.

## Decisoes pendentes para alinhamento

- label de excecao definida como `precisa_revisao`;
- decidir se a nota interna sera unica ou acumulativa por conversa;
- decidir se a memoria v1 sera somente historico do `Chatwoot` ou se depois havera armazenamento auxiliar;
- definir criterio exato de reativacao do bot apos remocao da label;
- definir quando a operacao humana futura sera incorporada.

## Recomendacao final

Seguir com `Chatwoot` como camada oficial de widget e atendimento, e manter o `n8n` como backend de automacao e IA. Esta separacao melhora a experiencia do usuario, prepara o caminho para operacao humana e reduz o risco de retrabalho caso o atendimento evolua para uma fila real de suporte.
