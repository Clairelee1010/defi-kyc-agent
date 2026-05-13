# DeFi Compliance KYC Agent

> Solving the compliance gap for traditional banks entering DeFi
> One Ethereum address → regulatory-grade due diligence PDF report in under 30 seconds
> 傳統銀行進入 DeFi 業務的合規斷點解決方案  
> 一個 Ethereum 地址輸入 → 30 秒內產出符合監管格式的盡職調查 PDF 報告

[![Python](https://img.shields.io/badge/Python-3.10+-blue.svg)](https://python.org)
[![LangGraph](https://img.shields.io/badge/LangGraph-0.2+-green.svg)](https://github.com/langchain-ai/langgraph)
[![License](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

---
## Why This Exists
## 為什麼做這個
When a traditional bank obtains a virtual asset licence, the first thing that breaks is not the technology — it is the compliance workflow.

Existing tools (Chainalysis, Elliptic) can tell you a wallet's risk score. What they cannot do is produce the document that regulators actually look at during an inspection. When auditors find that gap, the bank is on the hook.

This tool fills that gap: **from an on-chain address to a MAS/FSC-formatted due diligence report, fully automated, without touching existing systems.**

The insight comes from first-hand experience running compliance at Standard Chartered and researching Atomic Settlement — knowing exactly who reads each line of that document and what they are looking for.
傳統銀行拿到虛擬資產牌照之後，第一個爆掉的不是技術，是合規流程。

現有工具（Chainalysis、Elliptic）能告訴你一個錢包地址的風險分數，卻無法產出監管機關在檢查時要看的那份文件——審計發現這個空白，銀行要自己扛。

這個工具專門填補這個斷點：**從鏈上地址到符合 MAS、FSC 格式的盡職調查報告，自動化產出，不動現有系統。**

切入點來自在渣打做合規與 Atomic Settlement 研究的實戰經驗——知道那份文件裡每一行是誰在看、看什麼。

---
## Architecture
## 架構

```
START
  │
  ▼
AddressValidator          ← format / checksum / chain detection    (格式驗證 / checksum / chain 識別)
  │
  ├─ invalid ──► ErrorReport ──► END
  │
  ▼
DataCollector             ← Etherscan API v2 + 3x retry + audit trail  (Etherscan API v2 + 3x retry + audit trail)
  │
  ├─ API fail ─► ErrorReport ──► END
  │
  ├─ empty wallet ──► RiskAssessor [NewWallet]
  │                       └─ applies (套用) New Wallet rule
  │
  └─ has data ──► RiskAssessor [Full]
                      └─ OFAC SDN + Tornado Cash + 1-hop
  │
  ▼
ReportGenerator           ← narrative + cost summary + PDF hook
  │
  ▼
END
```

### State Write-Responsibility Matrix
### State 寫入權責矩陣

| Field | AddressValidator | DataCollector | RiskAssessor | ReportGenerator |
|-------|:---:|:---:|:---:|:---:|
| `validation_result` | write | read | read | read |
| `wallet_data` | — | write | read | read |
| `risk_assessment` | — | — | write | read |
| `narrative` | — | — | — | write |
| `final_report` | — | — | — | write |
| `execution_logs` | append | append | append | append |
| `raw_api_responses` | — | append | — | append |
| `cost_tracking` | — | append | — | append |

---

## Features
## 功能
### Risk Rule Engine
### 風險規則引擎
| Rule | Severity | Score | KYC Tier |
|------|---------|-------|----------|
| OFAC SDN direct match | CRITICAL | 100 | Tier 4 — Prohibited |
| Tornado Cash direct interaction | HIGH | 90 | Tier 4 — Prohibited |
| OFAC 1-hop exposure | HIGH | 85 | Tier 3 — Enhanced DD |
| New wallet (< 5 transactions) | LOW | 10 | Tier 1 — Simplified DD |


### KYC Tier Mapping (rule-based, not LLM)
### KYC Tier 自動對應（機械決定，非 LLM）
```
score ≥ 90 → Tier 4 - Prohibited / Escalate
score ≥ 80 → Tier 3 - Enhanced Due Diligence
score ≥ 40 → Tier 2 - Standard Due Diligence
score < 40 → Tier 1 - Simplified Due Diligence
```


### Audit Highlights
### 審計亮點
- **Audit Trail** — every API call is fully logged; responses over 5 MB are stored as metadata + SHA-256 checksum
- **Cost Tracking** — each node records resource consumption; final output includes ROI comparison against manual review cost
- **Hallucination Check** — every finding in ReportGenerator is forced to map to a rule_id, preventing the LLM from fabricating conclusions
- **Audit Trail**：每次 API 呼叫完整記錄，body > 5MB 自動轉存 SHA-256 checksum
- **Cost Tracking**：每個 Node 記錄資源消耗，最終輸出 ROI 對比人工審查成本
- **Hallucination Check**：ReportGenerator 的 findings 強制對應到 rule_id，防止 LLM 憑空捏造

### PDF Report Format
### PDF 報告格式
- Page 1: Cover — Report ID, risk-level colour block, executive summary
- Page 2: Risk Matrix — TRIGGERED / CLEAR status for each rule
- Page 3: Evidence Appendix — flagged transactions with tx hashes
- Page 4: Analyst sign-off fields + disclaimer
- 第 1 頁：封面 + Report ID + 風險等級色塊 + 執行摘要
- 第 2 頁：風險矩陣（每條規則 TRIGGERED/CLEAR 狀態）
- 第 3 頁：Evidence Appendix（flagged transactions + tx hash）
- 第 4 頁：分析師簽核欄位 + 免責聲明

---
## Quick Start
## 快速開始

### Local Installation
### 本機安裝

```bash
git clone https://github.com/YOUR_USERNAME/defi-kyc-agent.git
cd defi-kyc-agent

python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

pip install -r requirements.txt

cp .env.example .env
# OpenOpen .env，填入 ETHERSCAN_API_KEY
```

### Get a Free Etherscan API Key
### 申請 Etherscan API Key（免費）
1. Register at https://etherscan.io/register
2. Go to https://etherscan.io/myapikey after login
3. Click "Add", copy the key, paste it into .env
1. 前往 https://etherscan.io/register 註冊
2. 登入後到 https://etherscan.io/myapikey
3. 點 "Add" 建立 Key，貼入 `.env`

### CLI Usage
### CLI 執行

```bash
# High-risk address (Tornado Cash)
python main.py 0x722122dF12D4e14e13Ac3b6895a86e84145b6967

# Low-risk address
python main.py 0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045

# Invalid format test
python main.py 0xINVALID
```

### Google Colab (no local setup required)
### Google Colab（不需本機安裝）

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/YOUR_USERNAME/defi-kyc-agent/blob/main/notebooks/DeFi_KYC_Agent_Demo.ipynb)

---

## Project Structure
## 專案結構

```
defi-kyc-agent/
├── agents/
│   ├── __init__.py
│   ├── state.py          ← Pydantic State Schema（AgentState + 所有 sub-models）
│   ├── nodes.py          ← 5 pure node functions (strict write-responsibility)  5 個 Node 純函數（嚴格寫入權責）
│   └── workflow.py       ← LangGraph StateGraph + conditional edges
├── reports/
│   ├── __init__.py
│   ├── pdf_generator.py  ← WeasyPrint + Jinja2 PDF generator 產生器
│   └── templates/
│       └── report_template.html
├── config/
│   ├── risk_rules.yaml   ← risk rules (edit directly, no code changes needed)  風險規則（可直接修改，不需改程式碼）
│   └── pricing.yaml      ← API unit costs + human baseline for ROI  API 單位成本 + 人工對比成本
├── notebooks/
│   └── DeFi_KYC_Agent_Demo.ipynb
├── main.py               ← CLI entry point  CLI 入口
├── requirements.txt
├── .env.example
└── README.md
```

---

## Sample Output
## 輸出範例

```json
{
  "address": "0x722122df12d4e14e13ac3b6895a86e84145b6967",
  "overall_risk_tier": "CRITICAL",
  "overall_score": 90.0,
  "suggested_kyc_tier": "Tier 4 - Prohibited / Escalate",
  "executive_summary": "Due diligence for 0x722122df... Risk: CRITICAL (score: 90/100). 1 rule(s) triggered. Action: Tier 4 - Prohibited / Escalate.",
  "key_findings": [
    "Direct interaction with 1 Tornado Cash contract(s)."
  ],
  "triggered_rules": ["TORNADO_CASH_DIRECT"],
  "flagged_transactions_count": 5,
  "cost_summary": {
    "total_agent_cost_usd": 0.0002,
    "human_baseline_cost_usd": 25.0,
    "estimated_savings_usd": 24.9998,
    "roi_ratio": 125000.0
  }
}
```

---
## Design Decisions
## 設計決策
### Why LangGraph over CrewAI?
### 為什麼選 LangGraph 而不是 CrewAI？
LangGraph's state machine model is a stronger fit for compliance workflows with well-defined, sequential steps. Conditional edges map directly to business logic (valid/invalid, empty/has data), and the graph structure produces architecture diagrams that are immediately readable by both engineers and compliance officers.
LangGraph 的 State Machine 模型對「合規流程」這種有明確步驟的任務表達力更強，conditional edges 直接對應業務邏輯（valid/invalid、empty/has data），也更容易畫出架構圖給面試官或合規長看。

### Why is KYC Tier determined by rules, not the LLM?
### 為什麼 KYC Tier 由規則機械決定，不用 LLM？
Compliance decisions must be traceable and auditable. The LLM's role is limited to translating structured facts into natural-language narrative — it cannot alter the risk tier. This reflects the principle: LLM translates, code judges, data proves.
合規判斷需要可追溯、可審計。LLM 只負責把結構化事實「翻譯」成自然語言敘述，不能改變風險等級。這是「LLM 負責翻譯、代碼負責判斷、資料負責證明」三位一體原則。

### Why not display a numeric risk score (0–100) externally?
### 為什麼不用數字風險分數（0-100）對外展示？

A numeric score without a regulatory basis erodes credibility rather than building it. External-facing outputs use HIGH / MEDIUM / LOW tiers. The numeric score is used internally only for rule routing.
數字分數在沒有監管依據的情況下，反而會降低可信度。對外展示使用 HIGH/MEDIUM/LOW 三級，分數只用於內部規則路由。

---

### Relationship to KYA (Know Your Agent)

Visa TAP, Ethereum ERC-8004, and Trulioo DAP are building identity infrastructure for AI agents. This tool addresses the corresponding bank-side problem: when a compliance team needs to review a suspicious agent address, existing tools only provide a risk score. This tool produces the due diligence document that regulators actually inspect.

---

## Roadmap

- [ ] Week 1 — CLI + risk rule engine
- [ ] Week 2 — LangGraph Multi-Agent architecture
- [ ] Week 3 — WeasyPrint PDF report
- [ ] Week 4 — Streamlit frontend + Loom demo
- [ ] Future — multi-chain support (BSC, Solana)
- [ ] Future — FATF Travel Rule report format
- [ ] Future — live OFAC API integration (replace static list)
- [ ] Future — LLM narrative via Claude API

---

## Disclaimer
## 免責聲明
This tool is for research and demonstration purposes only. It does not constitute legal or compliance advice. All report outputs must be reviewed by a qualified compliance officer before use in any formal decision-making process.
本工具為研究與展示用途，不構成法律或合規建議。所有報告結果須經具資質之合規人員審核後方可用於正式決策。

---

## About the Author
## 作者背景
Compliance experience · Atomic Settlement researcher · Product Manager.
 合規經驗 + Atomic Settlement 研究 + PM 背景。  
This tool was built from direct experience with the broken seams in bank KYC workflows, and a deep understanding of what the internal audit process actually requires.
這個工具的切入點來自親身經歷銀行 KYC 流程的斷點，以及對傳統銀行內部審計邏輯的深度理解。

