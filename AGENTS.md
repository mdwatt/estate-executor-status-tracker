# ESTATE EXECUTOR STATUS TRACKER
**Build this exactly as specified. No additional features beyond what's listed. No shortcuts. No substitutions.**

---

## **PRODUCT OVERVIEW**

Build a Flask-based executor status tracker web application that helps attorneys monitor probate document submissions from executors. This is a linear, server-rendered HTML application with PostgreSQL database and cloud-hosted local filesystem document storage (no AWS/S3).

**Core Purpose:** Attorneys create cases, executors upload probate documents via magic link, attorneys review submissions and close cases when complete. State-specific probate form guides are built into the application.

---

## **DEPLOYMENT (CLOUD MVP)**

**Target:** Cloud-hosted deployment (not local-only).

**MVP hosting model (single box):**
- 1 cloud VM runs:
  - Flask app (Gunicorn behind Nginx)
  - PostgreSQL
  - Upload storage on VM disk (`UPLOAD_ROOT_DIR`)
- Domain + TLS required (executor magic link must be HTTPS)

**No AWS/S3. No object storage in MVP.** All uploaded documents are stored on the VM filesystem.

---

## **CONFIGURATION (ENV VARS)**

**Required:**
- `APP_BASE_URL` (e.g., `https://app.yourdomain.com`)
- `FLASK_SECRET_KEY`
- `DATABASE_URL` (PostgreSQL)
- `UPLOAD_ROOT_DIR` (e.g., `/var/app/uploads`)
- `MAX_UPLOAD_MB` (default `25`)

**Mailgun:**
- `MAILGUN_API_KEY`
- `MAILGUN_DOMAIN`
- `MAILGUN_FROM_EMAIL`

**Stripe (subscription required to create cases):**
- `STRIPE_SECRET_KEY`
- `STRIPE_WEBHOOK_SECRET`
- `STRIPE_PRICE_ID`

---

## **AUTHENTICATION & ACCESS CONTROL**

**Executors:**
- No login system — magic-link email only
- Magic links expire after 30 days
- One magic link per case, reusable by executor
- Link format: `/executor/<magic_token>`

**Attorneys:**
- No authentication in MVP — direct URL access to attorney pages
- All attorney routes start with `/attorney/`

**Security:**
- Magic tokens must be cryptographically secure (use `secrets.token_urlsafe(32)`)
- Validate magic_token on every executor request
- No executor can access another executor's case

---

## **BILLING & STRIPE SUBSCRIPTION (PAID APP)**

**Billing model:** Attorney subscription. Executors are never charged.

**Rule:** An attorney must have an **active** Stripe subscription to:
- Create a new case (`/attorney/case/new`)
- Export case PDF (`/attorney/case/<case_id>/export`)
- Close a case (`/attorney/case/<case_id>/close`)

**Attorney identity in MVP (no login):**
- First visit to any `/attorney/*` route shows a required form: `Attorney Email`
- Store in Flask session: `session['attorney_email']`
- All billing checks key off `session['attorney_email']`

**Routes:**
- `GET /attorney/billing` — shows current subscription status and "Subscribe" button
- `POST /attorney/billing/checkout` — creates Stripe Checkout Session (mode=subscription) and redirects to Stripe-hosted checkout
- `POST /stripe/webhook` — Stripe webhook receiver (updates subscription status in DB)
- `GET /attorney/billing/success` — post-checkout landing page
- `GET /attorney/billing/cancel` — post-cancel landing page

**Stripe Python SDK:**
- Use `stripe` (Python) only. No JS/NPM.

