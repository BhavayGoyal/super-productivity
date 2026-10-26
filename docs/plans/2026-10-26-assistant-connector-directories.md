# Assistant connector: from capture to planning

Oct 26, 2026 · rev. 1 · status: proposal

Make Super Productivity (SP) a strong answer to "connect a task manager to my AI
assistant", for people who care about privacy and time tracking. We control
quality, not placement: the platforms decide what gets featured in their
directories. So this is a product investment first. Directory submissions come
second.

It builds on, and in places revises, two plans that already shipped in v19.3.0
(Oct 23):

- [Local desktop MCP](2026-09-22-local-desktop-mcp.md) (#10251)
- [Agent task capture](2026-10-11-agent-task-capture.md) (#10759, #10785, #10829)

§5 lists every earlier decision this plan touches and whether it keeps or
reverses it.

**Decision (proposed).**

1. Make both endpoints directory-ready. That means tool annotations, server
   metadata, a privacy-policy section, a published `.mcpb` and MCP Registry
   entries (M1, M2).
2. Give the local endpoint the tools daily planning needs: complete, schedule,
   update, track and log time, and a time summary. They sit behind a new
   permission that is off by default, and there is still no delete (M3, M4).
3. Remote reading and editing for cloud assistants (claude.ai, ChatGPT) is
   designed here as a relay to the user's own running app (M5). It stays
   gated on evidence and is not built in this round.
4. A server-side headless client is rejected for this purpose.

---

## 1. Goal and what "done" means

- **Reach.** SP is listed in Claude's connector directory (Community label), in
  the official MCP Registry, and is submitted to ChatGPT's plugin directory.
- **Quality.** A fixed set of planning scenarios (§6.7) works end to end against
  Claude Code, Codex and Claude Desktop on the local endpoint, and against
  claude.ai and ChatGPT on the hosted one, for whatever tools it has.
- **Uptake, without telemetry.** The server already stores capture grants with
  a client name. Counting them per client is the only signal; nothing new is
  collected. This count decides the M5 gate.

## 2. Where we are (v19.3.0)

| Surface                                                                | Reaches                                                                                      | Tools                                                                                | Auth                                                                       | Account          |
| ---------------------------------------------------------------------- | -------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------ | -------------------------------------------------------------------------- | ---------------- |
| Hosted `/mcp` on SuperSync (`packages/super-sync-server/src/capture/`) | Every client, including claude.ai web and mobile, ChatGPT, and Claude Code cloud sessions    | `create_task` only. It is write-only; items are sealed and imported on the next sync | OAuth with a connect code (Claude, Claude Code, ChatGPT); `sp_cap_` tokens | SuperSync (beta) |
| Local `/mcp` on the desktop app (`electron/mcp/`), at `127.0.0.1:3876` | Clients on the same computer: Claude Code, Codex, Cursor, and Claude Desktop through `.mcpb` | `get_status`, `list_tasks`, `get_task`, `list_projects`, `list_tags`, `create_task`  | `sp_mcp_` key and three scopes                                             | None             |
| Local REST API (`electron/local-rest-api.ts`)                          | Scripts, Parallel Code                                                                       | Full task CRUD, start/stop tracking, time logging                                    | REST token, all or nothing                                                 | None             |

Known gaps in what shipped:

- **No annotations on the local tools.** The hosted `create_task` has
  `readOnlyHint` and `destructiveHint` but no `title`
  (`create-task.tool.ts:94`).
- **The `.mcpb` is not published anywhere.** Its manifest has no icon, and
  `pack.js` needs the root `node_modules` (`fflate`).
- **The SuperSync privacy policy** (`templates/privacy.template.html`) does not
  mention AI assistants. This was M4 of the capture plan, still open.
- **The hosted instance may be behind.** It answered a claude.ai session on
  Oct 26 with a `create_task` that had only `title` and `notes`, while v19.3.0
  ships `dueDate`, `dueTime` and `repeat`. Either the instance is older than
  #10829, or Claude cached the tool list.
- **No MAS entitlement.** Mac App Store builds cannot listen on the port; the
  entitlement was deferred to its own release (09-22 plan, §3).

Parallel Code (johannesjo/parallel-code, MIT, ~1.1k★) already closes the
execute-and-track loop locally through the REST API:

- It creates a matching SP task when a task is first focused.
- It sets SP's current task, so SP tracks the time.
- It keeps titles in sync and marks the task done on merge.

The plan step, where an assistant plans the day in SP, is what is missing.

## 3. Facts checked on Oct 26, 2026

The confidence labels follow `docs/market/README.md`: **read** means the
primary page was read; **snippet** means only a search excerpt was available.
OpenAI's and modelcontextprotocol.io's hosts were blocked from the research
sandbox. The registry docs were read from their GitHub source instead.

**Claude connector directory (read)**

- **Sources.** claude.com/docs/connectors/building/{submission,
  review-criteria, authentication, mcpb}.md, and the Software Directory Policy
  (support article 13145358).
- **Submission.** Through the portal at claude.ai/directory/manage. The
  submitter needs a paid plan. An automated policy scan lists the server as
  **Community** by default. Anthropic then automatically escalates "highly
  useful" listings to **Verified** review, where reviewers test each tool.
- **Required:**
  - Every tool has a `title` plus `readOnlyHint: true` (read tools) or
    `destructiveHint: true` (tools that "modify or delete data").
  - Read and write are separate tools. Tool names are at most 64 characters.
  - OAuth through DCR or CIMD with S256 PKCE. CIMD needs
    `client_id_metadata_document_supported` and `none` among the token auth
    methods.
  - A 401 carries `resource_metadata`.
  - Test credentials for a fully populated account.
  - Public docs by the publish date, a privacy-policy URL, a support contact
    and an icon.
  - At least three working example prompts (policy §3E).
  - The server domain should match the service.
- **Not accepted:** money transfer, and AI media generation.
- **Annotations drive auto-permissions.** Read-only tools run without asking;
  destructive tools always prompt.
- **Several URLs per listing are Verified-only.** A listing whose users connect
  to different URLs, such as self-hosted servers, needs the Verified label.
- **Desktop extension listings are deprecated.** "The directory no longer
  accepts MCPB submissions." A local server reaches the directory only inside
  a plugin. Plugin local servers load in Claude Code and local Cowork, not in
  Chat. A `.mcpb` is still installed by double-click.

**ChatGPT (snippet only)**

- **Rename.** The app directory became the ChatGPT/Codex **Plugin directory**
  on 2026-07-09.
- **Before submitting:**
  - Identity verification.
  - A public HTTPS server on your own domain.
  - A domain token at `/.well-known/openai-apps-challenge`.
  - A project with global data residency. EU-residency projects cannot submit
    MCP plugins.
- **Annotations.** All three hints as explicit booleans. `destructiveHint`
  is false only for additive writes, and `openWorldHint` is false for a private
  workspace.
- **Submission package:**
  - 5 positive and 3 negative test cases
  - a demo video
  - privacy, terms, support and website URLs
  - a test account without MFA
- **UI is optional.** A widget is not required, and tools-only servers are a
  documented path.
- **Plans (unverified).** Custom write connectors may be limited to Business,
  Enterprise and Edu. If so, a listing is the only route for consumer users.

**Official MCP Registry (read, GitHub `modelcontextprotocol/registry` docs)**

- **Status.** Preview; API frozen at v0.1. Published with the `mcp-publisher`
  CLI. Versions are immutable, and an entry is at most 4 KB.
- **Namespaces.** `io.github.<org>/*` through GitHub OAuth or GitHub Actions
  OIDC; an org namespace needs an org owner. A domain namespace is verified by
  a DNS TXT record or by `/.well-known/mcp-registry-auth`.
- **Remote servers.** A `remotes` entry with `streamable-http`. A URL belongs
  to one name.
- **`.mcpb` packages.** They must be hosted on GitHub or GitLab releases, the
  URL must contain "mcp", and `fileSha256` is required.
- **Downstream (snippet).** GitHub's MCP registry (VS Code `@mcp`) and Glama
  build on it. Docker's catalog takes a PR. Smithery lists servers by URL.
  PulseMCP has paused submissions.

**Competitors (Claude listings read; ChatGPT snippet)**

- **Claude directory, all "Anthropic verified"** and all read and write:
  Todoist (27 tools), TickTick (92), Linear, Notion and ClickUp (including
  timer start/stop and time entries). Asana is mostly read, plus
  `update_task`.
- **ChatGPT:** Notion, Todoist, TickTick, Asana, ClickUp, Linear and Trello.
- **Official servers outside both directories:** Sunsama, Any.do, Akiflow
  (snippet).
- **No official server:** Toggl, Clockify, Motion, Microsoft To Do, Things.

**So the shelf is not empty for todo apps.** It is empty for time trackers,
and for anything open-source, local-first or account-free.

## 4. Prior art

| App      | Read / write                          | Time tracking tools              | Delete        | Account / price                                 | Where data lives |
| -------- | ------------------------------------- | -------------------------------- | ------------- | ----------------------------------------------- | ---------------- |
| Todoist  | Both, 27 tools                        | None found                       | Yes (snippet) | Account; free tier                              | Vendor cloud     |
| TickTick | Both, 92 tools                        | Focus records (snippet)          | Yes (snippet) | Account; free tier                              | Vendor cloud     |
| ClickUp  | Both                                  | Start/stop timer, add time entry | Yes (snippet) | Account; Free capped at 100 calls/day (snippet) | Vendor cloud     |
| Asana    | Mostly read, `update_task`            | None                             | No (read)     | Account                                         | Vendor cloud     |
| Sunsama  | Official server, not listed (snippet) | Unknown                          | Unknown       | Paid account                                    | Vendor cloud     |

Sources: claude.com/marketplace/connectors pages (read, 2026-10), and vendor
pages and community lists (snippet, 2026-09/10). Nothing was tested hands-on.

**Take:**

- Separate tools per action.
- Time tracking as tools (ClickUp is the only listed precedent).
- "What did I work on" as a first-class read (no listed competitor answers it
  from tracked time).

**Deliberately don't:**

- **A 90-tool surface.** Every tool is permanent public contract and prompt
  surface; about 12 well-described tools cover the scenarios in §6.7.
- **Delete, archive and bulk tools.** The blast radius of a steered assistant
  outgrows the benefit; the user deletes in the app.
- **Project and tag creation.** That is organization churn, not planning.
- **Any vendor-cloud copy of the user's data.**

Scorecard rows: dimension 1 (quick capture, must not lose badly), 7 (time
tracking, must be best), 15 (extensibility, tracked), 19 (privacy, must be
best) and 21 (AI, open decision). This plan serves the must-be-best rows 7
and 19. It does not try to close the AI row by building an assistant.

## 5. What earlier plans decided, and what changes

| #   | Earlier decision (source)                                                                                                    | This plan                                                                                                                                                                                                   | Why                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| --- | ---------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 1   | "Editing is out of scope until usage shows a need" (09-22, Decision)                                                         | **Partly reversed** for the local endpoint: a new `tasks:edit` scope covers complete, schedule, update, start/stop tracking and log time. It is off by default. There is no delete, archive or re-parenting | For: two community MCP servers already ship full CRUD on SP (the larger has 19 tools including time tracking, `community-plugins.json`); Parallel Code edits through REST; every listed competitor reads and writes; Verified review tests usefulness. Against: no SP issue asks for MCP editing (searched 2026-10-26), and v19.3.0 uptake is three days old. The scope stays off by default, so a user who never opts in sees nothing new |
| 2   | Reads forward to the existing REST routes; capture uses its own internal route (09-22, §5)                                   | Edits use **internal `/assistant/*` routes** like capture, not REST `PATCH`                                                                                                                                 | REST `PATCH` sets `dueDay` through a generic `updateTask`. The app schedules with `planTasksForToday`, `PlannerActions.planTaskForDay`, `scheduleTaskWithTime` and `unscheduleTask` (`task-context-menu-inner.component.ts:800–850`). An assistant should do exactly what the menu does, as capture does with the add-task bar                                                                                                             |
| 3   | "Only active tasks … the archive is never exposed" (09-22, §Tools; wiki 3.01)                                                | **Kept.** The time summary (M4) gives totals per day and project for all tracked time, but titles only for active tasks                                                                                     | It answers "how did I spend this week?" without breaking the published promise; titles of archived tasks would need their own scope and a wiki change                                                                                                                                                                                                                                                                                      |
| 4   | No `instructions` in `initialize` (09-22, §Protocol)                                                                         | **Kept**                                                                                                                                                                                                    | Tool descriptions carry the domain rules (Inbox, Today = due today); one less public string to maintain                                                                                                                                                                                                                                                                                                                                    |
| 5   | `.mcpb` release asset and in-app download are follow-ups (09-22, §.mcpb, §8)                                                 | **Done now** (M2)                                                                                                                                                                                           | Claude Desktop users have no other route; the MCP Registry needs a GitHub release URL and a SHA-256                                                                                                                                                                                                                                                                                                                                        |
| 6   | MAS `network.server` entitlement in its own release, after a dev-build check (09-22, §3, §8)                                 | **Unchanged**                                                                                                                                                                                               | Still the condition for Mac App Store users. It stays a maintainer step and is not in this branch                                                                                                                                                                                                                                                                                                                                          |
| 7   | Hosted path only for capture (10-11, Decision 1)                                                                             | **Kept**                                                                                                                                                                                                    | —                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| 8   | "Out of scope: reading tasks remotely, editing …, server-created ops" (10-11)                                                | **Kept for this round.** M5 designs remote read/edit as a relay that runs on the user's device and is **gated**. The server still never creates ops and never stores task content                           | The relay is the only remote design that keeps the pitch true ("your data stays on your devices"); its cost (§6.5) needs uptake evidence first                                                                                                                                                                                                                                                                                             |
| 9   | The headless client "stays deferred" as the path to remote read (10-11, Decision)                                            | **Rejected** for directory use                                                                                                                                                                              | It needs the user's key and full plaintext state on the server, plus reducer extraction (~240 files); that would turn "private by design" into "trust our server"                                                                                                                                                                                                                                                                          |
| 10  | `create_task` annotations `readOnlyHint: false, destructiveHint: false` (10-11, §Tool contract)                              | Add `title`, `idempotentHint: false` and `openWorldHint: false`                                                                                                                                             | Claude needs `title`; ChatGPT needs all three hints explicitly                                                                                                                                                                                                                                                                                                                                                                             |
| 11  | M4: wiki per client, privacy-policy addition, release notes, enable on the hosted server (10-11, Milestones)                 | The privacy-policy section is written here; redeploying and enabling stay operator steps                                                                                                                    | Both directories need a privacy-policy URL that covers this                                                                                                                                                                                                                                                                                                                                                                                |
| 12  | Connect code and client allowlist instead of server sign-in (10-11, Decision 5)                                              | **Kept**, plus a reviewer runbook (§6.6)                                                                                                                                                                    | Server sign-in would need a login-then-continue flow the server lacks. The runbook gives reviewers a demo account and a running app that shows the code. Risk: ChatGPT asks for "a test account without MFA"                                                                                                                                                                                                                               |
| 13  | "No shared package until a second user exists" (10-11, §/mcp)                                                                | **Kept.** The two protocol handlers stay separate                                                                                                                                                           | The relay (M5) would be that second user                                                                                                                                                                                                                                                                                                                                                                                                   |
| 14  | Market page: "AI: built-in assistant, or only external assistants through MCP" is an open decision (`docs/market/README.md`) | **Assumes "external assistants through MCP"**                                                                                                                                                               | Proposed for the maintainer to record; only the maintainer edits that page                                                                                                                                                                                                                                                                                                                                                                 |
| 15  | Listing the local version as a Claude Desktop extension (raised in the Oct 26 discussion)                                    | **Dropped**                                                                                                                                                                                                 | Deprecated by Anthropic (§3). A Claude Code plugin is possible but is not in this round                                                                                                                                                                                                                                                                                                                                                    |

## 6. Design

### 6.1 Annotations and server metadata (M1)

Every tool gets a top-level `title` (protocol 2025-06-18 and later) and
`annotations` with `title` (2025-03-26 clients read only this), plus all four
hints as explicit booleans:

| Tool                          | title                       | readOnly | destructive | idempotent | openWorld |
| ----------------------------- | --------------------------- | -------- | ----------- | ---------- | --------- |
| `get_status`                  | Check connection            | true     | false       | true       | false     |
| `list_tasks`                  | List tasks                  | true     | false       | true       | false     |
| `get_task`                    | Get task                    | true     | false       | true       | false     |
| `list_projects`               | List projects               | true     | false       | true       | false     |
| `list_tags`                   | List tags                   | true     | false       | true       | false     |
| `get_time_summary` (M4)       | Summarize tracked time      | true     | false       | true       | false     |
| `create_task` (local, hosted) | Add task to Inbox           | false    | false       | false      | false     |
| `log_time` (M3)               | Log time on task            | false    | false       | false      | false     |
| `start_tracking` (M3)         | Start time tracking         | false    | false       | true       | false     |
| `stop_tracking` (M3)          | Stop time tracking          | false    | false       | true       | false     |
| `complete_task` (M3)          | Mark task done or not done  | false    | true        | true       | false     |
| `schedule_task` (M3)          | Schedule or unschedule task | false    | true        | true       | false     |
| `update_task` (M3)            | Update task                 | false    | true        | true       | false     |

- **Additive writes are not destructive.** That matches the MCP definition
  ("only additive updates") and ChatGPT's rule. Tools that overwrite existing
  values are destructive, so Claude always asks before running them.
- **`openWorldHint` is false** everywhere: the tools touch only the user's own
  workspace.

`serverInfo` gains `title: "Super Productivity"` and `websiteUrl` (2025-11-25
`Implementation`). No icons: the directories take their icon from the
submission form.

### 6.2 Distribution (M2)

- **`.mcpb`.**
  - `tools/mcpb/pack.js` becomes dependency-free, writing a stored zip with
    Node's `zlib` and CRC32. It then runs with a bare Node in the release job.
  - It stamps the manifest `version` with the app version, adds the app icon
    (`icon.png`, 512 px) and prints the SHA-256.
  - The `create-release` job uploads `super-productivity.mcpb` to the draft
    release.
- **In-app.** Under "Connect an assistant", one line links to the latest
  release asset: "Claude Desktop: install the extension". It links to
  `releases/latest/download/super-productivity.mcpb`, which redirects to the
  newest published release.
- **MCP Registry.**
  - Two entries, because the two servers have different tools, auth and
    reach:
    - `io.github.super-productivity/super-productivity` is the `.mcpb`
      package (local, full permissions model).
    - `io.github.super-productivity/supersync-capture` is the hosted
      `remotes` URL.
  - A `workflow_dispatch` workflow publishes both with GitHub Actions OIDC. It
    downloads a pinned `mcp-publisher`, verifies its checksum, then fills in the
    version and SHA-256 from the release.
  - Docker's catalog, Smithery and Glama follow by hand once the registry
    entries exist.
- **Privacy policy.** A new section in the SuperSync privacy template covers
  AI assistant connections:
  - what the assistant sends and what the server sees in memory
  - sealed storage, and deletion after import or 45 days
  - the grant metadata kept: client name, created, last used, expiry
  - revocation, and that the assistant's provider processes the data under its
    own terms

  It goes to the maintainer for legal review before the hosted server serves it.

### 6.3 Local planning tools (M3)

**New scope.** `tasks:edit` ("Change tasks: complete, schedule, rename,
estimate, move between projects and tags, and track or log time").

- `normalizeScopes` drops it without `tasks:read`, as it does for
  `tasks:read_notes`: an assistant cannot edit sensibly without ids from a read.
- Writing `notes` additionally needs `tasks:read_notes`. An assistant that
  cannot see notes must not overwrite them blindly.

**Tools.** Main validates arguments strictly, as for capture. The renderer
validates again.

- `complete_task {id, isDone? = true}`. Uses `TaskService.setDone` or
  `setUnDone`, like the checkbox. An unchanged state is a no-op success.
- `schedule_task {id, dueDate, dueTime?}`, or `{id, dueDate: null}` to clear.
  It follows the context menu exactly:
  - with a time: `scheduleTaskWithTime` and the default reminder option
  - today: `planTasksForToday`
  - another day: `PlannerActions.planTaskForDay`
  - `null`: `unscheduleTask`

  A day in the past is rejected.

- `update_task {id, title?, notes?, timeEstimate?, projectId?, tagIds?}`. At
  least one field is required.
  - Title, notes and estimate go into one `updateTask`, with
    `isIgnoreShortSyntax: true`.
  - `projectId` uses the same move as REST (`moveTaskToProject`, which carries
    repeat configs, #10675); subtasks are rejected.
  - `tagIds` uses `TaskService.updateTags`; subtasks are rejected, and unknown
    tag ids are rejected.
  - The Today tag id is rejected; Today is a due date (`schedule_task`).
- `start_tracking {id}` and `stop_tracking {}` use `setCurrentId`, like the
  play button.
  - A parent resolves to its first open subtask (`task.reducer.ts:290`). The
    result names the task actually tracked.
  - A done task is rejected (`TASK_DONE`), although the app would reopen it
    (`reopenStartedDoneTask$`). Without the reopen, the tool only moves the
    tracking pointer, plus the user's own "add worked-on tasks to Today"
    setting, which only schedules unscheduled tasks. That keeps it additive
    (`destructiveHint: false`).
- `log_time {id, date, minutes}` uses the REST time path (`addTimeSpentOnDay`):
  the date must be today or earlier, at most 24 h per day, and a parent task is
  rejected.

**Outcome discipline** is the capture service's:

1. `APP_BUSY` while remote ops apply.
2. `PERSIST_DEGRADED` if persistence already failed.
3. Dispatch, then flush.
4. `OUTCOME_UNKNOWN` if the flush throws or the persist flag flips.

An edit that ran is reported even if the grant was revoked meanwhile, as
capture does. The result echoes the task summary, never notes.

**Routing.** Internal routes `/assistant/tasks/:id/{complete,schedule,update,
track,time}` and `/assistant/tracking/stop`. They are accepted only with
`source: 'mcp'`, which an HTTP client cannot set. The new code lives in
`src/app/features/tasks/assistant-capture/`: a new `AssistantTaskEditService`
and its routes service. Nothing is added to `task.service.ts` or the REST
handler.

**List filter.** `list_tasks` gains `due: "overdue"`: open tasks whose
`dueDay`, or the day of `dueWithTime`, is before the logical today. It is
served by a new optional REST query `?due=overdue` (additive, documented),
because only the renderer knows the logical day.

### 6.4 Time summary (M4)

`get_time_summary {startDate, endDate}` covers at most 31 days and needs scope
`tasks:read`. It answers through an internal renderer route that reads
`timeSpentOnDay` of active and archived tasks and returns:

- `days: [{date, totalMs, byProject: [{projectId, ms}]}]`
- `tasks: [{id, title, projectId, ms}]`: active tasks only, the top 50 by time
  in range
- `untitledMs`: time on archived tasks, which are counted but not named

The response cap and projection stay in main, as for the other tools.

### 6.5 Remote planning through the user's device (M5, gated, not built now)

- **Shape.** The hosted `/mcp` keeps one URL. OAuth adds the scopes
  `tasks:read` and `tasks:edit` to `scopes_supported` and to the consent page.
  - For a tool other than `create_task`, the server picks an online device of
    that user that announced `assistantRelay` over the sync WebSocket
    (`sync/websocket.routes.ts`).
  - It sends `{type: 'assistant_request', id, tool, args, grantScopes}` and
    waits up to 20 s for `{id, result}` (at most 256 KB).
  - It returns the result and stores nothing.
- **Device.** Desktop only in v1. The renderer receives the message and passes
  it over IPC to main's existing `runTool`. The scopes applied are the
  intersection of the grant's scopes and this device's Assistant access scopes,
  so the user opts in per device and per assistant.
- **Offline.** `create_task` keeps the sealed queue. Other tools answer
  "Super Productivity is not running on a computer with assistant access on."
- **Privacy text.** "Your tasks stay on your devices. When your assistant asks,
  your own computer answers; the sync server passes the answer through without
  storing it. The assistant's provider sees what it reads." Storage stays E2EE.
  The server sees content in memory while relaying, exactly as it does for
  capture today.
- **Directory review** needs an always-on demo device: the desktop app on a
  small VM signed in to a demo SuperSync account.
- **Gate (all three):**
  1. The hosted capture grants show sustained use (the maintainer sets the
     number).
  2. The M3/M4 tools pass §6.7 locally.
  3. The capture connector holds a Claude Community listing.
- **Estimate:** 4–6 focused weeks, including E2E on SuperSync and a
  fresh-context review.
- **Rejected alternatives:**
  - Headless server client (§5, row 9).
  - A zero-knowledge tunnel that terminates TLS on the device. It needs
    per-user hostnames and certificates, and the assistant's vendor sees the
    plaintext anyway.

### 6.6 Directory submissions (M6, maintainer)

- **Claude, hosted capture connector, after M1 is deployed and the privacy
  section is live:**
  - Portal submission with docs (wiki 3.01 § Agent Task Capture), privacy URL,
    support (GitHub issues) and icon.
  - Three example prompts: "Add 'renew passport' to my Super Productivity
    Inbox for next Monday", "Remind me every weekday at 9:00 to review my
    plan", and "Save these three action items from this email as tasks".
  - The reviewer runbook: a demo SuperSync account, plus a running app that
    shows the connect code.
- **ChatGPT.** The same connector, after identity verification and the domain
  token. It needs 5 positive and 3 negative cases, a demo video (`e2e/store-video`
  tooling) and a global-residency project.
- **Both listings are capture-only until M5.** The listing text says so ("adds
  tasks; it can't read them").

### 6.7 Scenarios (the quality bar)

Run by hand per release against a fixture profile. Each scenario passes only
when it works first try with no follow-up prompt.

1. "What's on my plate today, and is it realistic for 6 hours?" (read +
   estimates)
2. "Move everything overdue to tomorrow." (`list_tasks due:overdue` +
   `schedule_task`)
3. "Add 'call Anna' for tomorrow at 15:00." (`create_task`)
4. "Mark the dentist task done." (`list_tasks query` + `complete_task`)
5. "Put the three inbox tasks about the website into project Website." (`update_task
projectId`)
6. "I spent 45 minutes on the report yesterday; log it." (`log_time`)
7. "Start tracking the invoice task." / "Stop." (`start_tracking` /
   `stop_tracking`)
8. "How did I spend my time last week, by project?" (`get_time_summary`)
9. Negative: "Delete all my tasks." The assistant has no tool for it and must
   say so.
10. Negative: with only `tasks:read`, "Mark X done". The tool is not listed, and
    the assistant explains that permission is missing.

## 7. Milestones

| #   | Deliverable                                                                                                                                                   | Verify                                                                                                                                                                                                                                                                                                                    | In this round |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------- |
| M1  | Annotations and titles on all local and hosted tools; `serverInfo.title` and `websiteUrl`                                                                     | Electron tests (`npm run test:electron`) and server tests assert annotations on every listed tool; the bridge end-to-end test still passes                                                                                                                                                                                | Yes           |
| M2  | Dependency-free `pack.js` with icon, version and SHA-256; release-job upload; in-app link; registry `server.json` files and publish workflow; privacy section | The packed file passes `unzip -t` and `mcpb validate` (run from a temp directory, no repo dependency); registry JSON validates against the published schema; the privacy-page test (`legal-pages.spec.ts`) still passes                                                                                                   | Yes           |
| M3  | `tasks:edit` scope, settings checkbox, six edit tools, internal routes, edit service, `due: overdue` filter, wiki 3.01 and 3.02                               | Electron tests for scope normalization, tool listing, validation, routing and revocation; renderer specs: one op per call (except `update_task` combining a move with field changes), the exact actions dispatched, the outcome codes; `npm run test:affected`; `checkFile`; assistant smoke script extended with an edit | Yes           |
| M4  | `get_time_summary` route and tool                                                                                                                             | Renderer spec with active and archived fixtures (no archived title leaks); electron tool tests for caps                                                                                                                                                                                                                   | Yes           |
| M5  | Relay                                                                                                                                                         | Gated (§6.5)                                                                                                                                                                                                                                                                                                              | No            |
| M6  | Submissions                                                                                                                                                   | Maintainer                                                                                                                                                                                                                                                                                                                | No            |

PR split: M1+M2 together (no app-state change), M3, then M4.

## 8. Long-term cost

- **Sync.** No synced field, no new action, no meta-reducer, no schema bump.
  Each edit tool dispatches the action its menu item dispatches. Remote ops
  never trigger these routes.
- **Device-local state.** The new scope string sits in main's simple store,
  which is not part of `MODEL_CONFIGS`, not synced and not in backups. An older
  app drops the unknown scope on load (`normalizeScopes`), so a downgrade fails
  closed.
- **Public contracts.** Tool names, arguments, annotations and result shapes
  can be added to, but not renamed or removed. The REST query `due=overdue` is
  additive. Registry versions are immutable.
- **Blast radius.** An assistant steered by a web page can now change tasks on
  this computer, once the user grants `tasks:edit`. Limits:
  - no delete, archive or bulk tools
  - writing notes needs notes read access
  - destructive tools prompt in Claude
  - every change is an ordinary op the user can see and revert in the app
- **Maintenance.**
  - Annotations are re-checked when a tool changes.
  - Directory rules are re-checked yearly (both changed in 2026).
  - The `.mcpb` follows the MCPB manifest version.
  - Until M5, tool logic lives only in the Electron main process.

## 9. Decisions for the maintainer

1. Accept the partial reversal of "no editing" (§5, row 1) for the local
   endpoint, behind `tasks:edit`, which is off by default.
2. Set the M5 gate: the number of active capture grants, and the review date.
3. Record "external assistants through MCP, no built-in assistant" on the
   market page.
4. Submit the capture connector to Claude's directory now (recommended) or wait
   for M5.
5. ChatGPT: verify as an individual or a business; create a project with
   global data residency.
6. Legal review of the privacy section before the hosted server serves it.

## 10. Out of scope

- Delete, archive and bulk tools.
- Subtask creation, and project or tag creation.
- Calendar events.
- A built-in assistant.
- Telemetry.
- Server-created ops.
- A headless server client.
- Relay devices other than desktop (in M5).
- A Claude Code plugin.
- The 2026-07-28 protocol revision (unchanged from 09-22).

## Sources

- Repository: `electron/mcp/*`, `electron/shared-with-frontend/assistant-access.model.ts`,
  `src/app/features/tasks/assistant-capture/*`,
  `src/app/core/electron/local-rest-api-handler.service.ts`,
  `src/app/features/tasks/task-context-menu/task-context-menu-inner/task-context-menu-inner.component.ts`,
  `packages/super-sync-server/src/capture/*`, `packages/super-sync-server/templates/privacy.template.html`,
  `tools/mcpb/*`, `.github/workflows/build.yml`, `docs/market/{README,scorecard}.md`.
- [Claude connector submission](https://claude.com/docs/connectors/building/submission.md),
  [review criteria](https://claude.com/docs/connectors/building/review-criteria.md),
  [authentication](https://claude.com/docs/connectors/building/authentication.md),
  [MCPB](https://claude.com/docs/connectors/building/mcpb.md),
  [Software Directory Policy](https://support.claude.com/en/articles/13145358-anthropic-software-directory-policy),
  [Todoist listing](https://claude.com/marketplace/connectors/todoist).
- [ChatGPT app/plugin submission](https://help.openai.com/en/articles/20001040-submitting-apps-to-the-chatgpt-app-directory),
  [app review](https://developers.openai.com/plugins/deploy/app-review),
  [plugin guidelines](https://developers.openai.com/plugins/plugin-guidelines) (snippet).
- [MCP Registry](https://github.com/modelcontextprotocol/registry) docs:
  `authentication.mdx`, `remote-servers.mdx`, `package-types.mdx`.
- [MCP tool annotations](https://modelcontextprotocol.io/specification/2025-11-25/server/tools)
  (spec source on GitHub).
- [Parallel Code](https://github.com/johannesjo/parallel-code) README.
