# Offer Accepted Onboarding

A skill for Claude that handles everything between a candidate saying **"yes"** and their **first day**.

Once installed, just tell Claude: **"Sam Taylor has accepted their offer"** and it will walk you through the rest, asking for your approval before anything is sent.

---

## What it does

1. **Pulls the new starter's details** from Ashby (role, start date, office, manager)
2. **Sends your offer accept email** using your Ashby template, and logs it in Ashby
3. **Nudges the manager on Slack**, halfway to the start date, to assign an onboarding buddy (with a ready-made intro email they can use)
4. **Books the first few days of meetings** around everyone's calendars, keeping set-up and lunch free
5. **Builds a first-day plan in Notion** and schedules it to the manager a week before the start date
6. **Schedules a "Your first day" email** to the new joiner for the Wednesday before they start
7. **Optionally kicks off an onboarding bootcamp** skill on day one

Nothing goes to another person without your clear yes in the chat.

---

## Before you start

You'll need:

- [Claude Desktop](https://claude.ai/download)
- An **Ashby** connection, for example the open-source [Ashby MCP](https://github.com/JakeTeagle/ashby-mcp)
- **Gmail**, **Google Calendar**, **Slack** and **Notion** connected in Claude

Don't use one of these? Open `SKILL.md` and delete or adapt that step.

---

## How to install

**Step 1:** Click the green **Code** button at the top of this page, then **Download ZIP**, and unzip it.

**Step 2:** Open `SKILL.md` in any text editor and fill in the **Setup** table near the top: your company name, email domain, signature, offices, Notion page and so on. Replace every `{{PLACEHOLDER}}`.

**Step 3:** In Claude Desktop, go to **Settings → Capabilities → Skills**, click **Upload skill**, and choose the folder (or a ZIP of it).

That's it. The skill is ready to use.

---

## How to use it

Open a new chat and type something like:

> *"Sam Taylor has accepted their offer, can you run onboarding?"*

Claude will pull the details, check them with you, and then go step by step, showing you every email, invite and message before it goes anywhere.

---

## Make it yours

Everything is plain English in `SKILL.md`. Common tweaks:

- **Change the timings**, for example send the buddy nudge two weeks out instead of halfway
- **Edit the message wording** to match your tone of voice
- **Add or remove sections** from the first-day plan
- **Add more offices**, each with its own arrival details

Save the file and re-upload it to apply your changes.
