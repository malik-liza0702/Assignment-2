# BookLeaf Author Royalty Dashboard

A royalty portal for BookLeaf authors and the BookLeaf team, built on **Bubble.io** with an **n8n** automation that writes personalised royalty summary emails using **Google Gemini**.

| | |
|---|---|
| App | https://malikliza2524.bubbleapps.io/version-test |
| Admin login | `admin@test.com` / `Admin@123` |
| Author login | `priya.sharma@email.com` / `Author@123` (Priya Sharma) |
| Demo video | https://drive.google.com/file/d/1ao9i7-6YveY4SgBdt10NQi2vjSu9_db3/view?usp=sharing |
| n8n workflow | `royalty-summary` JSON export (in this folder) |
| AI prompt | `prompt.md` (in this folder) |

The app runs on Bubble's free plan, which can't deploy to a live URL, so it is served from `version-test`.

---

## 1. What the app does

### Author portal
An author logs in and lands on their dashboard. They see:
- **Totals** across all their books: royalty earned, paid and pending.
- **My Books:** each book with its status, copies sold, and royalty earned, paid and pending, plus the last payout date.
- **A payout status badge:**
  - **green** = fully paid
  - **yellow** = pending
  - **red** = overdue

An author can only ever see their own books. This is enforced by Bubble privacy rules on the server, not just hidden in the page (see section 4).

### Admin portal
An admin logs in and lands on the **Authors** list.

- **Authors list:** every author with city, number of books, total pending and payout badge. It has:
  - a **name search**
  - a **city filter**
  - a **payout status filter**
  - **Clear filters**

  The filters combine, and any filter left empty is ignored.
- **Author detail (View):**
  - the author's profile and totals
  - a table of their books, each with its own royalty status badge (Paid up, Pending, Overdue, In production)
  - the history of summary emails sent to them
  - the **Generate & Send Royalty Summary** button
- **Generate & send for all eligible:** sends a summary to every author who has pending royalties (9 of the 10 authors), one after another.
- **Notification log:** every summary ever generated, newest first. Each row shows the author, a status badge (**Success**, **Pending** or **Failed**), the email subject or the error reason, and when it was sent.

Users who aren't logged in are sent back to the login page. Authors who try to open an admin page are redirected too.

---

## 2. How royalty status is worked out

Statuses are calculated against the dataset's snapshot date, **31 Dec 2025**.

**Per book:**

| Status | Rule |
|---|---|
| In production | The book is not yet "Published & Live" |
| Paid up | Nothing is pending |
| Overdue | Royalty is pending, the last payout (or the publication date, if never paid) is more than 90 days old, **and** the author's total pending is at least ₹1,000 |
| Pending | Royalty is pending but not overdue |

**Per author:** if any book is Overdue, the author is **Overdue**. Otherwise, if any book is Pending, the author is **Pending**. Otherwise they're **Fully paid**.

**The ₹1,000 rule** comes from BookLeaf's policy: amounts under ₹1,000 roll over to the next quarter. Kavita Deshmukh, for example, has ₹850 pending, so she's shown as Pending, not Overdue.

**Next payout date:** royalties are paid within 45 days of the quarter ending, so for the Oct–Dec 2025 quarter that's **14 Feb 2026**.

---

## 3. Data model

| Type | Fields |
|---|---|
| **User** | email, role (Admin / Author), author → Author |
| **Author** | author_id, name, email, phone, city, joined_date, book_count, total_earned, total_paid, total_pending, payout_status |
| **Book** | book_id, title, isbn, genre, status, publication_date, mrp, royalty_per_copy, copies_sold, royalty_earned, royalty_paid, royalty_pending, last_payout_date, print_partner, available_on, royalty_status, author → Author |
| **NotificationLog** | author → Author, status, request_id, triggered_by → User, email_subject, email_body, error_message, completed_at |

**Option sets:** Role, PayoutStatus, RoyaltyStatus, NotificationStatus (Pending / Success / Failed).

