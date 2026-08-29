# Project Persistence Context: NFL Email & n8n Automation Engine

> **Master Persistence Document**
> This file is the single source of truth for the email templates, n8n renewal notification workflows, Google Sheets integrations, credentials, and multi-channel notification policies across development environments.
> **Daily Routine**: Run `git pull origin main` before starting work; inspect this document to maintain uniform context; commit and `git push origin main` upon any change.

---

## 1. Project Overview & Scope

This repository manages two key operational systems for **Total TV USA**:
1. **Email Marketing Campaigns**: HTML email templates for NFL season campaigns and promotional blasts (`nfl_urgent_light_email.html`, `nfl_kickoff_email.html`).
2. **Automated Expiration & Renewal Pipelines (n8n)**: Workflows running on self-hosted n8n (`https://n8n.ac4.club/`) that monitor subscription expiration dates in Google Sheets, route customers based on payment methods, dispatch notifications via Gmail, Telnyx SMS, and WhatsApp Cloud API, generate NowPayments crypto discount links, and mark statuses back in the spreadsheet.

---

## 2. Google Sheets Integration Matrix

### Master Document
* **Document Name**: `Clientes TotalTV`
* **Document ID**: `1SNRbfgomUgtac58UmIMlH8UzizBXrTDVogxJEt-z9A0`
* **Credential**: `Google Sheets account` (`googleSheetsOAuth2Api`, ID: `Pw5wN2L5UopOruaj`)

### Sheet Tabs & Column Schemas

| Tab Name | Internal Sheet ID (`gid` / `sheetId`) | Primary Audience | Key Column Details |
| :--- | :--- | :--- | :--- |
| **`Mega`** | **`51202947`** | Total TV USA (Mega OTT Panel) | See Column Schema below |
| **`DnSpace`** | **`1823373862`** | Total TV Latina / TVTotal24 | Column F is `Vence` |

### `Mega` Tab Column Mapping (0-Indexed)
* Column A (`0`): `Nombre` (First name)
* Column B (`1`): `Apellido` (Last name)
* Column C (`2`): `Teléfono` (Numeric phone string)
* Column D (`3`): `Usuario` (Unique subscription username – **used for matching in updates**)
* Column E (`4`): `Clave` (Password)
* Column F (`5`): **`Vence`** (Expiration date, formatted as `YYYY-MM-DD` or `DD/MM/YYYY`)
* Column G (`6`): `1ra compra`
* Column H (`7`): `Conns/xxx`
* Column I (`8`): `Paga por` (Payment method preference, e.g., `zelle`)
* Column J (`9`): `Email` (Customer email address)
* Column K (`10`): `Observación`
* Column L (`11`): **`Ultimo Monto`** (Subscription amount, without accent, e.g., `15` or `15.00`)
* Column M (`12`): **`PLAY`** (Status tracking flag: `" "`, `"SENT"`, `"SENT4"`)

---

## 3. n8n Workflows Reference

### 3.1. `Mega expires SOON (Vence 4 días)`
* **Workflow ID**: `F7M6sLe1lo4zUObT`
* **Target Sheet**: `Clientes TotalTV` $\rightarrow$ `Mega` (`51202947`)
* **Trigger**: Scheduled daily at 9:00 AM (`America/Caracas` timezone)
* **Date Filter**: Matches customers where `Vence` is **in exactly 4 days** (`America/Caracas`) and `PLAY != 'SENT4'`.
* **Path A (Zelle)**:
  * Condition: `Paga por` equals `zelle`.
  * Channels:
    * **Gmail** (`Send a message Zelle`): Sends renewal email with Zelle payment details and amount from `Ultimo Monto`.
  * Update: Updates row matching by `Usuario`, setting `PLAY = "SENT4"`.
* **Path B (Non-Zelle / Payment Link)**:
  * Generates NowPayments invoice via HTTP Request with 20% discount on `Ultimo Monto`.
  * Preserves customer metadata and combines invoice URL safely via JavaScript Code node.
  * **Gmail** (`Send a message Link`): Sends full email with Zelle, Cash App, and NowPayments crypto payment link ($20% off).
  * Update: Updates row matching by `Usuario`, setting `PLAY = "SENT4"`.

