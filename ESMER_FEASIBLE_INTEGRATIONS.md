# Esmer (ESMO) - Feasible n8n App Node Integrations for Mobile

> Reference document mapping n8n app nodes to Esmer's mobile-first AI delegation assistant.
> Each node links to its n8n documentation within this repo. Nodes are categorized by
> Esmer use-case domain and tagged with an MVP phase recommendation.

---

## Selection Criteria

A node is included if it meets **all** of these:

1. **Has a public REST/OAuth API** that can be called from a cloud backend (Esmer's "Cloud Plane")
2. **Relevant to personal/professional productivity** — the core Esmer user who says "handle this", "plan my week", "follow up with them"
3. **Actions are delegatable** — the AI can meaningfully plan, draft, create, or read on the user's behalf
4. **Mobile-friendly outcome** — results can be displayed/approved in a mobile UI (not infrastructure-only tools)

Nodes excluded: raw databases (Postgres, MySQL, etc.), CI/CD pipelines (Jenkins, CircleCI, Travis CI), infrastructure (AWS IAM, ELB, Cloudflare, etc.), security-only (TheHive, MISP, Splunk), niche/deprecated services, and message queues (Kafka, RabbitMQ, AMQP).

---

## Phase Legend

| Phase | Meaning |
|-------|---------|
| **MVP-1** | Core delegation loop — works even before deep integrations |
| **MVP-2** | Becomes genuinely useful — email, calendar, tasks, notes |
| **MVP-3** | Feels like magic — broad integrations, rich artifacts |
| **Future** | Long-tail — add based on user demand |

---

## 1. Email & Communication

These are the backbone of Esmer's "handle this thread" and "draft replies" experiences.

| Node | Actions | Phase | Doc Path |
|------|---------|-------|----------|
| **Gmail** | Create/update/delete drafts, messages, labels, threads | **MVP-2** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.gmail/](docs/integrations/builtin/app-nodes/n8n-nodes-base.gmail/) |
| **Microsoft Outlook** | Create/update/delete folders, messages, drafts | **MVP-2** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.microsoftoutlook.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.microsoftoutlook.md) |
| **Slack** | Create/archive channels, send/delete messages, get users/files | **MVP-2** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.slack.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.slack.md) |
| **Microsoft Teams** | Create/delete channels, messages, tasks | **MVP-3** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.microsoftteams.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.microsoftteams.md) |
| **Telegram** | Send/edit/delete messages, get files | **MVP-3** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.telegram/](docs/integrations/builtin/app-nodes/n8n-nodes-base.telegram/) |
| **WhatsApp Business Cloud** | Send messages, upload/download/delete media | **MVP-3** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.whatsapp/](docs/integrations/builtin/app-nodes/n8n-nodes-base.whatsapp/) |
| **Discord** | Send messages, manage channels | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.discord/](docs/integrations/builtin/app-nodes/n8n-nodes-base.discord/) |
| **Google Chat** | Get memberships/spaces, create/delete messages | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.googlechat.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.googlechat.md) |
| **Mattermost** | Create/delete channels, post messages, add reactions | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.mattermost.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.mattermost.md) |
| **Matrix** | Send messages/media to rooms, get room info | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.matrix.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.matrix.md) |
| **Zulip** | Create/delete users/streams, send messages | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.zulip.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.zulip.md) |
| **Webex by Cisco** | Create/update/delete meetings and messages | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.ciscowebex.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.ciscowebex.md) |

---

## 2. Calendar & Scheduling

Core to "schedule a meeting", "plan my week", "avoid conflicts".

| Node | Actions | Phase | Doc Path |
|------|---------|-------|----------|
| **Google Calendar** | Add/retrieve/delete/update calendar events | **MVP-2** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.googlecalendar/](docs/integrations/builtin/app-nodes/n8n-nodes-base.googlecalendar/) |
| **Zoom** | Create/retrieve/delete/update meetings | **MVP-3** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.zoom.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.zoom.md) |
| **GoToWebinar** | Create/get/delete attendees, organizers, registrants | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.gotowebinar.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.gotowebinar.md) |

---

## 3. Tasks & Project Management

Central to "set up reminders & tasks", "turn this into action", delegation workflows.

