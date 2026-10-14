# Fluxo CRM

CRM de funil de vendas com chats unificados, tarefas, automações por etapa e cobrança por plano.
O modelo de dados e o fluxo de trabalho seguem o Kommo, com código, nome e visual próprios.

## Rodando

Requer **Node.js 24+**. Não há dependências nativas: o banco é o `node:sqlite` embutido.

```bash
npm install
npm run dev          # http://localhost:3000  (API + Vite com hot reload no mesmo processo)
```

Em `npm run dev`, o banco `data/fluxo.db` é criado com dados de demonstração na primeira execução (`FLUXO_DEMO=1`).
Em produção a instalação começa vazia; crie o administrador com `npm run setup -- --email voce@empresa.com --name "Seu Nome"`.

**Instalação em servidor:** veja [docs/INSTALACAO-VPS-CONTABO.md](docs/INSTALACAO-VPS-CONTABO.md).
**Referência de produto:** [docs/ENGENHARIA-REVERSA-KOMMO.md](docs/ENGENHARIA-REVERSA-KOMMO.md).

### Variáveis de ambiente

| Variável | Padrão | Para que serve |
|---|---|---|
| `PORT` / `HOST` | `3000` / `0.0.0.0` | Onde o servidor escuta (em produção use `HOST=127.0.0.1` atrás do Nginx) |
| `FLUXO_DB` | `data/fluxo.db` | Arquivo do banco SQLite |
| `FLUXO_INBOUND_KEY` | `dev-inbound-key` | Chave do webhook de mensagens recebidas |
| `FLUXO_BILLING` | *(desligada)* | `enabled` liga período de teste, planos, limites e a página *Plano e pagamento*. Desligada, a instância é ilimitada e não exibe cobrança |
| `FLUXO_DEMO` | — | `1` popula um banco vazio com dados de demonstração |

| Usuário | E-mail | Senha | Perfil |
|---|---|---|---|
| Admin | admin@fluxo.local | fluxo123 | Administrador |
| Mariana Costa | mariana@fluxo.local | fluxo123 | Gerente |
| Rafael Lima | rafael@fluxo.local | fluxo123 | Usuário |

Outros comandos:

```bash
npm run setup -- --email x@y.com [--name "Nome"]   # cria a conta (1ª vez) ou redefine a senha de um usuário
npm run seed         # apaga o banco e recria os dados de demonstração (recusado com NODE_ENV=production)
npm run build        # gera client/dist
npm start            # produção: serve client/dist + API (PORT, FLUXO_DB, FLUXO_INBOUND_KEY)
npm run typecheck
```

## O que tem

| Área | Funcionalidades |
|---|---|
| **Leads** | Kanban por funil com arrastar e soltar, soma por etapa e adição rápida. Filtros rápidos: ativos, meus, ganhos, perdidos, sem tarefas, com tarefas atrasadas. Coluna *Leads de entrada* com Aceitar/Recusar. Barra Ganho/Perdido/Excluir ao arrastar, com motivo de perda. Visão em lista com ações em massa. Filtros por usuário, tag e busca (inclui telefone e e-mail do contato) |
| **Card do lead** | Cabeçalho com etapa e dias na etapa, abas Principal/Estatísticas. Campos editáveis inline, barra de progresso, tags, campos personalizados, contatos e empresa vinculados. Feed cronológico (notas, ligações, eventos, mensagens dos chats) com tarefas pendentes fixadas no fim. Composer Nota / Tarefa / Chat |
| **Chats** | Caixa de entrada unificada com **envio real pelo WhatsApp** (API oficial da Meta ou QR Code via Evolution API), status ✓/✓✓/lida, regra da janela de 24h e templates. Outros canais via webhook. Não lidas, resposta e atalho para o lead. Mensagem de contato desconhecido cria contato + chat + lead em *Leads de entrada* |
| **Calendário** | Visões Dia, Semana e Mês com arrastar para remarcar e clique para criar, além da Lista agrupada em Atrasadas / Hoje / Amanhã / Esta semana / Mais tarde. Tipos (follow-up, ligação, reunião, e-mail) e conclusão com resultado registrado no feed |
| **Listas** | Contatos e empresas com busca e ficha própria (campos, leads, feed) |
| **Início e Insights** | Períodos Hoje/Ontem/Semana/Mês/Ano e filtro por usuário. Mensagens por canal, conversas não respondidas, tempo de resposta, receita ganha, ticket médio, ciclo de venda, conversão, funil por etapa, receita por dia, motivos de perda, origem dos leads, desempenho por usuário |
| **Configurações** | Perfil (nome, cor, senha), funis e etapas (cores, ordem, etapas de sistema), gestão de usuários com grupos e contadores, motivos de perda, campos personalizados, tags, automações, integrações, **Plano e pagamento** |
| **Automações** | Gatilhos *ao entrar na etapa* e *ao receber mensagem* (qualquer, primeiro contato ou palavra-chave; opcionalmente só fora do horário de atendimento). Ações: enviar mensagem, criar tarefa, nota, tag, responsável, mover etapa. Atraso opcional com "cancelar se o cliente responder", e variáveis `{{nome}}`, `{{lead}}`, `{{responsavel}}` e `{{empresa}}` |
| **Plano e pagamento** | Só com `FLUXO_BILLING=enabled`. Período de teste de 14 dias. Três planos por usuário/mês, contratação por 6 ou 12 meses, quantidade de usuários e total. Checkout por cartão, Pix ou boleto, com faturas. Limites por plano: automações a partir do Avançado, número de funis, número de usuários |

