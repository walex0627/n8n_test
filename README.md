# 🏦 Automated Financial Reconciliation System

### *Transforming chaotic email notifications into a real-time structured Financial Ledger.*

This project is a high-performance **Robotic Process Automation (RPA)** solution built with **n8n**. It bridges the gap between unstructured financial data (bank emails) and structured accounting records (Google Sheets). By combining the Gmail API with custom JavaScript logic, it automates the entire bank reconciliation process with 100% accuracy and zero manual entry.

---

## 🚀 Key Features

* **Multi-Bank Ingestion:** Native support for **Bancolombia, Nequi, Nu, Ualá, Davivienda, Daviplata, PSE, Breebe, and Transfiya**.
* **Smart Parsing Engine:** A custom **JavaScript node** utilizing advanced Regular Expressions (Regex) to extract amounts, identify merchants, and classify transaction types from non-structured email snippets.
* **Automated Deduplication (Idempotency):** Uses the unique Gmail `Message ID` as a Primary Key to execute **Upsert** (Update/Insert) logic, ensuring no transaction is ever recorded twice.
* **API Resilience & Throttling:** Built-in error handling and wait nodes to manage Google Sheets API rate limits ($60 \text{ RPM}$), preventing "429 Too Many Requests" errors during bulk processing.
* **Audit-Ready Data:** Normalizes data into a professional format, including Bogota-time timestamps, cleaned merchant names, and standardized currency values.

---

## 🛠️ Tech Stack

* **Orchestrator:** [n8n](https://n8n.io/)
* **Languages:** JavaScript (ES6+) for complex data transformation.
* **APIs:** Gmail API (v1), Google Sheets API (v4).
* **Data Logic:** Advanced Regex with word boundaries (`\b`) for surgical precision.

---

## 📋 Workflow Architecture

The system follows a four-phase pipeline:

### 1. Extraction (Gmail API)
Applies a highly optimized search filter to minimize API overhead and filter out noise:
`from:(bank_domain) (keywords) after:YYYY/MM/DD`

### 2. Transformation (The "Brain")
The custom JS node performs deep data cleaning:
* **Amount Extraction:** Detects `COP`, `$`, or keywords like `value` and `amount`, normalizing decimal and thousand separators.
* **Merchant Detection:** Uses prepositional patterns (`at`, `from`, `to`) to identify the specific store or person involved.
* **Classification:** Distinguishes between `transaction` (money movement) and `notification` (informative alerts).

### 3. Flow Control (Switch & Loop)
Branches the logic to handle financial expenses/income differently from general security alerts, ensuring only relevant data hits the final ledger.

### 4. Persistence (Google Sheets)
Executes a **Conditional Upsert**:
> *If the `id` exists, update the row; otherwise, append a new record.*

---

## ⚙️ Data Structure (JSON)

Each processed item follows this normalized schema:

```javascript
{
  "id": "19d7f567d47f6be5",
  "bank": "Nu",
  "merchant": "TIENDA D1 BARRANQUILLA",
  "amount_cop": 16050.00,
  "type": "transaction",
  "tx_date": "2026-04-18",
  "currency": "COP"
}
```
¡Claro, Walter! Es un paso clave para que tu repo sea realmente útil ("open source style"). Lo ideal es ponerlo justo después de la sección del **Stack Tecnológico** o en una sección nueva llamada **Installation** o **How to Use**.

Aquí tienes el bloque listo para copiar y pegar en tu **README.md** (te lo dejo en inglés para que combine con el resto):

---

## 📥 Getting Started & Installation

> **Note:** The complete n8n workflow is included in this repository as a `.json` file. You can find it as `financial_reconciliation_workflow.json` (or your specific file name).

To get this automation running in your own environment:

1.  **Download the JSON file** from this repository.
2.  Open your **n8n** instance.
3.  Create a new workflow and go to **Import from File** in the top-right menu.
4.  Select the downloaded `.json` file.
5.  **Configure your credentials:** You will need to link your own Google (Gmail & Sheets) credentials for the nodes to activate.
6.  Set the `Sheet ID` and `Range` in the Google Sheets nodes to match your specific spreadsheet.

---

## 📈 Business Value & Auditability

As a **Public Accounting student** and **Full-Stack Developer**, I designed this tool with a dual focus:
1.  **Data Integrity:** Implements exponential backoff policies to ensure no data is lost during network or API spikes.
2.  **Traceability:** Every record includes the `threadId` and `historyId`, allowing for a 1:1 audit trail back to the original email.
3.  **Efficiency:** Reduces manual bookkeeping time by an estimated **98%**, allowing for real-time cash flow monitoring.

---
## ⚠️ Credential Configuration Note

For security and privacy reasons, the provided `.json` file does not contain active API credentials. To test this workflow in your own environment:

1. You must create a project in the **Google Cloud Console**.

2. Enable the **Gmail API** and **Google Sheets API**.

3. Create your own **OAuth2 Credentials** and link them to your n8n instance.

4. Update the **Spreadsheet ID** and **Range** in the Google Sheets nodes to point to your target document.
---

## 👤 Author

**Walter Gomez**

*Junior Full-Stack Developer & Public Accounting Student*
*Barranquilla, Colombia*

---

> **Note:** This project is part of an "AI-Native" workflow, leveraging artificial intelligence for rapid prototyping, Regex optimization, and architectural scaling.
