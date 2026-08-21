#  Automação de Alertas com IA - Zabbix/Grafana → VerdanDesk → Notion

Automação que reproduz um fluxo real usado em ambiente de NOC/Service Desk:
alertas de monitoramento são recebidos, **triados por IA (Claude/ChatGPT)**,
viram chamados automáticos no **VerdanDesk** quando necessário, e são
registrados numa base do **Notion** para histórico e análise de recorrência.

##  Problema que resolve

Em operações de NOC, alertas de Zabbix/Grafana (conexão caindo, temperatura
de nobreak, disco cheio, etc.) geravam trabalho manual repetitivo: analisar o
alerta, decidir a severidade, abrir chamado e documentar. Essa automação
elimina a etapa manual de triagem, usando IA para resumir o problema e sugerir
a primeira ação - o analista recebe o chamado já com contexto.

##  Arquitetura

```
Zabbix / Grafana (alerta)
        │
        ▼
  config.yaml  ──►  regras de negócio (severidade, SLA, cria ticket?)
        │
        ▼
   automation.py
        │
        ├──► IA (Claude ou ChatGPT): resume o alerta e sugere 1ª ação
        │
        ├──► VerdanDesk: abre chamado (se a regra exigir)
        │
        └──► Notion: registra alerta + ticket + análise da IA
```

##  Tecnologias

- **Python** - orquestração da automação
- **YAML** - configuração declarativa de regras, thresholds e integrações
- **Notion API** - histórico e base de conhecimento pesquisável
- **VerdanDesk API** - abertura automática de chamados
- **API de IA (Claude / ChatGPT)** - triagem, resumo e sugestão de ação

##  Estrutura

```
notion-ai-ticket-automation/
├── config.yaml       # regras de alerta, SLAs e credenciais (via env vars)
├── automation.py      # script principal do fluxo
└── README.md
```

##  Configuração

Todas as credenciais ficam em variáveis de ambiente, referenciadas no
`config.yaml` como `${NOME_DA_VARIAVEL}`:

```bash
export ZABBIX_WEBHOOK_URL="..."
export GRAFANA_WEBHOOK_URL="..."
export VERDANDESK_API_TOKEN="..."
export NOTION_API_TOKEN="..."
export NOTION_DATABASE_ID="..."
export AI_API_KEY="..."
```

##  Como rodar

```bash
pip install pyyaml requests
python automation.py
```

O script já inclui um alerta de exemplo (`nobreak_temperature`) para
demonstrar o fluxo ponta a ponta sem precisar de um webhook real.

##  Possíveis evoluções futuras

- Expor um endpoint (Flask/FastAPI) para receber os webhooks do Zabbix/Grafana em tempo real
- Adicionar deduplicação de alertas recorrentes (evitar ticket duplicado)
- Dashboard no Notion com filtros por severidade e tempo médio de resolução
- Fine-tuning do prompt de IA para classificar automaticamente falsos positivos
