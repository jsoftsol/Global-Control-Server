# Global Control Server: Product Requirements Document (reverse-engineered)

This spec was reconstructed from the shipped backend code, not written before it. It describes what the system does today. Where behavior is inferred from names or structure, that is said. The web frontend is covered separately in [Global-Control-Frontend](https://github.com/jsoftsol/Global-Control-Frontend). Security internals, credentials and infrastructure details are left out on purpose.

## 1. Problem statement

Businesses and agencies keep contacts in one tool, send email from another, track deals in a third and wire them together by hand. Follow-up depends on someone remembering to do it.

Global Control Server is the engine behind a single platform for that work. It keeps contacts and their history in one place, fires automation when something happens to a contact, sends the messages, records what each contact did with them, and lets an agency run many client accounts under its own brand.

## 2. Target users of the API

| User | What they need from the server |
|---|---|
| Frontend app | A consistent API for every screen, plus live updates over sockets |
| Business owner or marketer | Reliable sending, correct segmentation by engagement, accurate reports |
| Agency | Client accounts, white-label branding, custom domains |
| Enterprise team member | Sub-user access to a parent account |
| External systems and AI agents | A tag and action API, and an API-key surface that mirrors the product |
| Third-party services | Webhook endpoints that report delivery, engagement and lead events |
| Contacts and leads (no login) | Public booking, chat bot, tracking and unsubscribe endpoints |

## 3. Scope

### 3.1 Implemented

- **Accounts.** Login, registration, password reset, enterprise login, platform-based login, JWT sessions with a stored session record, per-user API tokens, sub-users, client accounts, an admin area.
- **Contacts.** Create, import, search, filter by status and tag, bulk actions, export, history timeline, custom fields, smart lists, email validation.
- **Contact status.** Six states (new, active, inactive, passive, dead, undeliverable), one shared classification, a recorded transition each time the state changes, and a daily decay job.
- **Tags.** Groups, labels, actions, task tracking, firing by API, form submission and integration.
- **Workflows.** Nine step types, per-contact queues, timers through delayed jobs, conditional splits, goal tracking, freeze and release, sharing and import, live counts over sockets.
- **Broadcast email and newsletters.** Recipient queries, per-recipient jobs, tracking, unsubscribe, reports, delivery to Mailgun or custom SMTP.
- **Email events.** Webhook handling for Mailgun and SMTP providers, unmatched events held and re-applied on a schedule.
- **Pipelines and deals.** Stages, leads, deals, activities, histories, Gmail thread linking.
- **Appointments.** Availability, public booking, cancel, reschedule, reminders.
- **AI.** Chat bot builder and public chat on the OpenAI Assistants API, an email writing helper.
- **Integration engine.** Database-defined integrations, triggers and actions, OAuth and API-key accounts, sandboxed request execution.
- **Realtime.** Socket.IO with authenticated connections and per-user rooms.
- **Agency features.** White-label platforms, custom domains, SSL issuance with queued jobs.
- **Libraries.** Images, QR codes, rewards.
- **Satellite processing.** Heavy jobs run on a separate deployment of the same codebase, configured as a satellite through its environment file. The main server hands it work over HTTP through a dedicated set of job routes, and the satellite runs it through the queues.

### 3.2 Scaffolded or partial

- **Engagement campaigns.** The model and group exist, the route files define no routes. The working re-engagement behavior comes from status transitions and a re-engagement flag on broadcasts.
- **Zoom, Outlook calendar, PayPal.** Configuration fields on appointments, with no live integration code found.
- **SMS.** Delivered through the integration catalog and a number-listing helper, not a dedicated sending module.
- **Legacy workflow sequence models**, a backup workflow model and backup controllers remain in the tree.
- **MySQL layer.** Present only in a backup folder and not used.
- **Tag worker.** One queue worker is an empty stub.

### 3.3 Out of scope

- The user interface (separate frontend project)
- Billing and subscription handling (no payment flow found)
- Automated test suite
- Container images and CI

## 4. Key flows

### 4.1 A tag starts a workflow

1. A tag is fired by the app, a form, an external API call or another workflow.
2. The server removes any tags the tag says to remove, applies custom fields and adds the tag to the contact.
3. It records a tag task and processes the tag's actions. Integration and email actions go to the actions queue. Workflow actions add the contact to the workflow.
4. The workflow queue runs steps in order. Quick steps run immediately. A timer schedules a delayed job that resumes the contact later. Email, SMS and integration steps become action jobs.
5. Progress and counts are pushed to the web app over sockets.

### 4.2 A broadcast email goes out

1. The user picks recipients by engagement status, tags or a smart list, and writes the message.
2. The server builds the recipient query and hands the job to a satellite server.
3. The satellite creates a broadcast record, one conversation and one action job per contact, and queues them.
4. Workers send each message through Mailgun or the chosen SMTP config, with tracking and unsubscribe links added.
5. Provider webhooks report delivered, opened, clicked, failed, complained and unsubscribed. Each event updates the message record and the contact, and status transitions are recorded.

### 4.3 A contact's status changes

1. Delivery sets the last-sent time. An open or click sets the last-activity time.
2. The shared classifier turns those times into one of six states, using 30 and 60 day windows by default.
3. A transition row is written only when the state changes, so repeated events do not create noise.
4. A daily job records downward changes as the time windows run out.

### 4.4 A visitor books an appointment

1. The public booking endpoint returns available slots in the visitor's time zone.
2. The visitor registers, and a booking record and confirmation are created.
3. A delayed reminder job and a 30-minute cron send notifications.
4. The visitor can cancel or reschedule through public endpoints.

### 4.5 An agency adds a client

1. The agency creates a client account linked to its own.
2. The agency sets white-label branding on a platform record and can add a custom domain.
3. Domain checks and SSL issuance run as queued jobs.
4. The client signs in through the branded platform.

### 4.6 A new third-party service is added

1. An admin defines the integration, its triggers and actions, and its authentication type in the database.
2. Users connect an account through OAuth or an API key.
3. Tags and workflows can now call the service. The server runs the stored request definition in a sandbox.

## 5. Non-functional characteristics

- **Throughput.** Queues run with high concurrency, and heavy work can move to satellite instances. One webhook log collection holds tens of millions of records according to the project's own notes.
- **Reliability.** Unfinished tag and workflow tasks resume on restart. Webhook events that arrive early are held and retried.
- **Observability.** Queue and conversation records act as an audit trail. There is no structured logging or error tracking.
- **Deployment.** Node 20 under pm2. The main API runs on a production host with a dev instance beside it, and satellite deployments of the same code, set apart only by environment variables, handle queue work. Crons run only on the main production instance.
- **Testing.** None automated.

## 6. Risks

| Risk | Detail | Effect |
|---|---|---|
| Security review overdue | Secrets handling, authentication coverage, validation, rate limiting and webhook verification need an audit | Exposure of client and contact data |
| No tests or CI | Nothing catches regressions before a manual reload | Releases depend on manual checking |
| Repeated business rules | The contact status rule exists in several places, some drifted | Different screens can show different numbers |
| Very large files | Several controllers and libraries exceed 1,500 lines | Hard to change safely |
| Code stored in the database | Integrations carry executable snippets run in a sandbox package that is no longer maintained | Maintenance and safety concerns |
| Aging dependencies | Mongoose 6, AWS SDK v2, old JWT and upload libraries | Upgrade debt |
| Dead code in the tree | Backup folders, old models, unused packages | Confusion about what is live |
| Single main host | One server runs the API and all scheduled jobs, with satellite deployments for queue work | Downtime and scaling limits |
| Always-200 responses | Errors travel in the body | Weak monitoring and misleading client behavior |

## 7. Open questions

- Which legacy models, controllers and backup folders can be deleted?
- Is the engagement campaign feature still planned?
- Are Zoom, Outlook and PayPal meant to ship?
- What is the plan for moving off the unmaintained sandbox package?