**Checkout Session (server-side):**
```python
import os
import stripe
from flask import Blueprint, session, redirect, abort

stripe.api_key = os.environ["STRIPE_SECRET_KEY"]
STRIPE_PRICE_ID = os.environ["STRIPE_PRICE_ID"]
APP_BASE_URL = os.environ["APP_BASE_URL"]

billing_bp = Blueprint("billing", __name__)

@billing_bp.post("/attorney/billing/checkout")
def billing_checkout():
    attorney_email = session.get("attorney_email")
    if not attorney_email:
        abort(400)

    # DB: upsert attorney row by attorney_email, create Stripe customer if missing
    # attorney = get_or_create_attorney(attorney_email)

    customer_id = None  # attorney.stripe_customer_id
    if not customer_id:
        customer = stripe.Customer.create(email=attorney_email, metadata={"attorney_email": attorney_email})
        customer_id = customer["id"]
        # persist customer_id

    checkout = stripe.checkout.Session.create(
        mode="subscription",
        customer=customer_id,
        line_items=[{"price": STRIPE_PRICE_ID, "quantity": 1}],
        success_url=f"{APP_BASE_URL}/attorney/billing/success",
        cancel_url=f"{APP_BASE_URL}/attorney/billing/cancel",
        allow_promotion_codes=True,
        client_reference_id=attorney_email,
    )
    return redirect(checkout["url"], code=303)
```

**Webhook (server-side):**
```python
import os
import stripe
from flask import Blueprint, request, abort

stripe.api_key = os.environ["STRIPE_SECRET_KEY"]
STRIPE_WEBHOOK_SECRET = os.environ["STRIPE_WEBHOOK_SECRET"]

webhook_bp = Blueprint("stripe_webhook", __name__)

@webhook_bp.post("/stripe/webhook")
def stripe_webhook():
    payload = request.data
    sig = request.headers.get("Stripe-Signature")
    try:
        event = stripe.Webhook.construct_event(payload, sig, STRIPE_WEBHOOK_SECRET)
    except Exception:
        abort(400)

    etype = event["type"]
    obj = event["data"]["object"]

    if etype in ("customer.subscription.created", "customer.subscription.updated", "customer.subscription.deleted"):
        customer_id = obj["customer"]
        subscription_id = obj["id"]
        status = obj["status"]  # active, past_due, canceled, etc.
        period_end = obj.get("current_period_end")

        mapped = {
            "active": "active",
            "trialing": "active",
            "past_due": "past_due",
            "unpaid": "past_due",
            "canceled": "canceled",
            "incomplete": "inactive",
            "incomplete_expired": "inactive",
        }.get(status, "inactive")

        # DB: find attorney by stripe_customer_id, update:
        # stripe_subscription_id, subscription_status, subscription_current_period_end

    return ("", 200)
```

**Subscription gate (server-side):**
```python
def require_active_subscription(attorney) -> None:
    if attorney.subscription_status != "active":
        raise PermissionError("inactive_subscription")
```

---

## **DATABASE SCHEMA**

### **Table: attorneys**
```
attorney_email (TEXT, primary key)
stripe_customer_id (TEXT, nullable, unique)
stripe_subscription_id (TEXT, nullable, unique)
subscription_status (TEXT, 'inactive'|'active'|'past_due'|'canceled', default 'inactive')
subscription_current_period_end (TIMESTAMP, nullable)
created_at (TIMESTAMP, default now)
updated_at (TIMESTAMP, default now)
```

### **Table: estate_cases**
```
case_id (UUID, primary key)
decedent_name (TEXT, required)
executor_name (TEXT, required)
executor_email (TEXT, required)
attorney_email (TEXT, required)
state (TEXT, required, 2-letter code: 'NC', 'CA', 'TX', etc.)
magic_token (TEXT, unique, indexed)
status (TEXT, 'Open' or 'Closed', default 'Open')
created_at (TIMESTAMP, default now)
closed_at (TIMESTAMP, nullable)
attorney_notes (TEXT, nullable)
```

### **Table: checklist_items**
```
item_id (UUID, primary key)
case_id (UUID, foreign key to estate_cases)
step_number (INTEGER, 1-12, required)
step_name (TEXT, required, immutable)
status (TEXT, 'Not Started', 'Submitted', or 'Reviewed', default 'Not Started')
last_updated (TIMESTAMP, default now)
attorney_note (TEXT, nullable)
```

### **Table: documents**
```
document_id (UUID, primary key)
checklist_item_id (UUID, foreign key to checklist_items)
file_path (TEXT, relative path under UPLOAD_ROOT_DIR)
file_name (TEXT, original filename)
file_size (INTEGER, bytes)
uploaded_by (TEXT, 'executor')
uploaded_at (TIMESTAMP, default now)
deleted_at (TIMESTAMP, nullable)
soft_deleted (BOOLEAN, default false)
```

