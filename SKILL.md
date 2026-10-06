---
name: "offer-accepted-onboarding"
description: "Run when the user says a candidate has accepted their offer: sends the offer accept email, schedules a Slack nudge to the manager about an onboarding buddy, books first-day meetings, builds a first-day plan in Notion, schedules the first day email and (optionally) an onboarding bootcamp run."
---

# Offer Accepted Onboarding

Run this as soon as someone accepts an offer. It takes care of everything between "yes" and day one:

1. Pulls the new starter's details from Ashby
2. Emails them your offer accept template
3. Schedules a Slack nudge to their manager, halfway to the start date, about an onboarding buddy
4. Books their first few days of meetings, builds their first-day plan in Notion, and schedules it to the manager a week before they start
5. Schedules a "Your first day" email to the new joiner for the Wednesday before they start
6. Optionally schedules an onboarding bootcamp skill to run on day one

Anything that reaches another person (both emails, the calendar invites, both Slack messages, the Notion page link) gets shown to the user first. Nothing sends or gets scheduled without a clear yes in chat.

Writing style for anything you draft: {{WRITING_STYLE}}

---

## Setup: fill this in once for your company

Replace every `{{PLACEHOLDER}}` in this file with your own details before using the skill. The list below is everything you need.

| Placeholder | What it is | Example |
| --- | --- | --- |
| `{{COMPANY}}` | Your company name | Acme |
| `{{EMAIL_DOMAIN}}` | Your work email domain | acme.com |
| `{{RECRUITER_NAME}}` | Full name used to sign emails | Alex Morgan |
| `{{RECRUITER_FIRST_NAME}}` | First name used to sign emails | Alex |
| `{{RECRUITER_TITLE}}` | Job title in the email signature | Talent Lead |
| `{{RECRUITER_EMAIL}}` | Work email, cc'd on the first day email | alex@acme.com |
| `{{RECRUITER_PHONE}}` | Number new joiners can call or message | +44 (0)7700 900000 |
| `{{OFFER_ACCEPT_TEMPLATE}}` | Name of your Ashby email template | Default - Offer Accept |
| `{{OFFICES}}` | Each office, its timezone and arrival details (see Step 5) | London (Europe/London), New York (America/New_York) |
| `{{NOTION_PARENT_PAGE_ID}}` | The Notion page every first-day plan lives under | (a 32-character page ID) |
| `{{NOTION_REFERENCE_PAGE}}` | Link to an existing first-day plan to copy the format from | (a Notion URL) |
| `{{TOOL_ACCESS_INFO}}` | How new starters get access to tools and licences | "Post in the IT support Slack channel" |
| `{{COMPANY_MEETINGS}}` | Regular company-wide meetings to list on the plan | Monthly town hall, Friday all-hands at 2pm |
| `{{WRITING_STYLE}}` | How drafted messages should sound | Casual, British spelling, not over-enthusiastic, no em dashes |

**Tools this skill uses:** an Ashby connection (for example the open-source Ashby MCP), Gmail, Google Calendar, Slack, Notion, and Claude scheduled tasks. If you don't use one of these, delete or adapt that step.

---

## Step 1: Pull the starter's details from Ashby

If the user hasn't named the candidate, ask who's accepted.

1. Find them with `search_records_by_name` (entityType `candidate`), then `get_candidate`.
2. Use `filter_records` on `application` for that candidate to get the hired or offer-stage application, then `get_record_details` for:
   - the `application` (job title, hiring manager)
   - the latest `offer_version` (start date, location, and any manager field on the offer)
3. If the manager isn't on the offer, use the application's hiring manager. Use `get_record_details` with entityType `employee` to get the manager's name and email.

You need all of these:

- Candidate full name and first name
- Candidate's personal email (from the Ashby profile)
- Job title
- Start date
- Office location (one of `{{OFFICES}}`)
- Line manager's name, email and Slack member ID (find the Slack ID with `slack_search_users`)

Show the user the list and ask them to confirm or correct it:

```
Sam Taylor (sam.taylor@example.com)
Role: Account Executive
Starts: Mon 24 Aug 2026
Based: London
Manager: Jordan Lee (jordan@{{EMAIL_DOMAIN}}, Slack U0XXXXXXX)
```

Always get the user to confirm the **location** and the **manager**, even if Ashby looks clear. Location drives the timezone of every scheduled message. The manager receives two messages, so getting the wrong one is awkward.

Work out every date in Steps 3 to 6 with a quick Python calculation rather than in your head.

---

## Step 2: Send the offer accept email

1. Fetch the template fresh every run, since it may have been edited: `search_records_by_name` (entityType `communication_template`, query `{{OFFER_ACCEPT_TEMPLATE}}`), then `get_record_details` on its ID.
2. Fill it in:
   - Replace the candidate first name placeholder with their first name.
   - If the body ends with a signature placeholder token (a random-looking string), replace it with `{{RECRUITER_FIRST_NAME}}`.
   - Keep the subject exactly as the template has it.
   - Don't change any other wording.
