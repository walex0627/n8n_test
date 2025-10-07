# Gmail → n8n → Google Sheets Automation

**Automate the classification of banking emails into spreadsheets**

---

## 🎯 Objective

Automatically capture Gmail emails related to **banking transactions**, classify them into two types:

* **Transactions** → emails with a detected amount
* **Notifications** → emails without an amount

and record each in separate tabs within a **Google Sheets** file.

---

## 🧩 Workflow Structure

```
When clicking → Gmail (Get Many)
↓
Convert emails to JSON
↓
Convert JSON to data set
↓
Classify emails (Function → detects amounts, bank, dates, etc.)
├─ Only transactions (Function → filter with amount_cop > 0)
│    └─ Google Sheets (Append → transactions)
└─ Only notifications (Function → filter without amount)
└─ Google Sheets (Append → notifications)
```

---

## ⚙️ Step-by-Step

### 1️⃣ Gmail → Get Many

**Resource:** Message
**Operation:** Get Many
**Return All:** ✅
**Search (Expression):**

```js
{{
  "(" +
  [
    '"movimiento"','"movimientos"',
    '"extracto"','"estado de cuenta"',
    '"transferencia"','"consignación"','"consignacion"',
    '"abono"','"retiro"','"depósito"','"deposito"',
    '"comprobante"','"soporte de pago"',
    '"pago recibido"','"pago efectuado"',
    '"débitos"','"debitos"','"créditos"','"creditos"',
    '"nómina"','"nomina"','"ACH"','"PSE"'
  ].join(" OR ")
  + " OR from:(@bancolombia.com.co OR @davivienda.com OR @bbva.com OR @nequi.co OR @daviplata.com OR @nubank.com)"
  + " OR subject:(Bancolombia OR Davivienda OR BBVA OR Nequi OR Daviplata OR Nu OR Wompi OR PayU OR \"Mercado Pago\")"
  + ") "
  + "after:" + $now.startOf('month').toFormat('yyyy/MM/dd')
  + " before:" + $now.endOf('month').plus({ days: 1 }).toFormat('yyyy/MM/dd')
}}
```

🧠 **This only retrieves emails from the current month**, so you don’t need to update the date every month.

---

### 2️⃣ Function → Convert to JSON

Converts Gmail output into a clean format with all relevant fields (id, from, subject, snippet, etc.):

```js
const out = items.map(item => {
  const j = item.json || {};
  const mimeType = j.payload?.mimeType ?? j.mimeType ?? null;
  const labelsArr = Array.isArray(j.labels) ? j.labels.map(l => ({ id: l.id ?? l, name: l.name ?? l })) : [];
  return {
    id: j.id ?? null,
    threadId: j.threadId ?? null,
    snippet: j.snippet ?? null,
    payload: { mimeType },
    sizeEstimate: j.sizeEstimate ?? null,
    historyId: j.historyId ?? null,
    internalDate: j.internalDate ?? null,
    labels: labelsArr,
    From: j.From ?? j.headers?.From ?? null,
    To: j.To ?? j.headers?.To ?? null,
    Subject: j.Subject ?? j.headers?.Subject ?? null
  };
});
return [{ json: { emails: out } }];
```

---

### 3️⃣ Item Lists → Split Out Items

**Operation:** Split Out Items
**Property Name:** `emails`

> Converts the `{ emails: [...] }` array into individual items.

---

### 4️⃣ Function → Classification and Parsing

Analyzes each email to detect amounts, bank, merchant, and dates.