### **Table: state_guides**
```
guide_id (UUID, primary key)
state_code (TEXT, 2-letter, unique, indexed)
state_name (TEXT, required)
guide_content (JSONB, structured form mapping data)
created_at (TIMESTAMP, default now)
updated_at (TIMESTAMP, default now)
```

**JSONB Structure for guide_content:**
```json
{
  "steps": {
    "3": {
      "forms": ["AOC-E-201", "AOC-E-650"],
      "form_names": ["Application for Probate", "Estates Action Cover Sheet"],
      "notes": "Opens the probate estate with the court"
    },
    "4": {
      "forms": ["AOC-E-400", "AOC-E-403"],
      "form_names": ["Executor's Oath", "Letters Testamentary"],
      "notes": "Grants executor legal authority"
    }
  },
  "deadlines": {
    "11": "90 days from qualification"
  },
  "resources": {
    "forms_url": "https://www.nccourts.gov/forms",
    "statutes": "NCGS Chapter 28A"
  }
}
```


## **MIGRATIONS & SEEDING (LOCKED)**

**Migrations:** Alembic only.

**Requirements:**
- `alembic upgrade head` creates all tables.
- Seeding runs via a Flask CLI command and is idempotent.

**Seed command:**
- `flask seed` must upsert:
  - `state_guides` (initial states)
  - the 12 checklist step definitions used to create per-case checklist_items

**Acceptance check:**
1. Run migrations
2. Run `flask seed`
3. Create a case
4. Exactly 12 checklist_items are created for that case


---

## **THE 12 IMMUTABLE CHECKLIST STEPS**

When a case is created, seed these 12 checklist items automatically:

1. Certified Death Certificate(s) Obtained
2. Original Will Located
3. Probate Case Filed with Court
4. Letters Testamentary Issued
5. Estate Bank Account Opened
6. All Bank & Financial Accounts Identified
7. Real Property Secured and Identified
8. Insurance Policies Located
9. Creditors Notified
10. Outstanding Debts Identified
11. Inventory of Assets Completed
12. Final Accounting Prepared

**These steps never change. State-specific form numbers are displayed via the state guide system.**

---

## **SCREEN 1: EXECUTOR LANDING (MAGIC LINK)**

**Route:** `/executor/<magic_token>`

**Logic:**
- Look up case by magic_token
- If token invalid, expired (>30 days), or case closed → show error page
- Display decedent name, case ID (read-only), executor name
- Show all 12 checklist items in order with current status
- For each item: step number, step name, status badge, action button
- Action button logic:
  - "Upload" (primary button) if status = 'Not Started'
  - "View" (secondary button) if status = 'Submitted' or 'Reviewed'
- Footer: "Need help? support@stateexecutortracker.com"

**UI Notes:**
- Status badges: gray for Not Started, yellow for Submitted, green for Reviewed
- Mobile-responsive: stack on small screens
- No navigation menu (single-purpose page)

---

## **SCREEN 2: UPLOAD DOCUMENT**

**Route:** `/executor/<magic_token>/upload/<item_id>`

**Logic:**
- Validate magic_token and case status (must be Open)
- Display step name dynamically from checklist_items table
- Show current document if one exists (with "Replace Document" option)
- File upload input: accept .pdf, .jpg, .png only, max 25MB
- Two buttons: Cancel (returns to Screen 1) and Submit

**On Submit:**
1. Validate file type and size server-side
2. If document already exists for this item:
   - Soft delete old document (set soft_deleted=true, deleted_at=now)
   - Keep old document on disk for 30 days (soft-deleted)
3. Save new file to disk: `{UPLOAD_ROOT_DIR}/{case_id}/{item_id}/{document_id}/{timestamp}_{filename}`
4. Create new document record in documents table
5. Update checklist_items: status='Submitted', last_updated=now
6. Redirect to Screen 1 with success message

