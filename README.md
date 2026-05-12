# DeFi Compliance KYC Agent

> 傳統銀行進入 DeFi 業務的合規斷點解決方案  
> 一個 Ethereum 地址輸入 → 30 秒內產出符合監管格式的盡職調查 PDF 報告

[![Python](https://img.shields.io/badge/Python-3.10+-blue.svg)](https://python.org)
[![LangGraph](https://img.shields.io/badge/LangGraph-0.2+-green.svg)](https://github.com/langchain-ai/langgraph)
[![License](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

---

## 為什麼做這個

傳統銀行拿到虛擬資產牌照之後，第一個爆掉的不是技術，是合規流程。

現有工具（Chainalysis、Elliptic）能告訴你一個錢包地址的風險分數，卻無法產出監管機關在檢查時要看的那份文件——審計發現這個空白，銀行要自己扛。

這個工具專門填補這個斷點：**從鏈上地址到符合 MAS、FSC 格式的盡職調查報告，自動化產出，不動現有系統。**

切入點來自在渣打做合規與 Atomic Settlement 研究的實戰經驗——知道那份文件裡每一行是誰在看、看什麼。

---

## 架構

```
START
  │
  ▼
AddressValidator          ← 格式驗證 / checksum / chain 識別
  │
  ├─ invalid ──► ErrorReport ──► END
  │
  ▼
DataCollector             ← Etherscan API v2 + 3x retry + audit trail
  │
  ├─ API fail ─► ErrorReport ──► END
  │
  ├─ empty wallet ──► RiskAssessor [NewWallet]
  │                       └─ 套用 New Wallet rule
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

### State 寫入權責矩陣

| 欄位 | AddressValidator | DataCollector | RiskAssessor | ReportGenerator |
|------|:---:|:---:|:---:|:---:|
| `validation_result` | ✍️ | 👁️ | 👁️ | 👁️ |
| `wallet_data` | ❌ | ✍️ | 👁️ | 👁️ |
| `risk_assessment` | ❌ | ❌ | ✍️ | 👁️ |
| `narrative` | ❌ | ❌ | ❌ | ✍️ |
| `final_report` | ❌ | ❌ | ❌ | ✍️ |
| `execution_logs` | ✍️ append | ✍️ append | ✍️ append | ✍️ append |
| `raw_api_responses` | ❌ | ✍️ append | ❌ | ✍️ append |
| `cost_tracking` | ❌ | ✍️ append | ❌ | ✍️ append |

---

## 功能

### 風險規則引擎
| 規則 | 嚴重程度 | 分數 | KYC Tier |
|------|---------|------|----------|
| OFAC SDN 直接命中 | CRITICAL | 100 | Tier 4 - Prohibited |
| Tornado Cash 直接互動 | HIGH | 90 | Tier 4 - Prohibited |
| OFAC 1-hop 暴露 | HIGH | 85 | Tier 3 - Enhanced DD |
| 新錢包（< 5 筆交易） | LOW | 10 | Tier 1 - Simplified DD |

### KYC Tier 自動對應（機械決定，非 LLM）
```
score ≥ 90 → Tier 4 - Prohibited / Escalate
score ≥ 80 → Tier 3 - Enhanced Due Diligence
score ≥ 40 → Tier 2 - Standard Due Diligence
score < 40 → Tier 1 - Simplified Due Diligence
```

### 審計亮點
- **Audit Trail**：每次 API 呼叫完整記錄，body > 5MB 自動轉存 SHA-256 checksum
- **Cost Tracking**：每個 Node 記錄資源消耗，最終輸出 ROI 對比人工審查成本
- **Hallucination Check**：ReportGenerator 的 findings 強制對應到 rule_id，防止 LLM 憑空捏造

### PDF 報告格式
- 第 1 頁：封面 + Report ID + 風險等級色塊 + 執行摘要
- 第 2 頁：風險矩陣（每條規則 TRIGGERED/CLEAR 狀態）
- 第 3 頁：Evidence Appendix（flagged transactions + tx hash）
- 第 4 頁：分析師簽核欄位 + 免責聲明

---

## 快速開始

### 本機安裝

```bash
git clone https://github.com/YOUR_USERNAME/defi-kyc-agent.git
cd defi-kyc-agent

python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

pip install -r requirements.txt

cp .env.example .env
# 編輯 .env，填入 ETHERSCAN_API_KEY
```

### 申請 Etherscan API Key（免費）
1. 前往 https://etherscan.io/register 註冊
2. 登入後到 https://etherscan.io/myapikey
3. 點 "Add" 建立 Key，貼入 `.env`

### CLI 執行

```bash
# 高風險地址（Tornado Cash）
python main.py 0x722122dF12D4e14e13Ac3b6895a86e84145b6967

# 低風險地址
python main.py 0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045

# 無效格式測試
python main.py 0xINVALID
```

### Google Colab（不需本機安裝）

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/YOUR_USERNAME/defi-kyc-agent/blob/main/notebooks/DeFi_KYC_Agent_Demo.ipynb)

---

## 專案結構

```
defi-kyc-agent/
├── agents/
│   ├── __init__.py
│   ├── state.py          ← Pydantic State Schema（AgentState + 所有 sub-models）
│   ├── nodes.py          ← 5 個 Node 純函數（嚴格寫入權責）
│   └── workflow.py       ← LangGraph StateGraph + conditional edges
├── reports/
│   ├── __init__.py
│   ├── pdf_generator.py  ← WeasyPrint + Jinja2 PDF 產生器
│   └── templates/
│       └── report_template.html
├── config/
│   ├── risk_rules.yaml   ← 風險規則（可直接修改，不需改程式碼）
│   └── pricing.yaml      ← API 單位成本 + 人工對比成本
├── notebooks/
│   └── DeFi_KYC_Agent_Demo.ipynb
├── main.py               ← CLI 入口
├── requirements.txt
├── .env.example
└── README.md
```

---

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

## 設計決策

### 為什麼選 LangGraph 而不是 CrewAI？
LangGraph 的 State Machine 模型對「合規流程」這種有明確步驟的任務表達力更強，conditional edges 直接對應業務邏輯（valid/invalid、empty/has data），也更容易畫出架構圖給面試官或合規長看。

### 為什麼 KYC Tier 由規則機械決定，不用 LLM？
合規判斷需要可追溯、可審計。LLM 只負責把結構化事實「翻譯」成自然語言敘述，不能改變風險等級。這是「LLM 負責翻譯、代碼負責判斷、資料負責證明」三位一體原則。

### 為什麼不用數字風險分數（0-100）對外展示？
數字分數在沒有監管依據的情況下，反而會降低可信度。對外展示使用 HIGH/MEDIUM/LOW 三級，分數只用於內部規則路由。

---

## Roadmap

- [ ] Week 1：CLI + 風險規則引擎 ✅
- [ ] Week 2：LangGraph Multi-Agent ✅
- [ ] Week 3：WeasyPrint PDF 報告 ✅
- [ ] Week 4：Streamlit 前端 + Loom Demo
- [ ] Future：多鏈支援（BSC、Solana）
- [ ] Future：FATF Travel Rule 報告格式
- [ ] Future：OFAC API 即時串接（非靜態清單）
- [ ] Future：LLM narrative（Claude API 整合）

---

## 免責聲明

本工具為研究與展示用途，不構成法律或合規建議。所有報告結果須經具資質之合規人員審核後方可用於正式決策。

---

## 作者背景

Standard Chartered 合規實戰 + Atomic Settlement 研究 + PM 背景。  
這個工具的切入點來自親身經歷銀行 KYC 流程的斷點，以及對傳統銀行內部審計邏輯的深度理解。
