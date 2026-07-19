# Default Prompt Library — Outlook AI Agent (MCP)

**Purpose:** Ready-to-use prompts for LibreChat's Prompt Library, built around the capabilities of the `outlook-ai-agent-mcp` server (Email, Calendar, Attachments, Drafts, Follow-ups, MOM, To-Do, Planner, Charts).

**How to use:** Import each row below as a separate Prompt in LibreChat's Prompt Library (Title → Prompt Name, Prompt → Prompt Text). Users can invoke them directly or customize placeholders like `[sender]`, `[date range]`, `[project name]` before running.

---

## 1. Email Management

| S.No | Prompt Title | Prompt |
| ---- | ------------ | ------ |
| 1 | Unread Email Summary | Search my inbox for all unread emails from the last [24 hours / 3 days]. For each email, summarize the sender, subject, and a 1–2 sentence summary of the key ask or information. Group the results by priority: (1) Needs immediate response, (2) FYI only, (3) Can be archived. Do not fabricate any sender, subject, or content — use only what is retrieved from the mailbox. |
| 2 | Sender-Specific Digest | Find all emails received from [sender name/email] in the last [7/14/30] days. Summarize the conversation thread chronologically, highlighting any open questions, commitments made, or deadlines mentioned. Flag if any email from this sender is still unread. |
| 3 | Priority Inbox Triage | Scan my inbox from the last [X days] and classify every email into: Urgent (needs response today), Important (needs response this week), and Low Priority (no action needed). Present as a table with Sender, Subject, Received Date, and Category. Base classification only on retrieved email content — do not guess intent beyond what is stated. |
| 4 | Search Emails by Keyword | Search my mailbox for all emails containing the keyword or phrase "[keyword]" within the date range [start date] to [end date]. List the results with Sender, Subject, Date, and a short snippet of the matching content. |
| 5 | Flag Important Emails | Review my unread emails from the last [X days] and flag the ones that mention deadlines, approvals needed, or explicit action requests. Provide a short reason for each flag. |
| 6 | Mark Newsletters as Read | Identify all emails in my inbox from known newsletter/notification senders (e.g., subscriptions, automated alerts) received in the last [X days], and mark them as read. List what was marked as read for my confirmation before finalizing. |
| 7 | Folder Cleanup Assistant | Review emails older than [X days] in my inbox that appear to be informational only (no action required). Suggest which folder each should be moved to based on subject/sender, and list the recommended moves in a table before taking any action. |
| 8 | Semantic Email Search | Using semantic search, find emails related to the topic "[topic/project description]" even if they don't contain the exact keyword. Return the top matches with Sender, Subject, Date, and relevance summary. |

## 2. Drafting & Replies

| S.No | Prompt Title | Prompt |
| ---- | ------------ | ------ |
| 9 | Draft Professional Reply | Draft a professional reply to the email from [sender] with subject "[subject]". The reply should [acknowledge receipt / confirm attendance / request more information / provide an update on X]. Keep the tone formal and concise (under 120 words). Save the draft to Outlook but do not send it. |
| 10 | Draft Follow-Up Email | Compose a polite follow-up email to [sender/recipient] regarding "[topic]" that was originally sent on [date] and has not yet received a reply. Reference the original ask briefly, keep it under 100 words, and save it as a draft for my review. |
| 11 | Draft Meeting Invite | Draft a meeting invite titled "[meeting title]" for [date/time or "the earliest common availability"] with attendees [names/emails]. Include a brief agenda covering: [agenda points]. Set duration to [X minutes]. Save as a draft — do not send. |
| 12 | Bulk Reply Drafts for Thread | For the email thread with subject "[subject]", draft individual reply options for: (a) accepting the proposal, (b) requesting changes, (c) politely declining. Present all three drafts so I can choose which to send. |
| 13 | Out-of-Office Auto-Reply Draft | Draft an out-of-office style reply for emails I'm not able to respond to, mentioning I am unavailable from [start date] to [end date] and will respond after [return date]. Include an alternate contact: [name/email]. |

## 3. Calendar & Meetings

| S.No | Prompt Title | Prompt |
| ---- | ------------ | ------ |
| 14 | Today's Schedule Overview | Show my calendar for today. List each meeting with time, title, attendees, and location/link. Highlight any back-to-back meetings with less than 5 minutes buffer between them. |
| 15 | Weekly Calendar Summary | Summarize my calendar for the upcoming week ([Monday date] to [Friday date]). Group meetings by day, and flag any day with more than 4 hours of meetings as "heavy." |
| 16 | Find Common Availability | Check my calendar and find 3 mutually available 30-minute time slots between [start date] and [end date], during working hours [9 AM–6 PM]. Exclude any slots that conflict with existing meetings. Present the options as a simple list. |
| 17 | Meeting Conflict Check | Check whether scheduling a [duration]-minute meeting on [date] at [time] would conflict with my existing calendar. If there is a conflict, suggest the next 2 available alternative slots on the same day or the next business day. |
| 18 | Pre-Meeting Briefing | For my upcoming meeting titled "[meeting title]" on [date], pull the meeting details (attendees, agenda if available) and summarize any recent related email threads with the attendees so I can prepare. |