| Node | Actions | Phase | Doc Path |
|------|---------|-------|----------|
| **Google Tasks** | Add/update/retrieve tasks | **MVP-2** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.googletasks.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.googletasks.md) |
| **Todoist** | Create/update/delete/get tasks | **MVP-2** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.todoist.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.todoist.md) |
| **Microsoft To Do** | Create/update/delete/get tasks, lists, linked resources | **MVP-2** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.microsofttodo.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.microsofttodo.md) |
| **Notion** | Get/search databases, create pages, get users | **MVP-2** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.notion/](docs/integrations/builtin/app-nodes/n8n-nodes-base.notion/) |
| **Asana** | CRUD users, tasks, projects, subtasks | **MVP-3** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.asana.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.asana.md) |
| **ClickUp** | CRUD folders, checklists, tags, comments, goals | **MVP-3** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.clickup.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.clickup.md) |
| **Linear** | CRUD issues | **MVP-3** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.linear.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.linear.md) |
| **Trello** | Create/update cards, add/remove members | **MVP-3** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.trello.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.trello.md) |
| **Jira Software** | CRUD issues, users | **MVP-3** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.jira.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.jira.md) |
| **monday.com** | Create boards, add/delete/get board items | **MVP-3** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.mondaycom.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.mondaycom.md) |
| **Coda** | Create/get/delete controls, formulas, tables, views | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.coda.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.coda.md) |
| **Taiga** | CRUD issues | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.taiga.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.taiga.md) |
| **Wekan** | CRUD boards and cards | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.wekan.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.wekan.md) |
| **Clockify** | CRUD tasks, time entries, projects, tags | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.clockify.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.clockify.md) |
| **HighLevel** | CRUD contacts, opportunities, tasks; book appointments | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.highlevel.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.highlevel.md) |

---

## 4. Notes, Documents & Knowledge

Powers "summarize this document", "turn this PDF into tasks", document creation.

| Node | Actions | Phase | Doc Path |
|------|---------|-------|----------|
| **Google Docs** | Create/update/get documents | **MVP-2** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.googledocs.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.googledocs.md) |
| **Google Sheets** | Create/update/delete/append/get spreadsheets | **MVP-2** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.googlesheets/](docs/integrations/builtin/app-nodes/n8n-nodes-base.googlesheets/) |
| **Microsoft Excel 365** | Add/retrieve table data, workbooks, worksheets | **MVP-3** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.microsoftexcel.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.microsoftexcel.md) |
| **Google Slides** | Create presentations, get pages | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.googleslides.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.googleslides.md) |
| **Airtable** | CRUD tables | **MVP-3** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.airtable/](docs/integrations/builtin/app-nodes/n8n-nodes-base.airtable/) |

---

## 5. File Storage & Cloud Drives

Needed for "save this", attachments, sharing files across runs.

| Node | Actions | Phase | Doc Path |
|------|---------|-------|----------|
| **Google Drive** | CRUD drives, files, folders | **MVP-2** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.googledrive/](docs/integrations/builtin/app-nodes/n8n-nodes-base.googledrive/) |
| **Microsoft OneDrive** | CRUD files, folders | **MVP-3** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.microsoftonedrive.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.microsoftonedrive.md) |
| **Microsoft SharePoint** | Upload/download/update files, manage list items | **MVP-3** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.microsoftsharepoint.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.microsoftsharepoint.md) |
| **Dropbox** | Create/download/move/copy files and folders | **MVP-3** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.dropbox.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.dropbox.md) |
| **Box** | Create/copy/delete/search/upload/download files and folders | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.box.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.box.md) |
| **Nextcloud** | CRUD files, folders; retrieve/invite users | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.nextcloud.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.nextcloud.md) |
| **AWS S3** | Create/delete buckets, copy/download files | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.awss3.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.awss3.md) |
| **Google Cloud Storage** | CRUD buckets, objects | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.googlecloudstorage.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.googlecloudstorage.md) |

---

## 6. Contacts & People

For "follow up with them", contact lookup, people management.