Each author **User** is linked to one **Author** record. That link is what the privacy rules use.

---

## 4. Privacy rules

| Data type | Admin | Author | Everyone else |
|---|---|---|---|
| Author | Everything | Only their own record (`This Author is Current User's author`) | Nothing |
| Book | Everything | Only their own books (`This Book's author is Current User's author`) | Nothing |
| NotificationLog | Everything | Nothing | Nothing |
| User | Can read roles | Only their own record | Nothing |

The rules apply to searches as well as to fields. So if an author types another author's page address, or another author's data would appear in a list, Bubble simply returns nothing.

---

## 5. Generating a royalty summary (Bubble → n8n → Gemini)

### What happens when the admin clicks **Generate & Send Royalty Summary**

1. **Bubble creates a log entry.** It adds a NotificationLog with status **Pending**, linked to the author and the admin who clicked. It gets a unique `request_id` made from the author ID and the current time.
2. **Bubble calls the n8n webhook.** It sends the author's details and all their books as JSON, with a shared secret in the `X-Webhook-Secret` header.
3. **n8n processes the request:**
   - validates the payload
   - checks it isn't a repeat request
   - calculates the royalty facts
   - asks Gemini to write the email
   - checks the email
   - replies to Bubble
4. **Bubble stores the reply** in the same log entry:
   - **Success:** saves the email subject and body.
   - **Failed:** saves the reason n8n gives.
   - **n8n unreachable, too slow, or the secret is rejected:** a separate step marks the log **Failed** with the reason, so a log is never left stuck on Pending.
   - **Duplicate:** n8n has already handled that request. Bubble deletes the extra log entry.

The admin sees the new row appear in the author's **Summary emails** table and in the **Notification log**.

### Why the call is synchronous
Bubble's free plan doesn't expose its Data API or Workflow API. That means n8n can neither read data from Bubble nor call back into it. So Bubble sends everything n8n needs in the request, and **n8n's reply is the callback**.

On a paid Bubble plan, the same design could become asynchronous: n8n would answer immediately, read the data through the Data API, and post the result to a Bubble backend workflow.

### Bulk send
**Generate & send for all eligible** puts every author with pending royalties into a queue (a list custom state on the admin page). A workflow runs **every 5 seconds**. When the queue isn't empty and nothing is in progress, it:
1. takes the first author;
2. runs **exactly the same send** as the single button (a shared custom event);
3. removes that author from the queue.

A `busy` flag makes sure only one send runs at a time. While the queue is running, the button shows **Sending…** and can't be clicked. Each author takes about 20–40 seconds, so the admin keeps the page open until the queue is empty.

The free plan has no backend workflows, which is why the queue runs in the browser.

---

## 6. The n8n workflow (`royalty-summary`)

```
Webhook (POST, header auth)
  → Config
  → Validate payload ──invalid──────────────┐
  → Replay check ──seen before──→ Respond: Duplicate
  → Build royalty facts                     │
  → Build AI prompt                         │
  → Generate with Gemini                    ├──→ Capture failed step → Format failure → Respond: Failed
  → Parse AI output (guardrail)             │
  → Send email (SMTP)                       │
  → Build success response → Respond: Success
```

