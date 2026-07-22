# ✈️ Travel AI



An AI-powered engine for generating custom travel plan suggestions based on traveler preferences. The system automates the full cycle: from customer registration to the generation and human validation of the final itinerary.

Loom Demo: https://www.loom.com/share/663a4257940d4c58852c7f4ce8705e80

---

## 📋 Overview

Travel AI integrates two main processes:

1. **User profile registration** in Airtable, based on data entered by a new traveler.
2. **Proposal generation automation**, triggered when a status change is detected from `Pending` to `Processed by AI`. This flow deploys two intelligent agents:
   - **Agent 1:** validates airport IATA codes against the Airtable database.
   - **Agent 2:** performs external Google searches (and queries the flights API) to build the final travel proposal.

The result goes through a **Human in the loop** scheme: it is validated in Slack before being sent to the client via Gmail.

---

## 🏗️ Architecture Layers

| Component | Technology | Function |
|---|---|---|
| **Orchestrator** | n8n | Manages the profile creation flows and triggers proposal generation via triggers linked to the database. |
| **Data** | Airtable | Stores the traveler records and the airport tables queried by the AI agents. |
| **LLMs** | OpenAI (GPT-4o-mini / GPT-5) | Models operating as dual agents: one validates IATA codes, the other retrieves up-to-date destination data to suggest travel plans. |
| **Output** | Gmail and Slack | Result is validated via HITL in Slack before being sent to the client via Gmail. |



## 🗄️ Airtable data structure

### Table 1: Customers
Records people interested in traveling.

| Field | Type |
|---|---|
| Name *(primary)* | Customer identifier |
| Email | Single line text |
| Phone | Single line text |
| Travel preferences | Long text |
| Status | Single select: `Pending` → `Processed by AI` → `Approved by Human` / `Rejected` |
| Requests | Linked record to the *Requests* table |

### Table 2: Requests
Converts the customer's intent into a structured travel request.

| Field | Type |
|---|---|
| Request ID *(primary)* | Numeric |
| Origin | Single line text |
| Destination | Single line text |
| Dates | Single line text |
| Generated itinerary | Long text (final result) |
| Customers | Linked record to the *Customers* table |
| Travel preferences *(from Customers)* | Automatic lookup from the linked customer |

### Table 3: Airports
Reference catalog (non-transactional), used by the first AI agent to validate the airport before continuing the flow.

| Field |
|---|
| Airport name *(primary)* |
| IATA code |
| City |
| Country |

🔗 **Executive dashboard (Airtable):** [View base](https://airtable.com/appCPOqfBGyLWCYzB/shralv6Ies1xcHOTm)

---

## 🔒 Flow security and resilience

### Data governance
- **Data scope:** only name, email, phone, and a brief description of travel requirements are requested. No sensitive information is collected (credit cards, passwords, etc.).
- **Processing and retention:** n8n transforms the data before sending it to third-party APIs (OpenAI), adapting it to what is requested by the system prompts.
- **Segmentation in Airtable:** the "Travel Agency" base is segmented into Customers, Requests, and Airports tables. The AI agents only access the necessary fields, minimizing the exposure surface in the event of a credential breach.

### Error Handling
- **Error Trigger** nodes were configured at the critical stages of the flow (OpenAI API calls and external search queries).
- In the event of any failure (timeout, format error, service unavailability), the flow does not stop abruptly: an automatic notification is sent to the Slack channel **`error-handling`**.

---

## 🤖 Model and resource optimization

| Model / Method | Assigned task | Justification | Cost impact |
|---|---|---|---|
| **GPT-5** | Travel plan creation agent | Used together with Google search to design the itinerary, integrating flight information. Its autonomous reasoning capability makes it ideal for complex tasks (searching and analyzing information). | More expensive, but justified by the complexity of the task. |
| **GPT-4o-mini** | IATA validation and Airtable tool usage | A simpler model, ideal for basic tasks such as summarizing text or extracting data. | ~90% cheaper than the GPT-5 series (US $0.10 per million tokens vs. up to US $5 per million). |
| **Aviation Stack API** | Extracting airlines, flight numbers, and validating origin-destination routes | Needed to query real flight information and send it to the plan-generating agent. | Free in its trial stage (limited to 100 requests). |

---

## 🛠️ Tech stack

- [n8n](https://n8n.io/) — Workflow orchestration
- [Airtable](https://airtable.com/) — Database / CRM
- [OpenAI API](https://platform.openai.com/) — GPT-4o-mini and GPT-5 models
- [Aviation Stack API](https://aviationstack.com/) — Flight data
- Slack and Gmail — Human validation and customer notification

---