## Arquitetura

```
server/
  db.ts       schema SQLite (node:sqlite) + helpers q.all/get/run/tx
  domain.ts   regras: auth (scrypt + sessões), mudança de etapa, automações, mensagens de entrada, feed
  api.ts      rotas REST (Express 5)
  seed.ts     dados de demonstração
  index.ts    servidor HTTP; em dev monta o Vite como middleware
client/src/
  App.tsx     shell (sidebar, banners, roteamento por hash)
  pages/      Dashboard, Leads (kanban/lista), LeadCard, Chats, Tasks, Lists, Stats, Settings
  components/ Feed, TaskForm, Charts, ui (Modal, InlineEdit, …)
```

### Correspondência com o modelo do Kommo (API v4 pública)

| Kommo | Fluxo |
|---|---|
| `leads` com `pipeline_id` + `status_id`, `price`, `responsible_user_id`, `loss_reason` | tabela `leads` |
| Etapa *Incoming leads* (não triados) | `statuses.type = 'incoming'` |
| Status fixos 142 (ganho) / 143 (perdido) | `statuses.type = 'won' \| 'lost'` |
| `contacts`, `companies`, vínculo lead↔contato com contato principal | `contacts`, `companies`, `lead_contacts.is_main` |
| `notes` e `tasks` polimórficas (`entity_type` + `entity_id`) | idem |
| `custom_fields_values` | `custom` (JSON) + `custom_fields` |
| Funil digital / Salesbot (gatilho na etapa) | `automations` |

## Integrações

**Mensagens recebidas:** os canais enviam cada mensagem para

```
POST /api/inbound/message
x-inbound-key: $FLUXO_INBOUND_KEY        (padrão em dev: dev-inbound-key)
{ "channel": "whatsapp", "name": "Maria", "phone": "+5511999990000", "text": "Olá!" }
```

**API:** `POST /api/auth/login` devolve um token. Envie-o em `Authorization: Bearer <token>`.
Os recursos são `/api/leads`, `/api/leads/bulk`, `/api/contacts`, `/api/companies`, `/api/tasks`, `/api/notes`, `/api/chats`,
`/api/stats`, `/api/pipelines`, `/api/statuses`, `/api/users`, `/api/tags`, `/api/custom_fields`, `/api/automations` e `/api/billing`.

## Antes de ir para produção

- **Pagamento:** o checkout é simulado (cartão aprova na hora; Pix e boleto ficam pendentes até a confirmação manual).
  Integre um gateway (Stripe, Mercado Pago, Pagar.me) em `POST /api/billing/checkout`, tokenizando o cartão no navegador.
  O webhook do gateway deve chamar a lógica de `activateInvoice`.
- **Multi-conta:** hoje cada instalação é uma conta. Para SaaS, adicione `account_id` às tabelas.
- Defina `FLUXO_INBOUND_KEY` e rode atrás de HTTPS.
