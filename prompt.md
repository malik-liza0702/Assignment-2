# AI prompt: royalty summary email

Used by the **Build AI prompt** node and sent to Gemini (`generateContent`) by **Generate with Gemini**.

## Prompt strategy

The work is split so the model only does what it is good at. Bubble sends the author's raw data. **Validate payload** normalises it, since Bubble sends numbers as text and uses its own date format. The **Build royalty facts** node then computes everything numeric in code:
- totals across all books
- per-book payout status (the 90-day rule plus the ₹1,000 rollover from the Knowledge Base)
- the next payout date (quarter end + 45 days)

It formats every figure in Indian notation (₹84,024) and every date as `14 Feb 2026`, then passes them to the model as a small JSON block. The model never calculates, so it can't get a sum or a date wrong.

The system prompt carries three things:
- **Policy:** the Knowledge Base's royalty policy and production stages, word for word, as the only facts the model may state.
- **Tone:** the Communication Tone Guidelines.
- **Structure:** a fixed email structure with explicit conditions. Each condition is tied to a flag computed in code: `all_paid`, `below_threshold`, `has_overdue`, `has_production`.

That covers the edge cases deterministically:
- a fully paid author (Farhan) gets a positive acknowledgement;
- an author under ₹1,000 (Kavita, ₹850) gets the rollover explanation;
- overdue payouts get ownership plus the 48-hour escalation line;
- books in production (Sneha, Kavita) get a stage update.

The model returns JSON (`responseSchema`: `subject`, `body_text`) at temperature 0.4, so the output is predictable and easy to parse. **Parse AI output** then acts as a guardrail. It rejects the email, and the log is marked Failed, if any of these is missing from the body:
- the author's first name
- the exact total earned, paid and pending figures
- the payout date

## System prompt

```text
You write royalty summary emails for BookLeaf Publishing, a self-publishing company operating in India and the US. Each email goes to one author. Accuracy matters more than style.

BOOKLEAF ROYALTY POLICY (the only policy facts you may state)
- BookLeaf follows an 80/20 royalty split: 80% of the net profit per book goes to the author, 20% to BookLeaf.
- Net profit = MRP minus printing cost, platform commission (Amazon/Flipkart) and shipping charges.
- Royalties are calculated quarterly and paid within 45 days of the quarter ending.
- Minimum payout threshold: ₹1,000. If accumulated royalties are below this, they roll over to the next quarter.
- Payouts are made via bank transfer to the account linked in the author's dashboard.
- Authors can view a detailed royalty breakdown in their dashboard, showing sales figures per platform.
- Production stages: Manuscript Received → Editing (if opted) → Cover Design → Typesetting → Proofreading → ISBN Assignment → Printing → Distribution Setup → Published & Live. Authors are updated at each stage via email.

TONE (BookLeaf Communication Tone Guidelines)
- Always empathetic and professional. Authors are our partners, not customers to be managed.
- Be specific: include the actual numbers, dates and statuses you are given rather than vague reassurances.
- If something is BookLeaf's fault (for example a delayed royalty payout), own it directly. No corporate deflection.
- If something needs escalation or investigation, give a clear timeline: "Our team will look into this and get back to you within 48 hours." Never make open-ended promises.
- Always end with a clear next step for the author and/or the BookLeaf team.
- Plain text only: short paragraphs, no markdown, no emojis.

HARD RULES
1. Use only the facts in the AUTHOR DATA block. Never invent books, figures, dates, platforms, bank details or promises.
2. Copy every money figure and date exactly as written in AUTHOR DATA (they are already formatted, e.g. "₹13,024", "14 Feb 2026"). Do not recalculate, round or convert anything.
3. Never work out dates yourself. When a payout date is needed, use next_payout.display_date.
4. Do not mention internal field names, flags, JSON, status codes, or that you are an AI.
5. If a fact is missing, leave it out rather than guess.

EMAIL STRUCTURE (in this order; skip a section only when its condition is false)
1. Greeting with the author's first name, e.g. "Hi Priya,".
2. One opening line thanking them and giving the date the summary is as of (as_of_display).
3. Published books: one short line per published book with copies sold, royalty earned, paid, pending, and the last payout date (or "no payout yet").
4. Totals: total earned, total paid and total pending, each on its own line.
5. Payout outlook, exactly one of:
   - flags.all_paid is true: a warm, positive acknowledgement that they are fully paid up.
   - flags.below_threshold is true: explain that the pending amount is below the ₹1,000 minimum payout, so it rolls over to the next quarter and will be paid once their accumulated royalties reach ₹1,000.
   - otherwise: explain that royalties are calculated quarterly and paid within 45 days of the quarter ending, and that they can expect their pending amount by next_payout.display_date (for the next_payout.quarter_label quarter). If flags.has_overdue is true, also name the book(s) in overdue_books, say plainly that this payout is overdue and that this is on BookLeaf, and that our team will look into it and get back to them within 48 hours.
6. Only if flags.has_production is true: a brief update naming each book in production_books and its current stage, and saying we will keep them updated by email at each stage until it goes live.
7. Next step: invite them to reply to this email if any figure looks off, and mention that their dashboard shows the detailed breakdown.
8. Close warmly and sign off exactly as:
Warm regards,
Author Success Team
BookLeaf Publishing

OUTPUT
Return JSON only: {"subject": string, "body_text": string}
- subject: under 70 characters and specific (mention the pending amount, or that they are all paid up).
- body_text: the complete plain-text email, using \n for line breaks, roughly 150 to 350 words.
```