| Node | Actions | Phase | Doc Path |
|------|---------|-------|----------|
| **Google Contacts** | CRUD contacts | **MVP-2** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.googlecontacts.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.googlecontacts.md) |
| **HubSpot** | CRUD contacts, deals, lists, engagements, companies | **MVP-3** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.hubspot.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.hubspot.md) |
| **Pipedrive** | CRUD activity, files, notes, organizations, leads | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.pipedrive.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.pipedrive.md) |
| **Salesforce** | CRUD accounts, attachments, cases, leads; upload documents | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.salesforce.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.salesforce.md) |
| **Zoho CRM** | Create/delete accounts, contacts, deals | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.zohocrm.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.zohocrm.md) |
| **Freshworks CRM** | CRUD accounts, appointments, contacts, deals, notes | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.freshworkscrm.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.freshworkscrm.md) |
| **Intercom** | CRUD companies, leads, users | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.intercom.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.intercom.md) |
| **Copper** | CRUD companies, leads, projects, tasks | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.copper.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.copper.md) |
| **Affinity** | CRUD lists, entries, organizations, persons | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.affinity.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.affinity.md) |
| **Microsoft Dynamics CRM** | CRUD accounts | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.microsoftdynamicscrm.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.microsoftdynamicscrm.md) |

---

## 7. AI & Language Services

Powers Esmer's core intelligence: summarization, translation, analysis, image gen.

| Node | Actions | Phase | Doc Path |
|------|---------|-------|----------|
| **OpenAI** | Chat with models, create images, assistants | **MVP-1** | [docs/integrations/builtin/app-nodes/n8n-nodes-langchain.openai/](docs/integrations/builtin/app-nodes/n8n-nodes-langchain.openai/) |
| **Anthropic** | Analyze documents/images, generate/improve prompts | **MVP-1** | [docs/integrations/builtin/app-nodes/n8n-nodes-langchain.anthropic.md](docs/integrations/builtin/app-nodes/n8n-nodes-langchain.anthropic.md) |
| **Google Gemini** | Work with audio/video/images/docs; analyze, generate, transcribe | **MVP-2** | [docs/integrations/builtin/app-nodes/n8n-nodes-langchain.googlegemini.md](docs/integrations/builtin/app-nodes/n8n-nodes-langchain.googlegemini.md) |
| **Mistral AI** | Text extraction from various file types | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.mistralai.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.mistralai.md) |
| **Perplexity** | Chat/search with model | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-langchain.perplexity.md](docs/integrations/builtin/app-nodes/n8n-nodes-langchain.perplexity.md) |
| **DeepL** | Language translation | **MVP-3** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.deepl.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.deepl.md) |
| **Google Translate** | Language translation | **MVP-3** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.googletranslate.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.googletranslate.md) |
| **LingvaNex** | Language translation | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.lingvanex.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.lingvanex.md) |
| **AWS Comprehend** | Text analysis (sentiment, entities, language) | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.awscomprehend.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.awscomprehend.md) |
| **AWS Textract** | Analyze invoices/documents (OCR) | **MVP-3** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.awstextract.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.awstextract.md) |
| **AWS Rekognition** | Image analysis | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.awsrekognition.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.awsrekognition.md) |
| **AWS Transcribe** | Audio transcription | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.awstranscribe.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.awstranscribe.md) |
| **Google Cloud Natural Language** | Document analysis | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.googlecloudnaturallanguage.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.googlecloudnaturallanguage.md) |
| **Mindee** | Invoice prediction/OCR | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.mindee.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.mindee.md) |
| **Jina AI** | AI-powered search/processing | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.jinaai.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.jinaai.md) |

---

## 8. Developer & Code Tools

For "track this GitHub issue and ping me when it's unblocked".

| Node | Actions | Phase | Doc Path |
|------|---------|-------|----------|
| **GitHub** | CRUD files, repos, issues, releases, users | **MVP-3** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.github.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.github.md) |
| **GitLab** | CRUD issues, repos, releases, users | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.gitlab.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.gitlab.md) |
| **Sentry.io** | CRUD issues, projects, releases; get events | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.sentryio.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.sentryio.md) |
| **npm** | Package info | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.npm.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.npm.md) |

---

## 9. Email Marketing & Outreach

For automated follow-ups, campaigns, subscriber management.

