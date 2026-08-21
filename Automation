"""
automation.py
--------------
Automação de alertas de monitoramento -> Ticket (VerdanDesk) -> Notion -> IA.

Fluxo:
1. Recebe um alerta (Zabbix ou Grafana) via webhook/payload.
2. Consulta a IA (Claude/ChatGPT) para resumir o alerta, sugerir a
   primeira ação de troubleshooting e recomendar a severidade.
3. Se a regra em config.yaml indicar `auto_create_ticket: true`,
   abre um chamado no VerdanDesk já com o contexto da IA.
4. Registra o alerta, o ticket e a análise da IA numa base do Notion,
   criando um histórico consultável de incidentes recorrentes.

Este script é a versão "portfólio" da automação usada em produção:
as chamadas HTTP reais para VerdanDesk/Notion/IA ficam isoladas em
funções próprias, fáceis de trocar por chamadas reais (requests.post)
quando as credenciais e endpoints estiverem configurados via variáveis
de ambiente.
"""

import os
import yaml
import logging
import requests
from datetime import datetime
from string import Formatter

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(message)s",
)
log = logging.getLogger("notion-ai-ticket-automation")


def load_config(path: str = "config.yaml") -> dict:
    """Carrega o config.yaml e resolve variáveis de ambiente no formato ${VAR}."""
    with open(path, "r", encoding="utf-8") as f:
        raw = f.read()

    for key, value in os.environ.items():
        raw = raw.replace(f"${{{key}}}", value)

    return yaml.safe_load(raw)


def find_rule(config: dict, trigger: str) -> dict | None:
    for rule in config["monitoring"]["alert_rules"]:
        if rule["trigger"] == trigger:
            return rule
    return None


def ask_ai(config: dict, alert: dict) -> dict:
    """
    Envia o alerta para a IA (Claude ou ChatGPT) e retorna:
    resumo, sugestão de primeira ação e severidade recomendada.
    """
    ai_cfg = config["ai_assistant"]
    prompt = ai_cfg["prompt_template"].format(**alert)

    log.info("Consultando IA (%s) para triagem do alerta '%s'", ai_cfg["provider"], alert["trigger"])

    if ai_cfg["provider"] == "claude":
        response_text = _call_claude(ai_cfg, prompt)
    else:
        response_text = _call_openai(ai_cfg, prompt)

    # Em produção: parsear a resposta estruturada (ou pedir JSON).
    # Aqui devolvemos um resultado simplificado para fins de exemplo.
    return {
        "ai_summary": response_text.get("summary", ""),
        "ai_first_action": response_text.get("first_action", ""),
        "ai_recommended_severity": response_text.get("severity", alert["severity"]),
    }


def _call_claude(ai_cfg: dict, prompt: str) -> dict:
    """Chamada real à API da Anthropic (Claude)."""
    headers = {
        "x-api-key": ai_cfg["api_key"],
        "anthropic-version": "2023-06-01",
        "content-type": "application/json",
    }
    payload = {
        "model": ai_cfg["model"],
        "max_tokens": ai_cfg["max_tokens"],
        "messages": [{"role": "user", "content": prompt}],
    }
    resp = requests.post("https://api.anthropic.com/v1/messages", json=payload, headers=headers, timeout=30)
    resp.raise_for_status()
    data = resp.json()
    text = "".join(block.get("text", "") for block in data.get("content", []))
    return _parse_ai_text(text)


def _call_openai(ai_cfg: dict, prompt: str) -> dict:
    """Chamada real à API da OpenAI (ChatGPT)."""
    headers = {
        "Authorization": f"Bearer {ai_cfg['api_key']}",
        "Content-Type": "application/json",
    }
    payload = {
        "model": ai_cfg["model"],
        "max_tokens": ai_cfg["max_tokens"],
        "messages": [{"role": "user", "content": prompt}],
    }
    resp = requests.post("https://api.openai.com/v1/chat/completions", json=payload, headers=headers, timeout=30)
    resp.raise_for_status()
    data = resp.json()
    text = data["choices"][0]["message"]["content"]
    return _parse_ai_text(text)