| Node | What it does |
|---|---|
| **Webhook** | Receives Bubble's POST. **Header Auth** checks `X-Webhook-Secret`; a missing or wrong secret is rejected with 403 before anything runs. |
| **Config** | Central settings: Gemini model, snapshot date, sender and test inbox. |
| **Validate payload** | Checks the request has a request_id, author ID and name, and that every book has a title, status, valid numbers and valid dates. It also normalises Bubble's formats: numbers sent as text, empty values, Bubble's date format and comma-separated lists. |
| **Replay check** | Remembers each `request_id` for 10 minutes. A repeat gets **Duplicate** and no second email. |
| **Build royalty facts** | Calculates everything numeric **in code**: totals, each book's status, the next payout date, and the flags for the email's sections (all paid, below ₹1,000, overdue, in production). It also formats money (₹13,024) and dates (14 Feb 2026). |
| **Build AI prompt** | Combines the system prompt (BookLeaf policy and tone rules) with the facts. |
| **Generate with Gemini** | Calls Gemini, which returns JSON with `subject` and `body_text`. 20-second timeout, 2 tries. |
| **Parse AI output** | **Guardrail:** rejects the email if it's missing the author's first name, the exact earned, paid and pending totals, or the payout date. |
| **Send email** | Sends the email over SMTP. The dataset's author addresses aren't real, so in the demo this node is switched off and the generated email is stored in the Bubble log instead (`email_sent: false`). |
| **Respond: Success / Failed / Duplicate** | Every outcome replies with the same fields: `status`, `email_subject`, `email_body`, `email_sent`, `error_message`, `failed_step`, `completed_at`, `request_id`, `log_id`. |

**Every failure inside n8n still answers HTTP 200** with `status: Failed`, the name of the step that failed and the error message. Bubble therefore always has something it can log. The only exception is a bad secret, which gets 403.

---

## 7. The AI email

The full prompt and its reasoning are in `prompt.md`. In short:

- **Gemini only writes the wording; it never calculates.** Every figure and date is computed in code and handed to the model already formatted. The model is told to copy them exactly.
- **Policy and tone come from BookLeaf's knowledge base,** and are the only facts the model may state:
  - the 80/20 split
  - quarterly payouts within 45 days
  - the ₹1,000 minimum
  - the production stages
- **Each author's situation decides which paragraphs appear:**
  - **Fully paid** (e.g. Farhan): a short, positive acknowledgement.
  - **Below ₹1,000** (e.g. Kavita): explains the rollover to next quarter.
  - **Overdue** (e.g. Rohit): owns the delay and promises a reply within 48 hours.
  - **Books in production** (e.g. Sneha): a stage update for those books.
- **Structured output:** the model must return JSON (`subject`, `body_text`), plain text with no markdown.
- **Guardrail:** if the name, totals or payout date are missing from the email, it's rejected and the log shows **Failed** with the reason, rather than sending a wrong email.

---

## 8. Security, error handling and idempotency

**Security**
- n8n accepts requests only with the correct `X-Webhook-Secret`.
- In Bubble, the secret is a **private** API Connector header and the call runs **server-side**, so it never reaches the browser.
- The workflow export contains no credentials.

**Error handling**

| What goes wrong | What the admin sees |
|---|---|
| Bad or incomplete data | Failed: "Request validation: missing …" |
| Gemini slow, down or returns bad output | Failed, with the step and the reason |
| Email missing the real figures | Failed: "AI email failed validation, missing: …" |
| n8n switched off, unreachable or slow | Failed: "n8n did not respond: …" |

**Idempotency** (no double emails)
- **In Bubble:** the Generate button can't be clicked while that author already has a Pending log, so a double click creates only one entry.
- **In n8n:** a repeated `request_id` within 10 minutes is answered with **Duplicate** and no new email. Bubble then removes the extra log entry.
- **During bulk send:** the `busy` flag ensures one send at a time.

---

## 9. Trying it out

| Do this | Expect |
|---|---|
| Log in as Priya | Only her 2 books, ₹3,570 pending, yellow badge |
| Admin → filter city **Bangalore** | Only Sneha Kulkarni |
| Admin → filter **Fully paid** | Only Farhan Sheikh |
| Admin → search **"ka"** | Rohit Kapoor and Sneha Kulkarni |
| Generate for **Farhan** | Success: an "all paid up" email |
| Generate for **Kavita** | Success: explains the ₹850 rollover |
| Generate for **Rohit** | Success: owns the overdue payment, gives the 48-hour follow-up and the 14 Feb 2026 date |
| Generate for **Sneha** | Success: includes the book in production |
| Double-click Generate | One log entry |
| Generate & send for all eligible | One new log entry for each of the 9 eligible authors |
| Generate while the n8n workflow is off | Failed: "n8n did not respond…" |