### 3.2. `Mega expires TODAY (Vence HOY)`
* **Workflow ID**: `943Yu3CZMD4dzRCI`
* **Target Sheet**: `Clientes TotalTV` $\rightarrow$ `Mega` (`51202947`)
* **Trigger**: Scheduled daily at 9:00 AM (`America/Caracas` timezone)
* **Selector Node**: `Evaluar Categoría` (Switch Node):
  * **Output 0 ("Vence hoy")**:
    * Matches: `Vence` == TODAY (`America/Caracas`) and `PLAY != 'SENT'`.
    * **Zelle Path**:
      * **Gmail** (`Send a message Zelle`): Urgent email "*...subscription EXPIRES TODAY!*".
      * **Telnyx SMS** (`EnviarTextoZelle`): Direct text message to customer phone.
      * **WhatsApp Cloud API** (`NotifyClientZelle`): Template `expirestodayzelle|en` with Zelle QR image.
      * **Google Sheets Update** (`Update row in sheet Zelle`): Marks `PLAY = "SENT"` matching by `Usuario`.
    * **Link Path**:
      * Generates 20% discounted NowPayments link.
      * Combines invoice URL with customer data via Code node.
      * **Gmail** (`Send a message Link`): Urgent email with NowPayments link, Cash App, and Zelle instructions.
      * **Telnyx SMS** (`EnviarTextoLink`): Direct text message with instructions to check email.
      * **WhatsApp Cloud API** (`NotifyClientLink`): Template `expirestodaylink|en` with 5 parameters (Name, Email, Full Amount, Discounted Crypto Amount, Checkout URL).
      * **Google Sheets Update** (`Update row in sheet Link`): Marks `PLAY = "SENT"` matching by `Usuario`.
  * **Output 1 ("Limpiar registros pasados")**:
    * Matches: `Vence` < TODAY and `PLAY != ""` (contains past marks).
    * Runs parallel operations directly from the switch output:
      1. **`LimpiarSENTVencidos`**: Google Sheets node clearing `PLAY` column (`" "`) matching by `Usuario` on sheet ID `51202947`.
      2. **`PonerTextoRojoVence`**: Google Sheets API `batchUpdate` HTTP Request setting cell text color to vibrant red (`#d91919`) on Column F (`Vence`) for that row (`sheetId: 51202947`).

### 3.3. `Latin vence hoy y vence4`
* **Workflow ID**: `TfILC2hXao6SLQfE`
* **Target Sheet**: `Clientes TotalTV` $\rightarrow$ `DnSpace` (`1823373862`)
* **Trigger**: Scheduled daily at 9:00 AM
* **Nodes**:
  * Evaluates: Expirations for today (`VenceHoy`, `marcaSENThoy`), 4 days (`Vence4dias`, `marcaSENT4`), and past cleaning (`LimpiarSENTVencidos`).
  * **`PonerTextoRojoVence`**: Connected in parallel with `LimpiarSENTVencidos` to apply red text color (`#d91919`) to the `Vence` date in `DnSpace` for all past rows cleaned.

---

## 4. Credentials & Services Index

| Service | Credential Name in n8n | Credential ID | Type |
| :--- | :--- | :--- | :--- |
| **Google Sheets** | `Google Sheets account` | `Pw5wN2L5UopOruaj` | `googleSheetsOAuth2Api` |
| **Gmail** | `Gmail account` | `CXK8oN7vdrIzstyd` | `gmailOAuth2` |
| **WhatsApp (+17865486856)** | `WhatsApp +17865486856` | `SS1CIclsUQZMU1Bk` | `whatsAppApi` |
| **WhatsApp (3054229099)** | `WhatsApp 3054229099` | `5mdJEz0fPigGF7Wc` | `whatsAppApi` |
| **Evolution API** | `Evolution DigitelTTV` | `vtMtCCsADg27hjPA` | `evolutionApi` |
| **Telnyx Header** | `Telnyx Header Auth` | `wXkPfUVNV0RPIDNg` | `httpMultipleHeadersAuth` |
| **NowPayments** | Direct API Key | `08Y432W-7BA4G0X-KEZ4GWR-5S82WP2` | Header `x-api-key` |
| **Payram** | Direct API Key | `aebb77c33eda3a8eb4c36d8e822a231b` | Header `API-Key` |

---

## 5. Technical Implementation Details & Guardrails

