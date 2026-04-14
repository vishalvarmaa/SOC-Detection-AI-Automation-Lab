# SOC Detection & AI Automation Lab

A fully functional Security Operations Centre (SOC) home lab built from scratch — simulating real-world threat detection, automated incident response and AI-powered investigation via a Telegram bot.

Built on Kali Linux using Wazuh SIEM, n8n SOAR automation, multi-source threat intelligence enrichment and a Groq-powered AI analyst that answers security questions in plain English.

---

## Workflow Overview

![n8n Workflow](screenshots/workflow.png)

---

## Architecture

```
Wazuh SIEM (Kali Linux)
        │
        ▼
n8n Webhook receives alert
        │
        ▼
Extract Fields
(rule_id, severity, source_ip, agent, MITRE ATT&CK)
        │
        ▼
IF Severity >= 5 ──────────────────► Fast Path DB Save (Low severity)
        │
        ▼
IF External IP ────────────────────► Endpoint Alert Path (Internal IP)
        │
        ▼
┌───────────────────────────────────────┐
│         Threat Intelligence           │
│  VirusTotal → AbuseIPDB → OTX         │
│  Intermediate Score Calculation       │
│  IF Shodan Gate (score >= 50)         │
│  Shodan → IP Geolocation              │
└───────────────────────────────────────┘
        │
        ▼
Build Alert Data
(risk_score, risk_level, MITRE mapping, geo data)
        │
        ▼
Fix Risk Level → Switch Risk Level
        │
   ┌────┴─────────────────────┐
CRITICAL   HIGH   MEDIUM   LOW
   │
   ▼
Save to PostgreSQL → Telegram Alert

─────────────────────────────────────

Telegram Bot (AI SOC Agent)
        │
        ▼
Normalize Message
        │
        ▼
Switch Message Type
   │           │            │
Help        Greeting    AI SQL Generator
                              │
                         Groq LLM
                         (llama-3.3-70b)
                              │
                         Execute SQL Query
                              │
                    Aggregate → Deduplication
                              │
                         AI Summarizer
                         (Groq LLM)
                              │
                         Send AI Reply
```

---

## Features

### Automated Alert Pipeline
- Wazuh SIEM sends alerts via webhook to n8n on every triggered rule
- Fields extracted — rule ID, severity level, source IP, agent name, MITRE ATT&CK tactic and technique
- Severity gate filters alerts below level 5 — low severity alerts fast-pathed to database only
- External IP check separates network threats from endpoint threats automatically

### 4-Source Threat Intelligence Enrichment
- **VirusTotal** — malicious and suspicious engine count per IP
- **AbuseIPDB** — abuse confidence score and total reports
- **AlienVault OTX** — threat pulse count
- **Shodan** — open ports, CVEs and organisation (conditional — only when intermediate score >= 50)
- **IP Geolocation** — country, city and ISP via ip-api.com

### Risk Scoring Engine
```
Risk Score = (VT Malicious × 4) + (VT Suspicious × 1)
           + (AbuseIPDB Score × 0.4) + (OTX Pulses × 2)
           + (Shodan CVEs × 5) + (Wazuh Severity × 4)
           
Maximum capped at 100

CRITICAL  →  score >= 75  →  IMMEDIATE_RESPONSE
HIGH      →  score >= 50  →  INVESTIGATE
MEDIUM    →  score >= 25  →  MONITOR
LOW       →  score < 25   →  LOG_ONLY
```

### PostgreSQL Alert Storage
- All alerts saved to structured `alerts` table
- Fields stored — IP, alert name, risk level, risk score, MITRE ATT&CK, country, agent name, rule ID, Shodan ports, Shodan CVEs, VirusTotal score, AbuseIPDB score
- Separate save nodes per severity level — CRITICAL, HIGH, MEDIUM, LOW

### Real-Time Telegram Notifications
- CRITICAL and HIGH alerts trigger instant Telegram messages
- Messages include full threat intel summary — IP, location, ISP, MITRE mapping, all enrichment scores
- Endpoint alerts (internal IP) handled separately with host investigation messaging
- Sarcastic alert messages with context-aware humour

### AI SOC Agent — Telegram Bot
- Natural language to PostgreSQL query conversion using Groq LLM (llama-3.3-70b-versatile)
- Analysts can ask questions in plain English — no SQL knowledge required
- Query classification — DATABASE, KNOWLEDGE or GIBBERISH
- Deduplication and aggregation of results before AI summarization
- AI Summarizer responds with full threat context, severity-aware tone and remediation steps
- Handles help menu, greetings, knowledge questions, database queries and error states

### Demo Mode
- Built-in demo trigger simulates SSH brute force attack
- Pre-configured payload — rule 5710, severity 12, MITRE T1110 Credential Access
- Test the full pipeline without a real Wazuh alert

---

## Tech Stack

| Component | Technology |
|---|---|
| SIEM | Wazuh |
| Automation | n8n SOAR |
| AI Model | Groq — llama-3.3-70b-versatile |
| Database | PostgreSQL |
| Threat Intel | VirusTotal, AbuseIPDB, AlienVault OTX, Shodan |
| Geolocation | ip-api.com |
| Notifications | Telegram Bot API |
| OS | Kali Linux (VMware) |
| Agent | Windows 11 VM |