**Upload Storage (LOCKED — NO AWS):**
- Save files to VM disk under `UPLOAD_ROOT_DIR`
- Store `documents.file_path` as a **relative** path from `UPLOAD_ROOT_DIR`
- Download endpoint must verify either:
  - valid executor magic_token for the case, OR
  - attorney session email for the case
- Provide 1-hour signed download tokens for attorney "View File" links

**Save upload (server-side):**
```python
import os
from pathlib import Path
from werkzeug.utils import secure_filename

UPLOAD_ROOT_DIR = os.getenv("UPLOAD_ROOT_DIR", "./data/uploads")

def save_upload(file_storage, case_id: str, item_id: str, document_id: str, ts_prefix: str) -> str:
    filename = secure_filename(file_storage.filename or "upload.bin")
    rel_dir = Path(case_id) / item_id / document_id
    abs_dir = Path(UPLOAD_ROOT_DIR) / rel_dir
    abs_dir.mkdir(parents=True, exist_ok=True)
    rel_path = rel_dir / f"{ts_prefix}_{filename}"
    abs_path = Path(UPLOAD_ROOT_DIR) / rel_path
    file_storage.save(abs_path)
    return str(rel_path)
```

**Signed download token (1 hour):**
```python
from itsdangerous import URLSafeTimedSerializer

def make_download_token(secret_key: str, document_id: str) -> str:
    s = URLSafeTimedSerializer(secret_key)
    return s.dumps({"document_id": document_id})

def verify_download_token(secret_key: str, token: str, max_age_seconds: int = 3600) -> str:
    s = URLSafeTimedSerializer(secret_key)
    data = s.loads(token, max_age=max_age_seconds)
    return data["document_id"]
```

**Validation:**
- File size: max 25MB
- File types: .pdf, .jpg, .png only (check magic bytes, not just extension)
- One active document per checklist item (old ones soft-deleted)

**Error Handling:**
- File too large → "File exceeds 25MB limit"
- Invalid file type → "Only PDF, JPG, and PNG files allowed"
- Upload failed → "Upload failed. Please try again."

---

## **SCREEN 3: ATTORNEY VIEW (SINGLE CASE)**

**Route:** `/attorney/case/<case_id>`

**Layout:**

**Top Section: Case Header**
- Decedent name (editable inline or via edit button)
- Executor name (editable)
- Executor email (editable)
- Case status: Open/Closed (read-only badge)
- State: [state_code] (display only, cannot change after creation)
- Created date

**Summary Counts:**
- Not Started: [count]
- Submitted: [count]
- Reviewed: [count]

**State Guide Panel (Collapsible):**
- Title: "[State Name] Probate Forms Quick Reference"
- Display state-specific form mappings from state_guides table
- Show forms for steps 3, 4, 9, 11, 12 (critical steps)
- "Print Guide" button (generates printable HTML page)

**Checklist Table:**
Columns: Step | Status | Last Updated | Document | Action

For each checklist item:
- Step number and name
- Status badge (color-coded)
- Last updated timestamp
- Document column:
  - If document exists: "View File" link (`/attorney/document/<document_id>/download`, 1-hour signed token)
  - If document exists: "Delete" button (red, with confirmation)
  - If no document: "—"
- Action column:
  - If status='Submitted': "Mark Reviewed" button (green)
  - If status='Reviewed': "Reviewed ✓" (disabled, green text)
  - If status='Not Started': "—"

**Attorney Notes Section:**
- Large textarea below table
- Label: "Internal Notes (Attorney Only — Not visible to executor)"
- Auto-saves on blur or "Save Notes" button
- Stores in estate_cases.attorney_notes

**Bottom Actions:**
- "Export Case to PDF" button (generates PDF summary)
- "Edit Case Details" button (opens edit modal/form)
- "Close Case" button (only enabled if all 12 items = 'Reviewed', red button)

**Actions:**

**Mark Reviewed:**
- Update checklist_items: status='Reviewed', last_updated=now
- Refresh page to update summary counts

**Delete Document:**
- Show confirmation modal: "Are you sure? This will remove the document. The executor will need to upload it again. This action cannot be undone."
- Buttons: "Cancel" and "Delete Document" (red)
- On confirm:
  - Update documents: soft_deleted=true, deleted_at=now
  - Update checklist_items: status='Not Started', last_updated=now
  - After 30 days, background job purges soft-deleted files from disk