### Safe Field Lookup & Universal Amount Extractor
Used across JavaScript nodes to guarantee robust extraction regardless of casing or accented characters:
```javascript
const getField = (obj, regex) => {
  if (!obj) return '';
  for (const k of Object.keys(obj)) {
    if (regex.test(k)) {
      const v = obj[k];
      if (v !== undefined && v !== null) return v;
    }
  }
  return '';
};

const getAmount = (obj) => {
  if (!obj) return 0;
  for (const k of Object.keys(obj)) {
    if (/monto|amount/i.test(k)) {
      const v = obj[k];
      if (v !== undefined && v !== null && v !== '') {
        const n = parseFloat(String(v).replace(/[^0-9.]/g, ''));
        if (!isNaN(n) && n > 0) return n;
      }
    }
  }
  return 0;
};
```

### Date Normalization (America/Caracas)
Normalizes dates (`YYYY-MM-DD` and `DD/MM/YYYY`) to ISO format for deterministic string comparison against `$now`:
```javascript
const now = new Date();
const caracasOffset = -4 * 60;
const localNow = new Date(now.getTime() + (now.getTimezoneOffset() + caracasOffset) * 60000);
const pad = (n) => String(n).padStart(2, '0');
const todayStr = `${localNow.getFullYear()}-${pad(localNow.getMonth() + 1)}-${pad(localNow.getDate())}`;

const normalizeDate = (val) => {
  if (!val) return '';
  const raw = String(val).trim().split(' ')[0].split('T')[0];
  const p = raw.split(/[\/\-]/);
  if (p.length === 3) {
    return p[0].length === 4
      ? `${p[0]}-${pad(p[1])}-${pad(p[2])}`
      : `${p[2].length === 2 ? '20' + p[2] : p[2]}-${pad(p[1])}-${pad(p[0])}`;
  }
  return raw;
};
```

### Google Sheets Cell Formatting (Red Text)
Calls `POST https://sheets.googleapis.com/v4/spreadsheets/1SNRbfgomUgtac58UmIMlH8UzizBXrTDVogxJEt-z9A0:batchUpdate` using predefined credential `googleSheetsOAuth2Api` (`Pw5wN2L5UopOruaj`):
```json
{
  "requests": [
    {
      "repeatCell": {
        "range": {
          "sheetId": 51202947,
          "startRowIndex": row_number - 1,
          "endRowIndex": row_number,
          "startColumnIndex": 5,
          "endColumnIndex": 6
        },
        "cell": {
          "userEnteredFormat": {
            "textFormat": {
              "foregroundColor": {
                "red": 0.85,
                "green": 0.1,
                "blue": 0.1
              }
            }
          }
        },
        "fields": "userEnteredFormat.textFormat.foregroundColor"
      }
    }
  ]
}
```

---

## 6. Repository Backups Structure

All active workflows are backed up as JSON in the `n8n_backups/` directory:
* `n8n_backups/Mega_expires_SOON.json`: Live export of the 4-day notification workflow for `Mega`.
* `n8n_backups/Mega_expires_TODAY.json`: Live export of the same-day multi-channel workflow with cleaning and red text formatting.
* `n8n_backups/Latin_vence_hoy_y_vence4.json`: Updated workflow for `DnSpace` with red formatting.
* `n8n_backups/Email_expires_SOON.json`: Legacy baseline workflow.
* `n8n_backups/Email_expires_TODAY.json`: Legacy baseline workflow.
* Additional utility backups: `Card2Crypto_to_ME.json`, `Chatwoot_IA_Agent.json`, `Email_Now_Card2Crypto.json`, `NowPayments_to_me.json`, `Telnyx_to_ME.json`.

---

## 7. Change History & Activity Log

### 2026-08-29
- **Initial Setup**: Configured MinGit, GCM, repository remote tracking, and autonomous execution policies in `AGENTS.md` and `.agents/rules/autonomous.md`.
- **Root Cause Fixes**: Fixed broken paired-item references across `Mega expires SOON` and `Mega expires TODAY`, empty `order_id` in NowPayments, and lack of error branches.
- **Direct n8n Integration**: Integrated via n8n Public REST API (`https://n8n.ac4.club/api/v1/`) using API Token authentication to update and activate workflows directly on the production instance.
- **Syntax & Encoding Fix**: Sanitized non-ASCII identifiers in Node.js Code nodes (`TelÃ©fono` $\rightarrow$ `Telefono`) preventing V8 syntax errors.
- **Google Sheets ID Correction**: Discovered real numeric sheet ID for `Mega` tab (**`51202947`** vs legacy `1037714867`), resolving `Sheet with ID not found` and `No grid with id` errors in `LimpiarSENTVencidos`, `PonerTextoRojoVence`, and `Update row in sheet` nodes.