```js
const TZ_OFFSET_HOURS = -5;

function parseCopAmount(s) {
  const m = s?.match(/COP\s*([\d\.,]+)/i);
  if (!m) return null;
  return parseFloat(m[1].replace(/\./g, "").replace(",", "."));
}
function detectBank(from, subject) {
  const txt = `${from} ${subject}`.toLowerCase();
  if (txt.includes("bancolombia")) return "Bancolombia";
  if (txt.includes("davivienda")) return "Davivienda";
  if (txt.includes("bbva")) return "BBVA";
  if (txt.includes("nu.com.co")) return "Nu";
  if (txt.includes("daviplata")) return "Daviplata";
  if (txt.includes("nequi")) return "Nequi";
  return null;
}
function parseDate(snippet) {
  const m = snippet.match(/el\s+(\d{2})\/(\d{2})\/(\d{4})\s+a las\s+(\d{2}):(\d{2})/i);
  if (!m) return null;
  const [_, d, M, y, H, m2] = m;
  const utc = new Date(Date.UTC(+y, +M - 1, +d, +H - TZ_OFFSET_HOURS, +m2));
  return {
    tx_date: `${y}-${M}-${d}`,
    tx_time: `${H}:${m2}`,
    tx_iso_utc: utc.toISOString()
  };
}

return items.map(i => {
  const j = i.json;
  const snippet = j.snippet ?? "";
  const subject = j.Subject ?? "";
  const from = j.From ?? "";
  const bank = detectBank(from, subject);
  const amount_cop = parseCopAmount(snippet);
  const date = parseDate(snippet);
  return {
    json: {
      ...j,
      bank,
      amount_cop,
      merchant: snippet.match(/en\s+(.+?)\s+con tu/i)?.[1] ?? null,
      currency: amount_cop ? "COP" : null,
      tx_date: date?.tx_date ?? null,
      tx_time: date?.tx_time ?? null,
      tx_iso_utc: date?.tx_iso_utc ?? null
    }
  };
});
```

---

### 5️⃣ Function → Classify type (`transaction` or `notification`)

```js
return items.map(i => {
  const v = Number(i.json.amount_cop);
  const hasAmount = Number.isFinite(v) && v > 0;
  return { json: { ...i.json, type: hasAmount ? "transaction" : "notification" } };
});
```

---

### 6️⃣ Function → Filter by Type

#### a) Only transactions

```js
return items.filter(i => {
  const v = Number(i.json?.amount_cop);
  return Number.isFinite(v) && v > 0;
});
```

#### b) Only notifications

```js
return items.filter(i => {
  const v = Number(i.json?.amount_cop);
  return !(Number.isFinite(v) && v > 0);
});
```

---

### 7️⃣ Google Sheets → Append

#### ⚙️ Basic Configuration

* **Credential:** your Google account
* **Resource:** Sheet Within Document
* **Operation:** Append Row
* **Document:** By URL → `https://docs.google.com/spreadsheets/d/XXXXXXX/edit`
* **Sheet:** By Name → `transactions` or `notifications`
* **Mapping Column Mode:** Map Automatically ✅

#### 🧾 Headers in Google Sheets

In both tabs (`transactions`, `notifications`):

```
id, threadId, bank, subject, from, to, snippet, merchant, amount_cop, currency, card_last4, tx_date, tx_time, tx_iso_utc, internalDate, labels, mimeType, sizeEstimate, historyId, type
```

---

### (Optional) 8️⃣ Convert `internalDate` to Readable Date

Before the Append node, add this Function node:

```js
return items.map(i => {
  const ts = Number(i.json.internalDate);
  const local = new Date(ts + (-5 * 60 * 60 * 1000)); // Bogotá
  return { json: { ...i.json, received_at: local.toISOString() } };
});
```

Then add `received_at` to your headers.

---

## 🧠 Tips

* **403 Forbidden:** if you see this error when connecting Sheets, switch “By Name” to “By URL” and paste the file’s URL.
* **Duplicates:** you can insert a `Google Sheets → Lookup (Key=id)` node before the Append.
* **Testing:** run the workflow manually each month, or add a **Trigger** (monthly cron).
* **Logs:** use the right panel to verify which emails were classified as transactions.

---

## 📊 Result

* Emails with amounts (“You purchased COP…”) are stored in `transactions`.
* Informational emails without amounts (“Your account has been opened…”) go to `notifications`.
* Each row keeps sender data, subject, merchant, amount, and the exact date and time received.

---

## 🧱 Requirements

* Gmail API enabled in n8n
* Google Sheets credential with Editor access
* File `n8n_test` with two tabs: `transactions` and `notifications`

---

✅ Author

Workflow designed by Walter Gomez
Full banking classification automation in n8n ✨

📩 Contact me for more information: walteralex0627@gmail.com

---