# 📬 Client Mail Automation — n8n + Gmail + WhatsApp

An automated client communication workflow built with **n8n** that sends a personalized **email** and **WhatsApp message** to a client the moment you submit a Google Form — and automatically marks the status as **Completed** in Google Sheets.

---

## 🔄 Workflow Overview

```
Google Form Submitted
        ↓
Google Sheets Trigger (detects new row)
        ↓
Filter (only rows with Status ≠ Completed)
        ↓
Gmail → Send email to client
        ↓
WhatsApp (Meta API) → Send message to client
        ↓
Google Sheets → Update Status to "Completed"
```

---

## ✅ What You Need Before Starting

| Requirement | Details |
|---|---|
| n8n | Local install or [n8n Cloud](https://n8n.io) |
| Google Account | Gmail + Google Sheets + Google Cloud Console |
| Meta Developer Account | For WhatsApp Business Cloud API |
| A Google Form | Linked to a Google Sheet |

---

## 📋 Google Sheet Structure

Your Google Sheet (linked to the form) must have these columns in this exact order:

| Column | Header Name | Description |
|---|---|---|
| A | Timestamp | Auto-filled by Google Forms |
| B | Client Name | Client's full name |
| C | Client Email | Client's email address |
| D | Client WhatsApp Number | 10-digit number (no country code) |
| E | Amount | Payment / invoice amount |
| F | Mail Subject | Subject line for the email |
| G | Mail Body | Body text of the email/WhatsApp message |
| H | Status | Leave blank for new rows — n8n fills this |

> ⚠️ Add a **Status** column (Column H) manually. New form submissions leave it blank. n8n writes **Completed** after sending.

---

## 🔧 Step-by-Step Setup

### Step 1 — Enable Google APIs

1. Go to [Google Cloud Console](https://console.cloud.google.com)
2. Create a new project (or use an existing one)
3. Go to **APIs & Services → Library**
4. Search and enable:
   - **Gmail API**
   - **Google Sheets API**
5. Go to **APIs & Services → Credentials**
6. Click **Create Credentials → OAuth 2.0 Client ID**
7. Application type: **Web application**
8. Add Authorized redirect URI:
   ```
   https://YOUR-N8N-URL/rest/oauth2-credential/callback
   ```
   (For local n8n: `http://localhost:5678/rest/oauth2-credential/callback`)
9. Save your **Client ID** and **Client Secret**

---

### Step 2 — Set Up Google Credentials in n8n

1. Open n8n → **Settings → Credentials → Add Credential**
2. Search for **Google Sheets OAuth2 API**
3. Enter your Client ID and Client Secret → Connect with Google
4. Repeat for **Gmail OAuth2 API** with the same credentials
5. Grant permissions when prompted

---

### Step 3 — Set Up Meta WhatsApp Business API

1. Go to [developers.facebook.com](https://developers.facebook.com)
2. Create a new app → Choose type: **Business**
3. Add **WhatsApp** product to your app
4. Go to **WhatsApp → API Setup** and collect:
   - **Phone Number ID** (long numeric ID, e.g. `123456789012345`)
   - **Temporary or Permanent Access Token** (starts with `EAA...`)

> 💡 For a **permanent token** (recommended):
> Meta Business Suite → Settings → System Users → Create → Generate Token → select your app → enable `whatsapp_business_messaging` scope

5. Create a **Message Template**:
   - Go to Meta Business Suite → WhatsApp Manager → Message Templates
   - Click **Create Template**
   - Name: `client_notification`
   - Category: `Utility`
   - Body:
     ```
     Hello {{1}}, {{2}} If you have any questions, feel free to reply to this message.
     ```
   - Submit for approval (usually approved within a few hours)

> 📱 For **testing only**: Go to WhatsApp → API Setup → "To" section → Add up to 5 recipient numbers to the test list. Free-form text works without a template for these numbers.

---

### Step 4 — Import the Workflow into n8n

1. Download `client mail automation using n8n.json` from this repository
2. Open n8n → click **+** to create a new workflow
3. Click the **three-dot menu (⋮)** → **Import from file**
4. Select the downloaded `client mail automation using n8n.json`
5. The full workflow loads with all nodes

---

### Step 5 — Configure Each Node

After importing, open each node and reconnect your credentials:

#### 🟢 Google Sheets Trigger
- Document: Select your Google Sheet
- Sheet: `Form Responses 1`
- Trigger: `Row Added`
- Poll interval: `1 minute`

#### 🔵 Filter Node
- Condition: `Status` `is not equal to` `Completed`
- Enable: `Convert types where required`

#### 📧 Gmail — Send a message
- Credential: Select your Gmail OAuth2 account
- To: `{{ $('Google Sheets Trigger').item.json['Client Email'] }}`
- Subject: `{{ $('Google Sheets Trigger').item.json['Mail Subject'] }}`
- Body: `{{ $('Google Sheets Trigger').item.json['Mail Body'] }}`
- Email Type: `HTML`

#### 💬 HTTP Request — WhatsApp Message
- Method: `POST`
- URL:
  ```
  https://graph.facebook.com/v20.0/YOUR_PHONE_NUMBER_ID/messages
  ```
- Authentication: `Generic Credential Type → Header Auth`
  - Name: `Authorization`
  - Value: `Bearer YOUR_ACCESS_TOKEN`
- Body Content Type: `JSON`
- Body:
  ```json
  {
    "messaging_product": "whatsapp",
    "to": "91{{ $('Google Sheets Trigger').item.json['Client WhatsApp Number'] }}",
    "type": "template",
    "template": {
      "name": "client_notification",
      "language": { "code": "en_US" },
      "components": [{
        "type": "body",
        "parameters": [
          { "type": "text", "text": "{{ $('Google Sheets Trigger').item.json['Client Name'] }}" },
          { "type": "text", "text": "{{ $('Google Sheets Trigger').item.json['Mail Body'] }}" }
        ]
      }]
    }
  }
  ```

#### 🟢 Google Sheets — Update row in sheet
- Operation: `Update`
- Document: Your Google Sheet
- Sheet: `Form Responses 1`
- Mapping Column Mode: `Map Each Column Manually`
- Column to match on: `Timestamp`
- Values to Update:
  - `Timestamp` → `{{ $('Google Sheets Trigger').item.json['Timestamp'] }}`
  - `Status` → `Completed`

---

### Step 6 — Activate the Workflow

1. Click the **toggle** at the top right of the workflow (Inactive → Active)
2. The workflow now runs automatically every time a form is submitted

---

## 🧪 Testing the Workflow

1. Open your Google Sheet
2. Manually set one row's Status to blank or a non-Completed value
3. Go back to n8n → open the workflow → click **Execute workflow**
4. Watch each node turn green ✅
5. Check the client's email inbox and WhatsApp

---

## 📁 Repository Files

```
├── client mail automation using n8n.json        ← Import this into n8n
├── README.md                                    ← This file
└── .gitignore                                   ← Prevents accidental credential exposure
```

---

## 🔐 Security Notes

> **Never commit these to GitHub:**
> - API keys or access tokens
> - OAuth Client ID / Client Secret
> - n8n credential database files
> - Any `.env` files with real values

The `workflow.json` file exported from n8n **does not contain your credentials** — it only stores references. Each person who imports the workflow must connect their own credentials in n8n's Credentials panel.

---

## 🛠 Tech Stack

| Tool | Purpose |
|---|---|
| [n8n](https://n8n.io) | Workflow automation |
| Google Forms | Client data collection |
| Google Sheets | Data storage + status tracking |
| Gmail API | Sending emails |
| Meta WhatsApp Business Cloud API | Sending WhatsApp messages |

---

## 📄 License

MIT License — free to use, modify, and distribute.

---

## 👨‍💻 Author

Built by [@codewithkushagra](https://github.com/codewithkushagra)

---

## 🚀 Live Demo — Try It Yourself

Follow these steps to see the full automation in action:

### Step 1 — Fill the Google Form

👉 **[Click here to open the form](https://docs.google.com/forms/d/e/1FAIpQLSdK9p-EH2wC3OWzMylfbxNXjsaYqfNcA128LkSmWist1OcEFA/viewform)**

Fill in the following fields:

| Field | What to enter |
|---|---|
| Client Name | Your name |
| Client Email | Your email address |
| Client WhatsApp Number | Your 10-digit WhatsApp number (no +91) |
| Amount | Any number (e.g. 5000) |
| Mail Subject | Subject line for the email |
| Mail Body | The message you want to send |

Click **Submit**.

---

### Step 2 — Watch the Google Sheet Update Live

👉 **[Click here to open the Google Sheet](https://docs.google.com/spreadsheets/d/1ZeuECSsi197qK3ZOss1YmOYz0Qyyt_GU0SV8oz5s8-k/edit?usp=sharing)**

Within 60 seconds of submitting the form you will see:

- ✅ A new row appears with your submitted data
- ✅ The **Status** column changes to **Completed**
- ✅ An email arrives in the inbox you entered
- ✅ A WhatsApp message arrives on the number you entered

> 💡 Keep the sheet open and watch the Status column — it updates automatically once n8n processes the row.

---

### Step 3 — Copy the Form to Your Own Account

Want to use this form in your own setup?

👉 **[Click here to make a copy of the form](https://docs.google.com/forms/d/1i8NrRDs2xjYg6HgsVzxRzI0vJQ0AqhTizvDhzMowDAE/copy)**

This creates your own editable copy. Then follow the [setup guide above](#-step-by-step-setup) to link it to your own Google Sheet and n8n workflow.

---

## 🗂 Google Form Questions

The form collects these fields (same as the Google Sheet columns):

| # | Question | Type |
|---|---|---|
| 1 | Client Name | Short answer |
| 2 | Client Email | Short answer |
| 3 | Client WhatsApp Number | Short answer |
| 4 | Amount | Short answer |
| 5 | Mail Subject | Short answer |
| 6 | Mail Body | Paragraph |

> After making a copy, link your form to a Google Sheet: **Responses tab → Link to Sheets → Create new sheet**. Then update the Google Sheets Trigger node in n8n to point to your new sheet.

---

## 📥 How to Import the Workflow

1. Download the `client mail automation using n8n.json` file from this repository
2. Open your n8n dashboard
3. Click **"+ New Workflow"**
4. Click the **three-dot menu (⋮)** at the top right → **"Import from file"**
5. Select the downloaded `client mail automation using n8n.json`
6. All 5 nodes load automatically — now set up your credentials below

---

## 🔑 Credentials Setup — What You Need to Fill in Each Node

After importing the workflow, each node that needs credentials will show a **red warning**. Follow these steps to fix each one.

---

### 📧 Node 3 — Gmail "Send a message" node

**What you need:** A Google account with Gmail

**Steps:**
1. Click the **Gmail node** → open it
2. Under **Credential** → click the dropdown → click **"Create new credential"**
3. Select **"Gmail OAuth2"**
4. You need a **Google Cloud OAuth2 Client ID and Secret**:
   - Go to [console.cloud.google.com](https://console.cloud.google.com)
   - Create a project → Enable **Gmail API**
   - Go to **Credentials → Create → OAuth 2.0 Client ID**
   - Application type: **Web application**
   - Authorized redirect URI: `http://localhost:5678/rest/oauth2-credential/callback`
   - Copy the **Client ID** and **Client Secret**
5. Paste them in n8n → click **"Connect my account"** → sign in with your Google account
6. Grant all permissions when prompted

**Node fields to fill:**
| Field | Value |
|---|---|
| Credential | Your connected Gmail account |
| Resource | Message |
| Operation | Send |
| To | `{{ $('Google Sheets Trigger').item.json['Client Email'] }}` |
| Subject | `{{ $('Google Sheets Trigger').item.json['Mail Subject'] }}` |
| Message | `{{ $('Google Sheets Trigger').item.json['Mail Body'] }}` |
| Email Type | HTML |

---

### 🟢 Node 1 & 5 — Google Sheets nodes (Trigger + Update row)

**What you need:** Same Google OAuth2 credentials as Gmail above

**Steps:**
1. Click the **Google Sheets Trigger node** → open it
2. Under **Credential** → click **"Create new credential"** → **"Google Sheets OAuth2"**
3. Use the **same Client ID and Client Secret** from your Google Cloud project
   - Make sure **Google Sheets API** is also enabled in your Cloud project
4. Connect your Google account
5. Select your **Spreadsheet** and **Sheet** from the dropdowns
6. Repeat the same credential connection for the **"Update row in sheet"** node

---

### 💬 Node 4 — WhatsApp HTTP Request node (Meta API)

**What you need:** A Meta Developer account with WhatsApp Business API

**Steps to get your credentials:**
1. Go to [developers.facebook.com](https://developers.facebook.com)
2. Create a new app → type: **Business**
3. Add **WhatsApp** product → go to **WhatsApp → API Setup**
4. Copy your **Phone Number ID** (long number like `123456789012345`)
5. Generate a **Permanent Access Token**:
   - Meta Business Suite → Settings → System Users → Add → role: Admin
   - Click "Generate New Token" → select your app → enable `whatsapp_business_messaging`
   - Copy the token (starts with `EAA...`)

**Node fields to fill:**
| Field | Value |
|---|---|
| Method | POST |
| URL | `https://graph.facebook.com/v20.0/YOUR_PHONE_NUMBER_ID/messages` |
| Authentication | Generic Credential Type → Header Auth |
| Header Name | `Authorization` |
| Header Value | `Bearer YOUR_ACCESS_TOKEN` |
| Body Content Type | JSON |

**JSON Body:**
```json
{
  "messaging_product": "whatsapp",
  "to": "91{{ $('Google Sheets Trigger').item.json['Client WhatsApp Number'] }}",
  "type": "template",
  "template": {
    "name": "client_notification",
    "language": { "code": "en_US" },
    "components": [{
      "type": "body",
      "parameters": [
        { "type": "text", "text": "{{ $('Google Sheets Trigger').item.json['Client Name'] }}" },
        { "type": "text", "text": "{{ $('Google Sheets Trigger').item.json['Mail Body'] }}" }
      ]
    }]
  }
}
```

> ⚠️ **WhatsApp Template required:** Create a message template named `client_notification` in Meta Business Suite → WhatsApp Manager → Message Templates. Use category: **Utility**. Wait for approval (usually a few hours) before the workflow can send WhatsApp messages.

---

## ✅ Final Checklist Before Going Live

| Step | What to do |
|---|---|
| ☐ Import workflow | Import `workflow.json` into n8n |
| ☐ Google Sheets credential | Connect Google account to both Sheets nodes |
| ☐ Gmail credential | Connect Google account to Gmail node |
| ☐ WhatsApp credential | Add Meta Phone Number ID + Access Token to HTTP node |
| ☐ WhatsApp template | Create & get `client_notification` template approved in Meta |
| ☐ Google Sheet structure | Make sure sheet has all 8 columns (Timestamp, Client Name, Client Email, Client WhatsApp Number, Amount, Mail Subject, Mail Body, Status) |
| ☐ Activate workflow | Click **Publish** button in n8n → confirm in popup |
| ☐ Test it | Submit the Google Form → check email, WhatsApp, and sheet status |