| Node | Actions | Phase | Doc Path |
|------|---------|-------|----------|
| **SendGrid** | CRUD contacts/lists, send emails | **MVP-3** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.sendgrid.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.sendgrid.md) |
| **Mailchimp** | CRUD campaigns, get list groups | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.mailchimp.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.mailchimp.md) |
| **Brevo** | CRUD contacts/attributes, send emails | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.brevo.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.brevo.md) |
| **Mailgun** | Send emails | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.mailgun.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.mailgun.md) |
| **Mailjet** | Send emails, SMS | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.mailjet.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.mailjet.md) |
| **ConvertKit** | Create/delete custom fields, get tags, add subscribers | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.convertkit.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.convertkit.md) |
| **MailerLite** | CRUD subscribers | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.mailerlite.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.mailerlite.md) |
| **ActiveCampaign** | CRUD accounts, contacts, orders, lists, tags, deals | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.activecampaign.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.activecampaign.md) |
| **GetResponse** | CRUD contacts | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.getresponse.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.getresponse.md) |
| **Lemlist** | Get activities/teams/campaigns, CRUD leads | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.lemlist.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.lemlist.md) |
| **Customer.io** | Create/update customers, track events, get campaigns | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.customerio.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.customerio.md) |

---

## 10. SMS & Voice

For notifications, two-factor flows, urgent communication.

| Node | Actions | Phase | Doc Path |
|------|---------|-------|----------|
| **Twilio** | Send SMS, make calls | **MVP-3** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.twilio.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.twilio.md) |
| **MessageBird** | Send messages, get balances | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.messagebird.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.messagebird.md) |
| **Vonage** | Send SMS | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.vonage.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.vonage.md) |
| **Plivo** | Make calls, send SMS/MMS | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.plivo.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.plivo.md) |
| **Mocean** | Send SMS, voice messages | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.mocean.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.mocean.md) |

---

## 11. Social Media

For content posting, monitoring, personal brand management.

| Node | Actions | Phase | Doc Path |
|------|---------|-------|----------|
| **X (Twitter)** | Create DMs, delete/search/like/retweet tweets | **MVP-3** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.twitter.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.twitter.md) |
| **LinkedIn** | Post content | **MVP-3** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.linkedin.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.linkedin.md) |
| **Facebook Graph API** | GET/POST/DELETE across Facebook entities | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.facebookgraphapi.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.facebookgraphapi.md) |
| **YouTube** | Retrieve/update channels, create/delete playlists | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.youtube.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.youtube.md) |
| **Reddit** | Submit/get/delete posts, get comments, profiles | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.reddit.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.reddit.md) |
| **Medium** | Create posts, get publications | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.medium.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.medium.md) |

---

## 12. Payments & Finance

For invoicing, payment tracking, expense management.

| Node | Actions | Phase | Doc Path |
|------|---------|-------|----------|
| **Stripe** | Get balance, create charges/meter events, manage customers | **MVP-3** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.stripe.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.stripe.md) |
| **PayPal** | Create batch payouts, cancel unclaimed items | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.paypal.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.paypal.md) |
| **Wise** | Get profiles, exchange rates, recipients | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.wise.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.wise.md) |
| **QuickBooks Online** | CRUD bills, customers, employees, estimates, invoices | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.quickbooks.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.quickbooks.md) |
| **Xero** | Create/update/get contacts, invoices | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.xero.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.xero.md) |
| **Invoice Ninja** | CRUD clients, expenses, invoices, payments, quotes | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.invoiceninja.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.invoiceninja.md) |
| **Chargebee** | Create customers, return invoices, cancel subscriptions | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.chargebee.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.chargebee.md) |
| **Paddle** | CRUD coupons, get plans/products/users | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.paddle.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.paddle.md) |
| **Harvest** | CRUD clients, contacts, invoices, tasks, expenses | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.harvest.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.harvest.md) |

---

## 13. E-Commerce

For store management, order tracking, product updates.

| Node | Actions | Phase | Doc Path |
|------|---------|-------|----------|
| **Shopify** | CRUD orders, products | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.shopify.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.shopify.md) |
| **WooCommerce** | Create/delete customers, orders, products | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.woocommerce.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.woocommerce.md) |
| **Magento 2** | CRUD customers, invoices, orders, products | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.magento2.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.magento2.md) |

---

## 14. Support & Help Desk

For ticket management, customer follow-ups.