- Refresh page

**Edit Case Details:**
- Modal or inline form with fields:
  - Decedent name
  - Executor name
  - Executor email
- "Save Changes" button
- Update estate_cases record
- Do NOT allow changing state after creation

**Export Case to PDF:**
- Generate PDF with:
  - Case header (decedent, executor, dates)
  - Summary counts
  - All 12 checklist items with status
  - Attorney notes
  - Document filenames (not actual files, just list)
- Use a PDF library (e.g., ReportLab or WeasyPrint)
- Download as: `[decedent_name]_Estate_Case_[case_id].pdf`

**Close Case:**
- Redirect to Screen 5 (Close Case Confirmation)

---

## **SCREEN 4: ATTORNEY CASE LIST**

**Route:** `/attorney/cases`

**Two Tabs:**
1. **Active Cases** (default)
2. **Closed Cases**

**Active Cases Tab:**
- Show only cases where status='Open'
- Display for each case:
  - Decedent name (bold, clickable)
  - Executor name
  - State code badge (e.g., "NC")
  - Not Started count
  - Submitted count
  - Last activity date
- Sort by: (Not Started + Submitted) DESC, then last_updated ASC (oldest first)
- Each row clickable → opens Screen 3 for that case
- Top-right: "New Case" button (green, primary)

**Closed Cases Tab:**
- Show only cases where status='Closed'
- Display for each case:
  - Decedent name (clickable)
  - Executor name
  - State code
  - Closed date
- Sort by: closed_at DESC (most recent first)
- Each row clickable → opens Screen 3 in read-only mode
- In read-only mode: no edit buttons, no mark reviewed, no delete, show "Case Closed" banner

**No search in MVP. No filters. No bulk actions.**

---

## **SCREEN 5: CLOSE CASE CONFIRMATION**

**Route:** `/attorney/case/<case_id>/close`

**Pre-check:**
- Verify all 12 checklist items have status='Reviewed'
- If not, redirect back to Screen 3 with error message

**Display:**
- Header: "Close Case: [Decedent Name]"
- Warning banner: "⚠️ Confirm Case Closure"
- Confirmation message: "All checklist items have been reviewed. Are you sure you want to close this case?"
- Summary box:
  - Decedent: [name]
  - Executor: [name]
  - All items: Reviewed ✓
  - Case opened: [created_at date]
- Note: "Closed cases will be moved to the Closed Cases list and marked read-only. This action cannot be undone in MVP."
- Two buttons: "Cancel" (gray) and "Close Case" (red)

**On Close Case:**
1. Update estate_cases: status='Closed', closed_at=now
2. Redirect to Screen 4 with success message: "Case closed successfully"
3. Case now appears in Closed Cases tab

---

## **CREATE CASE FORM**

**Route:** `/attorney/case/new`

**Can be modal on Screen 4 or separate page.**

**Fields:**
1. Decedent Full Name (text, required)
2. Executor Full Name (text, required)
3. Executor Email (email, required)
4. Attorney Email (email, required, pre-filled if known)
5. **State (dropdown, required):**
   - Options: NC (North Carolina), CA (California), TX (Texas), etc.
   - Load from state_guides table where guide exists
   - Default: NC

**On Submit:**
1. Validate all required fields
2. Generate case_id (UUID)
3. Generate magic_token (secrets.token_urlsafe(32))
4. Create estate_cases record
5. Insert 12 checklist_items records (step_number 1-12, status='Not Started')
6. Send email to executor_email with magic link
7. Redirect attorney to Screen 3 for new case with success message

**Email to Executor:**
- Subject: `[Decedent Name] Estate — Action Required`
- Body:
```
Hello [Executor Name],

You have been designated as executor for the estate of [Decedent Name].

Please upload the required probate documents using this secure link:
[magic link URL]

This link will remain active for 30 days. You can return to it as many times as needed to upload documents.

If you have questions, contact your attorney at [Attorney Email] or our support team at support@stateexecutortracker.com.

Thank you,
Executor Status Tracker
```