## 4. Minutes of Meeting (MOM) & Follow-ups

| S.No | Prompt Title | Prompt |
| ---- | ------------ | ------ |
| 19 | Generate MOM from Meeting | Generate Minutes of Meeting (MOM) for the meeting titled "[meeting title]" held on [date]. Structure the output as: Attendees, Key Discussion Points, Decisions Made, and Action Items (with owner and due date if mentioned). Use only information available from the meeting/email context — do not invent action items. |
| 20 | Extract Action Items | Review the meeting notes/email thread for "[meeting title/subject]" and extract all action items mentioned. Present as a table with Task, Owner, and Due Date (mark "Not specified" where absent). |
| 21 | Follow-Up Tracker | Check which emails I've sent in the last [X days] that requested a response but have not yet received a reply. List them with Recipient, Subject, Date Sent, and Days Pending. |
| 22 | Compose Follow-Up Batch | For all pending follow-ups identified from my sent items in the last [X days] with no reply, draft a short, polite follow-up message for each one. Present each draft separately with the original subject line referenced, for my review before sending. |
| 23 | Reply Status Check | Check whether [sender/recipient] has replied to my email with subject "[subject]" sent on [date]. If no reply has been received, tell me how many days it has been pending. |

## 5. Tasks & Planner

| S.No | Prompt Title | Prompt |
| ---- | ------------ | ------ |
| 24 | Create To-Do from Email | Read the email with subject "[subject]" from [sender] and create a Microsoft To-Do task summarizing the required action. Set the due date based on any deadline mentioned in the email, or ask me if none is specified. |
| 25 | Create Planner Task | Create a Microsoft Planner task titled "[task title]" in the plan "[plan name]" with description "[description]", assigned to [assignee], due on [due date]. Confirm the details back to me before finalizing. |
| 26 | List My Open Tasks | List all my open tasks from Microsoft To-Do and Planner, sorted by due date (soonest first). Highlight any that are overdue. |
| 27 | Convert Meeting Actions to Tasks | From the action items extracted in the MOM for "[meeting title]", create individual Microsoft To-Do (or Planner, if a team task) entries for each action item, using the owner and due date identified. Confirm the full list before creating. |
| 28 | Daily Task Digest | Summarize my open tasks due today and tomorrow across To-Do and Planner. Group by priority if available, and note any that are overdue. |

## 6. Documents & Attachments

| S.No | Prompt Title | Prompt |
| ---- | ------------ | ------ |
| 29 | Summarize Email Attachment | Find the email from [sender] with subject "[subject]" and summarize the key content of its attachment(s) in 5–7 bullet points. Note the file type and name of each attachment summarized. |
| 30 | Extract Data from Attachment | Open the attachment named "[file name]" from the email with subject "[subject]" and extract [specific data — e.g., table of figures, key dates, action items]. Present the extracted data in a clean table. |
| 31 | Compare Two Attachments | Compare the content of attachment "[file 1]" (from email "[subject 1]") with attachment "[file 2]" (from email "[subject 2]"). Highlight key differences, especially in [specific focus, e.g., figures, dates, terms]. |

## 7. Search, Summarization & Reporting

| S.No | Prompt Title | Prompt |
| ---- | ------------ | ------ |
| 32 | Daily Briefing | Give me a daily briefing covering: (1) unread priority emails, (2) today's calendar with any conflicts, (3) tasks due today, and (4) any follow-ups pending beyond 3 days. Keep each section concise and use headers. |
| 33 | Project Status Summary | Search my mailbox and calendar for all activity related to the project "[project name]" in the last [X days]. Summarize key emails, meetings held, and outstanding action items into one status report. |
| 34 | Weekly Recap Report | Generate a weekly recap covering: emails sent/received on [topic/project], meetings attended, MOM highlights, and tasks completed vs. pending for the week of [date range]. |
| 35 | Chart My Task Status | Create a chart showing the breakdown of my current tasks by status (Not Started, In Progress, Completed) across To-Do and Planner. Use a pie chart. |
| 36 | Chart Email Volume Trend | Create a bar chart showing the number of emails received per day over the last 7 days, based on my inbox. |
| 37 | Reminder Digest | Review my calendar and tasks for the next [X days] and compile a reminders list of anything with an approaching deadline (meetings, task due dates, pending follow-ups), sorted by date. |

---

### Notes for Admin (LibreChat Prompt Library setup)
- Prompts use `[bracketed placeholders]` — LibreChat's Prompt Library supports variable substitution; convert brackets to `{{variable}}` syntax if using dynamic prompt variables.
- All prompts are designed to rely solely on data retrieved via the MCP tools (Outlook, Calendar, To-Do, Planner) and explicitly avoid encouraging fabricated content, consistent with the platform's output validation/governance rules.
- Prompts are grouped by category — consider mirroring these as folders/tags in the Prompt Library for discoverability (Email, Calendar, MOM/Follow-up, Tasks, Attachments, Reporting).
- Draft-related prompts (#9–13) always end in "save as draft" — none instruct auto-send, aligned with the platform's no-autonomous-send principle unless explicitly changed by admin policy.