3. Show the user the finished email and wait for a yes.
4. Send it from the user's Gmail to the candidate's personal email with `send_message`.
5. Log it in Ashby with `add_note_to_candidate`: "Offer accept email sent from Gmail on [date]." Emails sent from Gmail don't show up in Ashby's activity feed on their own, so this keeps the record straight.

---

## Step 3: Schedule the onboarding buddy nudge to the manager

**When:** halfway between today and the start date.

- Round to the nearest whole day. If that lands on a weekend, move it to the Friday before.
- Send at 10:00 in the manager's timezone (use the manager's Slack profile timezone).
- If the start date is under 4 days away, skip the halfway logic and send it now, after the user's yes.
- Slack can only schedule up to 120 days out. If the halfway date is further away, say so and set up a reminder for the user instead.

**How:** DM the manager using `slack_schedule_message`, with the manager's Slack member ID as `channel_id` and the time as a Unix timestamp.

**Message** (keep the wording as is unless the user changes it):

```
[First name] is starting on [Weekday D Month].
Can you assign an onboarding buddy and email them, checking in, and introducing them to their onboarding buddy please :)

email is: [candidate personal email]

Here's a draft you can use if it helps:

> Subject: Looking forward to having you on board
>
> Hi [First name],
>
> Hope you're well! Just wanted to drop you a quick note before you start on [weekday] to say we're really glad you're joining the team.
>
> I'd also like to introduce you to [buddy name], who'll be your onboarding buddy. They're the person to ask about anything in your first few weeks, whether that's where something lives in Notion or where to get a decent coffee near the office. [Buddy name], meet [First name]!
>
> If anything comes up before your first day, just give me a shout.
>
> See you soon,
> [Manager first name]
```

Show the user the message and the send time (with timezone) before scheduling. Once it's scheduled, tell them it can only be edited or cancelled from "Drafts and sent" in Slack.

---

## Step 4: Book the first few days and build the Notion plan

### 4a. Ask who they should meet

Ask the user who the starter should meet over their first few days. For each one you only need:

- **who** it's with
- **what** it's about

The user can also give a duration, a day or time, or say it's remote. If they don't:

- **Duration:** 30 minutes
- **Day and time:** you find it (4b)
- **Format:** in person if the colleague is in the same office as the starter, otherwise Google Meet

If `{{COMPANY_MEETINGS}}` includes a recurring meeting such as a town hall, look up the next one after the start date with Google Calendar `search_events`. If you can't find it, ask the user for the date.

### 4b. Find times

1. Resolve each colleague's work email with `slack_search_users`. If Slack doesn't have it, use the firstname@{{EMAIL_DOMAIN}} pattern and say which ones you guessed.
2. The new starter's address is assumed to be **{firstname}@{{EMAIL_DOMAIN}}** (lowercase). If the user gives a different one, use that.
3. Check each colleague's diary over the starter's first few days with `suggest_time` or `list_events`. The starter's own calendar won't exist yet, so only check the colleagues.
4. Times are in the starter's local time. Keep these slots free:
   - 10:00 to 10:30 on day one, for laptop and account set-up
   - 12:30 to 13:30 every day, for lunch
5. Put the most important or senior meetings on day one, and spread the rest over days two and three rather than cramming everything in.
6. If a colleague has no free slot in the first three days, tell the user and offer the closest options. Don't double-book anyone.

### 4c. Create the invites

Show the user the full list and wait for a yes:

```
Mon 24 Aug, 11:30 to 12:00 - Sam Taylor <> Jordan Lee - Team intro (in person)
Mon 24 Aug, 14:00 to 14:30 - Sam Taylor <> Priya Shah - Product walkthrough (Meet)
```

Once they approve, create each one on the user's primary calendar with `create_event`:

- **summary:** `{New hire full name} <> {Colleague full name} - {what it's about}`
- **attendees:** the new starter's {firstname}@{{EMAIL_DOMAIN}} and the colleague's email
- **duration:** 30 minutes unless the user said otherwise
- **timeZone:** the starter's office timezone
- **Meet link:** `addGoogleMeetUrl: true` for remote meetings only

Confirm back to the user with the final times.

### 4d. Create the Notion page

Create a **new** page. Don't duplicate an old one. Follow the format of the reference page at `{{NOTION_REFERENCE_PAGE}}`, fetching it each run with `notion-fetch` so any changes to it carry through. Before writing content, read `notion://docs/enhanced-markdown-spec` once so the callout and divider syntax is right.

- **Parent:** page ID `{{NOTION_PARENT_PAGE_ID}}`. Every first-day plan lives there. If it can't be found, ask the user rather than creating a new one.
- **Title:** `[Full name] - First Day Plan`
- **Icon:** 👋

Use these sections, in this order:

1. **Intro line:** "Welcome to {{COMPANY}}, [First name]! This page is your guide for day one..."
2. **Getting set up (10:00):** laptop and account set-up steps, copied from the reference page.
3. **Confirmed schedule:** a 📅 callout, "booked in your calendar", with each meeting from 4c grouped by day, showing the time, who it's with, what it's about and *(Meet)* where remote. Add lunch at 12:30 to 13:30 each day. If there's an onboarding bootcamp, add a line saying those sessions will land in their calendar on day one.
4. **Getting access to tools & licences:** {{TOOL_ACCESS_INFO}}
5. **Company-wide meetings:** {{COMPANY_MEETINGS}}, with the date from 4a. If the starter is in a different timezone from the meeting, add their local time.
6. **Your 30 / 60 / 90 plan:** copied from the reference page.
7. **Sign-off:** "Have a brilliant first day." Don't carry over one-off lines from the reference page.