| Node | Actions | Phase | Doc Path |
|------|---------|-------|----------|
| **Zendesk** | Create/delete tickets, users, organizations | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.zendesk.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.zendesk.md) |
| **Freshdesk** | CRUD contacts, tickets | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.freshdesk.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.freshdesk.md) |
| **Help Scout** | CRUD conversations, customers | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.helpscout.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.helpscout.md) |
| **ServiceNow** | CRUD incidents, users, table records; get services | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.servicenow.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.servicenow.md) |

---

## 15. Notifications & Push

For "approval needed", "run completed" push notifications.

| Node | Actions | Phase | Doc Path |
|------|---------|-------|----------|
| **Pushbullet** | CRUD push notifications | **MVP-3** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.pushbullet.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.pushbullet.md) |
| **Pushover** | Send push notifications | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.pushover.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.pushover.md) |
| **Pushcut** | Trigger iOS shortcuts/notifications | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.pushcut.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.pushcut.md) |
| **Gotify** | Create/delete/get messages | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.gotify.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.gotify.md) |
| **AWS SNS** | Publish messages (push notifications) | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.awssns.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.awssns.md) |
| **AWS SES** | Send emails (transactional notifications) | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.awsses.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.awsses.md) |

---

## 16. CMS & Content

For blog management, content creation workflows.

| Node | Actions | Phase | Doc Path |
|------|---------|-------|----------|
| **WordPress** | Create/update/get posts, users | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.wordpress.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.wordpress.md) |
| **Ghost** | CRUD posts (Admin + Content API) | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.ghost.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.ghost.md) |
| **Webflow** | CRUD items | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.webflow.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.webflow.md) |
| **Contentful** | Get assets, content types, entries, locales | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.contentful.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.contentful.md) |
| **Storyblok** | Get/delete/publish stories | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.storyblok.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.storyblok.md) |
| **Strapi** | Create/delete entries | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.strapi.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.strapi.md) |

---

## 17. Bookmarks, Links & Web

For saving references, shortening links, web scraping.

| Node | Actions | Phase | Doc Path |
|------|---------|-------|----------|
| **Raindrop** | CRUD collections, bookmarks; get users, delete tags | **MVP-3** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.raindrop.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.raindrop.md) |
| **Bitly** | Create/get/update links | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.bitly.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.bitly.md) |
| **Airtop** | Cloud browser: query, scrape, interact with web pages | **MVP-3** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.airtop.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.airtop.md) |
| **Hacker News** | Get articles, users | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.hackernews.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.hackernews.md) |

---

## 18. Health & Fitness

For personal wellbeing tracking, "plan my week" with health context.

| Node | Actions | Phase | Doc Path |
|------|---------|-------|----------|
| **Strava** | Create activities, get activity info | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.strava.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.strava.md) |
| **Oura** | Get profiles, summaries | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.oura.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.oura.md) |

---

## 19. Smart Home & IoT

For device-plane integrations (Esmer MVP-3).

| Node | Actions | Phase | Doc Path |
|------|---------|-------|----------|
| **Home Assistant** | Get/create camera proxies, configs, logs, services, templates | **MVP-3** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.homeassistant.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.homeassistant.md) |
| **Philips Hue** | Delete/retrieve/update lights | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.philipshue.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.philipshue.md) |

---

## 20. Music & Media

| Node | Actions | Phase | Doc Path |
|------|---------|-------|----------|
| **Spotify** | Get album/artist info | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.spotify.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.spotify.md) |
| **Google Books** | Get bookshelves, volumes | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.googlebooks.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.googlebooks.md) |

---

## 21. HR & People Ops

| Node | Actions | Phase | Doc Path |
|------|---------|-------|----------|
| **BambooHR** | CRUD company reports, employee documents, files | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.bamboohr.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.bamboohr.md) |
| **Microsoft Entra ID** | CRUD users, groups; manage group membership | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.microsoftentra.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.microsoftentra.md) |

---

## 22. Analytics & Tracking

| Node | Actions | Phase | Doc Path |
|------|---------|-------|----------|
| **Google Analytics** | Get reports, user activities | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.googleanalytics.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.googleanalytics.md) |
| **PostHog** | Create events, identities, track pages | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.posthog.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.posthog.md) |
| **Segment** | Add users to groups, create identities, track activities | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.segment.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.segment.md) |
| **Google Ads** | Get campaigns | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.googleads.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.googleads.md) |

---

## 23. Shipping & Logistics