---

## Workflow Nodes

| Node | Purpose |
|---|---|
| Wazuh Webhook | Receives alerts from Wazuh via HTTP POST |
| Extract Fields | Parses rule ID, severity, IP, agent, MITRE data |
| IF Severity >= 5 | Filters low severity alerts |
| IF External IP | Separates external and internal/endpoint threats |
| VirusTotal | IP reputation check |
| AbuseIPDB | Abuse confidence score |
| OTX AlienVault | Threat pulse count |
| Intermediate Score | Calculates pre-Shodan score to gate API call |
| IF Shodan Gate | Only calls Shodan if score >= 50 |
| Shodan | Open ports and CVE lookup |
| IP Geolocation | Country, city, ISP lookup |
| Build Alert Data | Assembles all enrichment into unified alert object |
| Fix Risk Level | Recalculates and fixes risk level from final score |
| Switch Risk Level | Routes alert to CRITICAL / HIGH / MEDIUM / LOW path |
| Save CRITICAL/HIGH/MEDIUM/LOW | Inserts alert into PostgreSQL |
| Fast Path DB Save | Direct DB insert for low severity alerts |
| Telegram CRITICAL/HIGH | Sends formatted alert message to Telegram |
| Telegram Trigger | Listens for incoming Telegram messages |
| Normalize Message | Extracts text, chat ID, sender name |
| Switch Message Type | Routes to Help, Greeting or AI path |
| Send Help Menu | Returns command list with sarcastic intro |
| Send Greeting | Sarcastic greeting response |
| AI SQL Generator | Converts natural language to SQL using Groq |
| Execute SQL Query | Runs generated SQL against PostgreSQL |
| Aggregate | Combines all result rows |
| Deduplication Code | Groups and deduplicates alerts by IP + name + risk |
| AI Summarizer | Generates human-readable response from query results |
| Send AI Reply | Sends final AI response to Telegram |
| Send SQL Error | Handles SQL execution errors gracefully |
| DEMO — Network Attack | Manual trigger for demo mode |
| Demo Network Payload | SSH brute force demo data |

---

## Setup

### Prerequisites
```
- n8n (self-hosted)
- Wazuh SIEM
- PostgreSQL
- Telegram Bot (via BotFather)
- API keys — VirusTotal, AbuseIPDB, AlienVault OTX, Shodan, Groq
```

### PostgreSQL Table
```sql
CREATE TABLE alerts (
  id          SERIAL PRIMARY KEY,
  ip          TEXT,
  alert_name  TEXT,
  risk_level  TEXT,
  risk_score  INTEGER,
  mitre_attack TEXT,
  country     TEXT,
  ai_explanation TEXT,
  created_at  TIMESTAMP DEFAULT NOW(),
  agent_name  TEXT,
  rule_id     TEXT,
  shodan_vulns TEXT,
  shodan_ports TEXT,
  vt_score    INTEGER,
  abuse_score INTEGER
);
```

### Configuration
```
1. Import Final no api.json into n8n
2. Add your credentials in n8n:
   - Telegram Bot API token
   - PostgreSQL connection
   - Groq API key
3. Replace all YOUR_API_KEY_HERE placeholders
4. Replace YOUR_Chat_ID with your Telegram chat ID
5. Configure Wazuh to send webhooks to your n8n URL
6. Activate the workflow
```

### Wazuh Webhook Configuration
```xml
<!-- Add to /var/ossec/etc/ossec.conf -->
<integration>
  <name>custom-webhook</name>
  <hook_url>http://YOUR_N8N_IP:5678/webhook/YOUR_WEBHOOK_PATH</hook_url>
  <level>5</level>
  <alert_format>json</alert_format>
</integration>
```

---

## Telegram Bot Commands

```
menu / help      →  Show command list
show critical    →  Latest critical alerts
show high        →  Latest high alerts
panic            →  Show all critical and high alerts
who is attacking →  Top attacking IPs
full report      →  Alert summary by severity
am i hacked      →  Check for critical threats
alerts today     →  Last 24 hours
top countries    →  Where attacks are coming from
summary / stats  →  Total alert counts
show last alert  →  Most recent alert
```

Any plain English question works — the AI converts it to SQL automatically.

---

## Screenshots

| | |
|---|---|
| Workflow Overview | `screenshots/workflow.png` |
| Telegram CRITICAL Alert | `screenshots/telegram-critical.png` |
| Telegram HIGH Alert | `screenshots/telegram-high.png` |
| Telegram Bot Response | `screenshots/telegram-bot-reply.png` |
| Wazuh Dashboard | `screenshots/wazuh-dashboard.png` |
| PostgreSQL Alerts Table | `screenshots/postgres-alerts.png` |

---

## Security Note

All API keys and credentials have been removed from this workflow.
Replace all `YOUR_API_KEY_HERE` and `YOUR_Chat_ID` placeholders with your own credentials before importing.
Never commit real API keys to a public repository.

---

## Author

**Vishal Varma**
SOC Analyst | Security Operations | Threat Detection
[LinkedIn](https://linkedin.com/in/vishalvarma-soc) · [GitHub](https://github.com/YOUR_USERNAME)