Show the user the page link and ask if they want any changes.

### 4e. Make sure the manager can open it

If the parent page is private, the new page is too. Before the link goes anywhere, ask the user to share it with the manager and confirm they've done it.

### 4f. Schedule it to the manager

**When:** 7 days before the start date, at 10:00 in the manager's timezone. If that's already passed, send it now, after the user's yes. If it lands on a weekend, move it to the Friday before.

**How:** `slack_schedule_message` to the manager's Slack member ID.

**Message:**

```
Hey! Here's [First name]'s first day plan for [Weekday D Month]: [Notion link]
Have a look and let me know if you want anything changed before they start :)
```

Show the user the message and send time before scheduling.

---

## Step 5: Schedule the "Your first day" email

Every new joiner gets a short email before they start, telling them when and where to arrive.

**When:** the Wednesday before their start date (the latest Wednesday that falls before it), at 10:00 in the starter's local time. If that time has already passed, send it straight away once the user approves.

**From:** the user's Gmail. **To:** the starter's personal email.

**Cc:** the line manager and {{RECRUITER_EMAIL}}. If an office has a named person who greets new starters, cc them too (only list each person once).

**Subject:** `Your first day with {{COMPANY}}`

Write one version per office in `{{OFFICES}}`. Each office version needs: the address, the arrival time, who to ask for at reception, and any set-up guide link. Template:

```
Hi [First name]

We're really looking forward to having you join us on [weekday]!

Please come into our [office] office at [office address] for [arrival time]. [Optional: ask for [greeter name] at reception and they'll bring you up.]

[Optional: Here are some instructions to help get you set up: [set-up guide link]]

If you have any questions or problems, please do not hesitate to email me or message me on {{RECRUITER_PHONE}}.

Many thanks,

{{RECRUITER_NAME}}
{{RECRUITER_TITLE}}
{{COMPANY}} | {{RECRUITER_PHONE}}
```

### How to schedule it

Gmail has no scheduled send through the connector, so use a one-off scheduled task.

1. Show the user the filled-in email (to, cc, subject, body) and the send time in both their time and the starter's. Wait for a yes.
2. Create the task with `create_trigger`, `run_once_at` set to the Wednesday at 10:00 starter's local time, converted to UTC with Python.
   - **Name:** `First day email - [Full name]`
   - **Prompt:**

```
Send this email from the user's Gmail with send_message. The user approved this exact wording on [today's date].

Before sending, check in Ashby that [Full name]'s application is still in the Hired or Offer stage. If it isn't, don't send. Tell the user instead.

To: [personal email]
Cc: [cc list]
Subject: Your first day with {{COMPANY}}

[full email body]

Then add a note to the candidate in Ashby with add_note_to_candidate: "First day email sent from Gmail on [date]."
```

3. Tell the user which approval setting the task got. If its runs will ask before acting, the email will sit waiting for them, so suggest switching the task to "Automatically approve" in its settings.

---

## Step 6 (optional): Schedule an onboarding bootcamp run for day one

Skip this step if you don't have a separate onboarding bootcamp skill.

If you do, create a one-off scheduled task (`create_trigger` with `run_once_at`) for **10:30 on the start date in the starter's local time**, converted to UTC with Python so daylight saving is handled properly.

**Before creating it,** check `list_triggers` for an existing bootcamp task on the same date. If there is one, update its prompt (`update_trigger`) to add this starter rather than creating a second task, so the cohort gets booked together.

**Name:** `Onboarding bootcamp - [date]`

**Prompt** (it runs in a fresh session, so it must stand on its own):

```
Run the onboarding bootcamp skill for these new joiners, who start today:

- [Full name], [job title], based in [office], work email [firstname]@{{EMAIL_DOMAIN}} (check against Slack)

Book all sessions between this afternoon and Friday. Today is their first day, so don't book anything before 14:00 today. If they won't fit, tell the user.

Follow every approval step in the skill. Don't send or create anything without the user's yes.
```

Tell the user when the task will run, in both their time and the starter's if they differ, and which approval setting the task got.

---

## Order of play and approvals

1. Pull details from Ashby → **user confirms** (especially location and manager)
2. Fill in the offer accept email → **user approves** → send from Gmail → note in Ashby
3. Buddy nudge to the manager, halfway to the start date → **user approves** → schedule
4. Ask who they're meeting and what about → find times → **user approves** → send the invites → build the Notion page → **user reviews** → user shares it → schedule the link to the manager 7 days out → **user approves**
5. Fill in the first day email for their office → **user approves** → schedule it for 10:00 local on the Wednesday before they start
6. (Optional) Schedule the bootcamp run for 10:30 local on day one, or add them to an existing task

Finish with a short summary of what's been sent, what's booked, what's scheduled and when, plus the Notion link.