**Email Delivery (LOCKED — Mailgun HTTP API):**
- Use Mailgun v3 HTTP API via `requests`
- No SMTP

```python
import os
import requests

MAILGUN_API_KEY = os.environ["MAILGUN_API_KEY"]
MAILGUN_DOMAIN = os.environ["MAILGUN_DOMAIN"]
MAILGUN_FROM_EMAIL = os.environ["MAILGUN_FROM_EMAIL"]

def send_executor_magic_link(to_email: str, subject: str, body: str) -> None:
    url = f"https://api.mailgun.net/v3/{MAILGUN_DOMAIN}/messages"
    resp = requests.post(
        url,
        auth=("api", MAILGUN_API_KEY),
        data={"from": MAILGUN_FROM_EMAIL, "to": [to_email], "subject": subject, "text": body},
        timeout=20,
    )
    resp.raise_for_status()
```

---

## **STATE GUIDE SYSTEM**

### **Initial State Guides to Seed**

**North Carolina (NC):**
```json
{
  "steps": {
    "1": {
      "notes": "Certified death certificate from NC Vital Records"
    },
    "3": {
      "forms": ["AOC-E-201", "AOC-E-650"],
      "form_names": ["Application for Probate", "Estates Action Cover Sheet"],
      "notes": "Opens probate estate with Clerk of Superior Court"
    },
    "4": {
      "forms": ["AOC-E-400", "AOC-E-403"],
      "form_names": ["Executor's Oath", "Letters Testamentary"],
      "notes": "Grants executor legal authority to act"
    },
    "9": {
      "forms": ["AOC-E-307"],
      "form_names": ["Affidavit of Notice to Creditors"],
      "notes": "Must publish notice and mail to known creditors"
    },
    "11": {
      "forms": ["AOC-E-505"],
      "form_names": ["Inventory"],
      "notes": "90-day deadline from qualification"
    },
    "12": {
      "forms": ["AOC-E-506", "AOC-E-507"],
      "form_names": ["Annual Accounting", "Final Accounting"],
      "notes": "Required before estate can close"
    }
  },
  "deadlines": {
    "11": "90 days from executor qualification",
    "9": "Creditor claim period: 90 days from first publication"
  },
  "resources": {
    "forms_url": "https://www.nccourts.gov/forms",
    "statutes": "North Carolina General Statutes Chapter 28A"
  }
}
```

**Template for Other States:**
```json
{
  "steps": {
    "3": {
      "forms": ["[State Form Number]"],
      "form_names": ["[Form Name]"],
      "notes": "[State-specific notes]"
    }
  },
  "deadlines": {},
  "resources": {
    "forms_url": "[State court website]",
    "statutes": "[State statute reference]"
  }
}
```

### **Displaying State Guides**

On Screen 3 (Attorney View), render the state guide as:

```
[STATE NAME] PROBATE FORMS QUICK REFERENCE

Step 3: Probate Case Filed with Court
→ Forms: AOC-E-201 (Application for Probate), AOC-E-650 (Estates Action Cover Sheet)
→ Notes: Opens probate estate with Clerk of Superior Court

Step 4: Letters Testamentary Issued
→ Forms: AOC-E-400 (Executor's Oath), AOC-E-403 (Letters Testamentary)
→ Notes: Grants executor legal authority to act

[Continue for all steps with state-specific info]

DEADLINES:
• Step 11: 90 days from executor qualification
• Step 9: Creditor claim period: 90 days from first publication

RESOURCES:
• Official Forms: [forms_url]
• State Law: [statutes]
```

**Print Guide Button:**
- Generates clean HTML page (no app chrome)
- Printable one-pager
- Can be saved as PDF by attorney

---

## **SOFT DELETE SYSTEM FOR DOCUMENTS**

**Purpose:** Allow attorneys/executors to delete wrong uploads with 30-day recovery window.

**Soft Delete Logic:**
1. When document deleted:
   - Set documents.soft_deleted=true
   - Set documents.deleted_at=now
   - File remains on disk under UPLOAD_ROOT_DIR
   - Document no longer visible in UI