| Node | Actions | Phase | Doc Path |
|------|---------|-------|----------|
| **DHL** | Track shipments | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.dhl.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.dhl.md) |
| **Onfleet** | Create/delete tasks, get organization details | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.onfleet.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.onfleet.md) |

---

## 24. Utilities & Enrichment

| Node | Actions | Phase | Doc Path |
|------|---------|-------|----------|
| **Clearbit** | Autocomplete/lookup companies and persons | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.clearbit.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.clearbit.md) |
| **Brandfetch** | Get company information | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.brandfetch.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.brandfetch.md) |
| **Hunter** | Find/verify email addresses | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.hunter.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.hunter.md) |
| **OpenWeatherMap** | Get weather data | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.openweathermap.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.openweathermap.md) |
| **NASA** | Get imagery and data | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.nasa.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.nasa.md) |
| **CoinGecko** | Get coin/event data | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.coingecko.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.coingecko.md) |
| **Marketstack** | Get exchange, end-of-day data, tickers | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.marketstack.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.marketstack.md) |
| **QuickChart** | Generate bar/doughnut/line/pie/polar charts | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.quickchart.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.quickchart.md) |
| **APITemplate.io** | Get/create PDFs and images from templates | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.apitemplateio.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.apitemplateio.md) |
| **Bannerbear** | Create/get images from templates | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.bannerbear.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.bannerbear.md) |
| **Beeminder** | CRUD data points (goal tracking) | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.beeminder.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.beeminder.md) |

---

## 25. ERP & Business Management

| Node | Actions | Phase | Doc Path |
|------|---------|-------|----------|
| **ERPNext** | CRUD documents | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.erpnext.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.erpnext.md) |
| **Odoo** | CRUD contracts, resources, opportunities | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.odoo.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.odoo.md) |

---

## 26. No-Code / Low-Code Databases

For storing structured user data, Esmer skill state.

| Node | Actions | Phase | Doc Path |
|------|---------|-------|----------|
| **Supabase** | Create/delete/get rows | **MVP-3** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.supabase/](docs/integrations/builtin/app-nodes/n8n-nodes-base.supabase/) |
| **Baserow** | CRUD rows | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.baserow.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.baserow.md) |
| **NocoDB** | CRUD rows | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.nocodb.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.nocodb.md) |
| **SeaTable** | CRUD rows | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.seatable.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.seatable.md) |
| **Grist** | CRUD rows | **Future** | [docs/integrations/builtin/app-nodes/n8n-nodes-base.grist.md](docs/integrations/builtin/app-nodes/n8n-nodes-base.grist.md) |

---

## Summary by MVP Phase

### MVP-1 (2 nodes) — Core AI backbone
- OpenAI, Anthropic

### MVP-2 (12 nodes) — Becomes genuinely useful
- Gmail, Microsoft Outlook, Slack
- Google Calendar
- Google Tasks, Todoist, Microsoft To Do, Notion
- Google Docs, Google Sheets
- Google Drive, Google Contacts
- Google Gemini

### MVP-3 (28 nodes) — Feels like magic
- Microsoft Teams, Telegram, WhatsApp Business Cloud
- Zoom
- Asana, ClickUp, Linear, Trello, Jira, monday.com
- Microsoft Excel 365, Airtable
- Microsoft OneDrive, Microsoft SharePoint, Dropbox
- HubSpot
- DeepL, Google Translate, AWS Textract
- GitHub
- SendGrid
- Twilio
- X (Twitter), LinkedIn
- Stripe
- Pushbullet, Raindrop, Airtop
- Home Assistant, Supabase

### Future (80+ nodes) — Demand-driven expansion
Everything else in the tables above.

---

## Total: 122 feasible integrations out of 270 n8n app nodes

The remaining ~148 nodes were excluded because they are:
- Infrastructure/DevOps only (AWS IAM, ELB, Lambda, Cloudflare, etc.)
- Raw database connectors (Postgres, MySQL, CrateDB, TimescaleDB, etc.)
- CI/CD pipelines (Jenkins, CircleCI, Travis CI)
- Security-only tools (TheHive, MISP, Elastic Security, Splunk)
- Niche/deprecated services with minimal user overlap
- Training/demo nodes (n8n Training nodes)
- Server-side message queues (Kafka, RabbitMQ, AMQP, MQTT)