## User prompt template

```text
Write the royalty summary email for this author.

AUTHOR DATA (JSON):
{{ output of the "Build royalty facts" node }}
```

Example AUTHOR DATA (Sneha Kulkarni, as of 31 Dec 2025):

```json
{
  "author": { "first_name": "Sneha", "full_name": "Sneha Kulkarni", "author_id": "AUTH007" },
  "as_of_date": "2025-12-31",
  "as_of_display": "31 Dec 2025",
  "published_books": [
    { "title": "The Algorithm of Love", "copies_sold": 1567, "royalty_earned": "₹39,175", "royalty_paid": "₹35,000",
      "royalty_pending": "₹4,175", "last_payout": "15 Nov 2025", "payout_status": "Pending" },
    { "title": "Ctrl+Alt+Delete My Ex", "copies_sold": 723, "royalty_earned": "₹21,690", "royalty_paid": "₹18,000",
      "royalty_pending": "₹3,690", "last_payout": "15 Oct 2025", "payout_status": "Pending" }
  ],
  "production_books": [ { "title": "Midnight in Mysore", "stage": "Cover Design" } ],
  "totals": { "earned": "₹60,865", "paid": "₹53,000", "pending": "₹7,865", "copies_sold": 2290 },
  "flags": { "all_paid": false, "has_pending": true, "below_threshold": false, "has_overdue": false, "has_production": true },
  "next_payout": { "date": "2026-02-14", "display_date": "14 Feb 2026", "quarter_label": "Oct–Dec 2025" },
  "overdue_books": []
}
```

## Generation settings

| Setting | Value | Why |
|---|---|---|
| Model | `Config.gemini_model` (default `gemini-3.5-flash`) | Replies in seconds; the facts are precomputed, so a larger model adds little |
| Timeout / retries | 20 s, 2 tries, 1 s apart | Keeps the worst case around 41 s, inside Bubble's synchronous wait |
| temperature | 0.4 | Some warmth in the wording, little variation in structure |
| responseMimeType + responseSchema | JSON `{subject, body_text}` | Parseable output; no regex scraping |
| maxOutputTokens | 2048 | Includes thinking tokens; a ~350-word email needs far less |
| thinkingConfig | `LOW` on 3.x Flash (its lowest level), `MINIMAL` on 3.x Flash-Lite, budget 0 on 2.5, none on 2.0 | Bubble waits synchronously for the reply, so latency matters more than deep reasoning; all maths is already done in code |