2. Checklist item status reverts to 'Not Started'
3. Background job runs daily:
- Implement as OS cron calling a Flask CLI command (no in-process scheduler):
  - Cron: `0 2 * * * /path/to/venv/bin/flask run_jobs`
  - `flask run_jobs` performs:
    - purge of soft-deleted documents older than 30 days (disk + DB)

**CLI stub:**
```python
import os
from pathlib import Path
from datetime import datetime, timedelta, timezone
import click
from flask.cli import with_appcontext

UPLOAD_ROOT_DIR = os.getenv("UPLOAD_ROOT_DIR", "./data/uploads")

@click.command("run_jobs")
@with_appcontext
def run_jobs():
    cutoff = datetime.now(timezone.utc) - timedelta(days=30)
    # DB query: documents where soft_deleted=true and deleted_at < cutoff
    # for each:
    #   abs_path = Path(UPLOAD_ROOT_DIR) / doc.file_path
    #   try unlink
    #   delete document row
    pass
```

   - Find documents where soft_deleted=true AND deleted_at < (now - 30 days)
   - Delete file from disk (unlink file_path)
   - Delete document record from database

**Implementation:**
- Soft-deleted documents excluded from all queries: `WHERE soft_deleted = false`
- Background job: OS cron invokes `flask run_jobs` daily

---

## **PDF EXPORT SYSTEM**

**Route:** `/attorney/case/<case_id>/export`

**Engine (LOCKED):** ReportLab (Python). No HTML-to-PDF.

**Gate:** Requires active attorney subscription (see Billing section).

**Generate a single PDF containing:**

**Page 1: Case Summary**
- Decedent: [decedent_name]
- Executor: [executor_name]
- Executor Email: [executor_email]
- Attorney Email: [attorney_email]
- State: [state]
- Case Status: [Open|Closed]
- Created: [created_at]
- Closed: [closed_at or '—']
- Summary counts: Not Started / Submitted / Reviewed

**Page 2+: Checklist Detail**
For steps 1–12, in order:
- Step number + name
- Status
- Last updated
- Attorney note (if present)
- Document status:
  - "Missing" if no active document
  - "Submitted" if active document exists (include file_name and uploaded_at)

**Output:**
- PDF is streamed as an attachment download
- Filename: `[Decedent Name] - Executor Status Tracker.pdf`

**ReportLab implementation (server-side):**
```python
from io import BytesIO
from reportlab.lib.pagesizes import LETTER
from reportlab.pdfgen import canvas
from flask import send_file

def build_case_pdf(case, checklist_rows):
    buf = BytesIO()
    c = canvas.Canvas(buf, pagesize=LETTER)
    width, height = LETTER

    y = height - 72
    c.setFont("Helvetica-Bold", 14)
    c.drawString(72, y, "Estate Executor Status Tracker — Case Export")
    y -= 24

    c.setFont("Helvetica", 11)
    c.drawString(72, y, f"Decedent: {case.decedent_name}")
    y -= 16
    c.drawString(72, y, f"Executor: {case.executor_name}  <{case.executor_email}>")
    y -= 16
    c.drawString(72, y, f"Attorney: <{case.attorney_email}>")
    y -= 16
    c.drawString(72, y, f"State: {case.state}    Status: {case.status}")
    y -= 24

    # Add counts + notes summary as needed...

    c.showPage()

    c.setFont("Helvetica-Bold", 12)
    c.drawString(72, height - 72, "Checklist")
    y = height - 96
    c.setFont("Helvetica", 10)

    for row in checklist_rows:
        line = f"{row.step_number}. {row.step_name} — {row.status} — {row.last_updated}"
        c.drawString(72, y, line[:110])
        y -= 14
        if y < 72:
            c.showPage()
            y = height - 72
            c.setFont("Helvetica", 10)

    c.save()
    buf.seek(0)
    return buf

def export_case_pdf(case, checklist_rows):
    pdf = build_case_pdf(case, checklist_rows)
    filename = f"{case.decedent_name} - Executor Status Tracker.pdf"
    return send_file(pdf, mimetype="application/pdf", as_attachment=True, download_name=filename)
```

---