def _parse_ai_text(text: str) -> dict:
    """Parser simples: adapte conforme o formato de saída pedido no prompt."""
    return {
        "summary": text[:280],
        "first_action": "Ver recomendação completa no corpo da resposta da IA.",
        "severity": None,
    }


def create_verdandesk_ticket(config: dict, alert: dict, ai_result: dict) -> str:
    """Abre um chamado no VerdanDesk com o contexto já enriquecido pela IA."""
    vd_cfg = config["verdandesk"]

    body_fields = {**alert, "ai_summary": ai_result["ai_summary"]}
    title = vd_cfg["ticket_template"]["title"].format(**alert)
    body = vd_cfg["ticket_template"]["body"].format(**body_fields)

    payload = {
        "queue": vd_cfg["default_queue"],
        "title": title,
        "body": body,
    }

    headers = {"Authorization": f"Bearer {vd_cfg['api_token']}"}
    resp = requests.post(f"{vd_cfg['api_base_url']}/tickets", json=payload, headers=headers, timeout=30)
    resp.raise_for_status()

    ticket_id = resp.json().get("id", "N/A")
    log.info("Ticket criado no VerdanDesk: %s", ticket_id)
    return ticket_id


def log_to_notion(config: dict, alert: dict, ai_result: dict, ticket_id: str | None) -> None:
    """Registra o alerta, o ticket e a análise da IA na base do Notion."""
    notion_cfg = config["notion"]
    props = notion_cfg["properties_map"]

    headers = {
        "Authorization": f"Bearer {notion_cfg['api_token']}",
        "Notion-Version": "2022-06-28",
        "Content-Type": "application/json",
    }

    payload = {
        "parent": {"database_id": notion_cfg["database_id"]},
        "properties": {
            props["title"]: {"title": [{"text": {"content": f"{alert['trigger']} - {alert['host']}"}}]},
            props["status"]: {"select": {"name": "Aberto" if ticket_id else "Monitorado"}},
            props["severity"]: {"select": {"name": alert["severity"]}},
            props["host"]: {"rich_text": [{"text": {"content": alert["host"]}}]},
            props["source"]: {"select": {"name": alert["source"]}},
            props["ticket_id"]: {"rich_text": [{"text": {"content": str(ticket_id or "-")}}]},
            props["created_at"]: {"date": {"start": datetime.utcnow().isoformat()}},
            props["ai_summary"]: {"rich_text": [{"text": {"content": ai_result["ai_summary"]}}]},
            props["ai_recommendation"]: {"rich_text": [{"text": {"content": ai_result["ai_first_action"]}}]},
        },
    }

    resp = requests.post("https://api.notion.com/v1/pages", json=payload, headers=headers, timeout=30)
    resp.raise_for_status()
    log.info("Alerta registrado no Notion com sucesso.")


def process_alert(config: dict, alert: dict) -> None:
    """Orquestra o fluxo completo para um único alerta recebido."""
    log.info("Novo alerta recebido: %s | host=%s | severidade=%s", alert["trigger"], alert["host"], alert["severity"])

    rule = find_rule(config, alert["trigger"])
    if rule is None:
        log.warning("Nenhuma regra encontrada para o trigger '%s'. Ignorando.", alert["trigger"])
        return

    ai_result = ask_ai(config, alert)

    ticket_id = None
    if rule.get("auto_create_ticket"):
        ticket_id = create_verdandesk_ticket(config, alert, ai_result)

    log_to_notion(config, alert, ai_result, ticket_id)


if __name__ == "__main__":
    cfg = load_config()

    # Exemplo de payload que chegaria via webhook do Zabbix/Grafana
    example_alert = {
        "source": "zabbix",
        "host": "nobreak-datacenter-01",
        "trigger": "nobreak_temperature",
        "severity": "critical",
        "description": "Temperatura do nobreak atingiu 45°C, acima do limite de 40°C.",
        "timestamp": datetime.utcnow().isoformat(),
    }

    process_alert(cfg, example_alert)
