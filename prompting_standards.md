# Claude Prompting Standards

A compiled reference of current best practices for writing prompts across every Claude product. Hand this file to any agent — Claude or otherwise — so it can generate high-quality, standards-compliant prompts for the user to run, rather than the user writing them by hand.

**Maintained by** — daily scheduled task, "Claude & Claude Code best-practices watch"
**Last updated** — 2026-09-09
**Storage** — Claude.ai Artifact (canonical, interactive, always-current version): https://claude.ai/code/artifact/8b328789-7ea5-4055-84f6-ae75e1863ede — this file is a one-time portable markdown export of that artifact as of the date below, generated for handing to agents or tools that can't authenticate to claude.ai. It is a snapshot, not a live sync — re-export from the artifact URL above for the latest version.

---

## 0. How to use this file

If you are an agent that has been handed this file, and the user asks you to write, improve, or generate a prompt for a Claude product:

- Identify which product the prompt targets — Chat, Code, Cowork, Design, or API/Platform. Ask or infer from context if it's unclear.
- Apply the Universal Principles (§1) as a baseline, always.
- Layer on the product-specific section's techniques and patterns.
- Check the Model-Specific Notes (§6) if the user names a specific model — current-generation models respond differently to some older patterns (verification instructions, prefill, subagent delegation), and Claude Fable 5.1 in particular needs several explicit corrections to its default behavior — see §6.
- If community guidance (§9) conflicts with official docs, default to official guidance, and only surface the conflict if it's relevant to the user's case.
- Produce the actual prompt text the user can copy and run — not commentary about prompting.

## 1. Universal principles

From Anthropic's official prompt-engineering documentation. Hold regardless of which product surface is in use.

- **Be explicit and direct.** State exactly what you want — treat Claude like a capable new hire with no institutional context. Spell out what "done" looks like.
- **Give context and motivation**, not just instructions. Explaining *why* something matters lets Claude generalize correctly to edge cases you didn't enumerate.
- **Use examples.** Three to five, relevant and diverse, covering edge cases. Wrap them in `<example>`/`<examples>` tags in longer prompts.
- **Structure complex prompts with XML tags** — `<instructions>`, `<context>`, `<document>`, `<output_format>`. Nest tags when content has hierarchy.
- **Assign a role when it sharpens focus** — a short system-level framing ("you are a senior security engineer reviewing this diff") tightens tone and attention.
- **Say what to do, not just what to avoid.** "Write in flowing prose" beats "don't use bullet points."
- **Put long documents first**, above your instructions or question — this alone can lift accuracy meaningfully on long-context tasks. Wrap in `<document>` tags with source/metadata.
- **Allow uncertainty.** Explicitly permitting "say so if you're not sure" measurably reduces hallucination.
- **Match your prompt's formatting to your desired output.** Markdown-heavy prompts pull toward markdown-heavy answers.
- **Shortest reliable prompt wins.** Add structure only where it fixes an observed failure mode — don't over-engineer.

## 2. Claude Chat

Questions, brainstorming, short-to-medium exchanges. Not for producing files across apps (Cowork) or writing/running code across a repo (Code).

- Default to direct, conversational phrasing. §1's XML/example structuring earns its keep once a prompt gets genuinely complex — multi-part questions, style constraints, structured output.
- State desired output format explicitly (tables, JSON, tone). Classic prefill (seeding the start of Claude's reply) is being phased out as a technique for current-generation models called directly via the API — see §6 — and doesn't apply to claude.ai chat the same way regardless.
- Use Projects for recurring topics instead of re-explaining background every conversation.
- If the user actually wants a deliverable file (doc, sheet, deck) rather than a conversational answer, redirect to Cowork — see §4 for Anthropic's own framework for choosing a surface.

## 3. Claude Code

An agentic coding environment: reads files, runs commands, edits code, works autonomously. Prompting it well is mostly about managing context and giving it a way to verify its own work.

### Core practices

- Front-load a `CLAUDE.md` with project conventions, build/test commands, and gotchas — this is read at the start of every session and avoids re-teaching the same context repeatedly.
- Give it a verification loop: tests, a linter, a type-checker, or an explicit "run this and check the output" instruction. Claude Code does its best work when it can tell for itself whether it succeeded.
- For large or multi-part work, ask for a plan first (plan mode), then approve before it edits — cheaper to redirect a plan than to unwind a large diff.
- Delegate isolated sub-problems to subagents (the `Agent` tool / Task tool) rather than growing one context window indefinitely; hand each subagent a self-contained brief since it starts with no memory of the parent conversation. The `Agent` tool's model override now includes **Fable** alongside sonnet/opus/haiku — reach for it on a subagent doing long-horizon research, writing, or a task where Opus/Sonnet evaluations fall short, but correct for its default behavior first (see §6).
- Periodically run `/skill-doctor` (added v2.1.261) to see which loaded skills are going unused and what they're costing in context — trim skills that never fire rather than leaving them loaded "just in case."
- Before a large edit, check `/diff` (fullscreen diff panel, added v2.1.260) to review uncommitted changes as you go, and check `/cost`/the status line for likely causes of prompt-cache misses if a session feels slower or pricier than expected.

### Recently changed (verify still current before relying on specifics)

- **`/skill-doctor`** (v2.1.261, 2026-09-04/05) shows which loaded skills go unused and their context cost — the recommended way to audit skill bloat in a project or session.
- **Fullscreen diff panel** (v2.1.260, 2026-09-03/04), toggled with `/diff`, shows uncommitted changes without leaving the session. `/cost` and the status line now also surface likely causes of prompt-cache misses.
- **Larger inline output**: new `bashOutputMaxChars` and `taskOutputMaxChars` settings (v2.1.261) raise how much command/background-task output Claude receives inline before it's spilled to a file, up to 128K characters — useful for verbose test suites or build logs.
- **`--append-subagent-system-prompt-file`** (v2.1.261) lets a large, reusable subagent system prompt live in a file instead of being pasted inline each time.
- **Headless/unattended improvements**: `/reload-plugins` now works in headless sessions (v2.1.260); `--permission-prompts none` (v2.1.259) suppresses permission prompts for unattended headless hosts — pair with a scoped permission config rather than using it to skip permissioning entirely.
- **`managedMcpServers`** setting (v2.1.259) lets an org centrally define MCP servers rather than each user configuring them locally.
- **Output styles.** A built-in **"Concise"** style ships as of v2.1.237 (2026-08-20): Claude leads with results and skips preamble/narration while still working thoroughly. Select it under `/config → Output style` for automation-friendly, low-narration sessions. v2.1.238 also fixed custom/project/plugin output styles silently drifting back to the default voice mid-session — worth re-testing any custom style you rely on in long sessions.
- **`ANTHROPIC_DEFAULT_MODEL`** environment variable (v2.1.236, 2026-08-19) sets which model new sessions start on — useful for pinning a team or CI default without per-session `/model` picking. A manual `/model` pick still overrides it and persists across restarts.
- **Long-session memory:** v2.1.238 fixed unbounded memory growth in long interactive sessions — subagent tool results are now released once they scroll off the recent display window. Reduces the need to manually trim context in long-running sessions.
- **MCP / plugin security:** plugin marketplaces can now run a `headersHelper` command that mints short-lived tokens for catalog/archive fetches, rather than embedding long-lived credentials in config. Prefer this pattern when authoring or auditing a plugin marketplace.
- **v2.1.263** (2026-09-06): bug-fix/reliability release, no user-facing prompting changes. v2.1.261 (2026-09-04) also shipped a long tail of fixes worth knowing about if you hit them: a safer `rm -rf` guard that now catches positional-parameter and quoted-script forms, a fix for `claude -p --resume` adopting malformed session IDs, and an "Organization policy" line in `/status`/`claude doctor` explaining why an org policy failed to load.

## 4. Cowork

File and task automation across a workspace — the surface for requests that end in a deliverable file, a scheduled routine, or work spanning multiple connected apps.

- **Surface-choosing rule of thumb (official framing):** if the request's success condition is "I have a file/artifact I can open, share, or keep," route to Cowork. If it's "I have an answer," that's Chat. If it's "the codebase changed," that's Code.
- State the deliverable's format up front (doc, sheet, deck, persisted page) — this determines which output-format skill gets invoked and changes how much structure the prompt needs.
- For anything recurring, say so explicitly ("do this every Monday") rather than asking for a one-off — Cowork's scheduled-task tooling is built for this and produces a materially different, more maintainable setup than a one-shot prompt repeated by hand.
- When a task needs the user's own connected data (mail, drive, chat tools), name the specific source rather than a generic "check my stuff" — this avoids an unnecessary connector search and a slower first turn.

> **Official — Desktop app changes (claude.com/docs/cowork/changelog, 2026-09-02 to 09-05).** The Cowork/Claude desktop app changelog is tracked separately from the Claude Code CLI changelog in §3 and hadn't been checked in earlier runs of this document. Four items from the last two weekly cycles are worth folding into how you scope a Cowork request:
>
> - **Device-bridge folder access widened** (v1.46388.3, 09-04): a user can now attach their whole home folder, Windows Documents/AppData, the macOS Library folder, or an entire drive — not just individual project folders. Claude's own config/session data, and credential or shell-startup locations (SSH keys, AWS/Google Cloud credentials, bash/zsh/PowerShell profiles), stay permanently off-limits regardless. Prompting implication: for a task spanning many folders, it's now reasonable to ask the user to attach one broad root instead of enumerating every subfolder.
> - **Scheduled tasks now auto-retry** (v1.46388.1, 09-04): a failed scheduled-task run gets an automatic re-run rather than silently failing once and going quiet. Reinforces the existing recommendation above to use Cowork's native scheduled-task tooling for recurring work rather than a manual "remind me and I'll re-run it by hand" pattern.
> - **Code sessions inside the desktop app** (v1.46388.1, 09-04) gained message queueing across the 5-hour usage-limit window — a message sent while limited now sends once the limit resets instead of being dropped — plus a keep-awake option for long-running sessions.
> - **Broader entry point** (v1.44121.1, 09-02): Claude was added to the OS "Open with" menu for common work files on macOS/Windows, and default artifact-sharing visibility now follows organization-level settings instead of a fixed default — worth checking with enterprise users before assuming an artifact you publish is private by default.

> **Known issue, confirmed first-hand 2026-09-09 — GitHub connector and Cowork sandbox both refuse writes to GitHub.** Tried, in one session: connecting GitHub as a custom remote MCP connector (`https://api.githubcopilot.com/mcp/`) authenticates fine and every read works (`get_me`, `get_file_contents`, `search_repositories`, …), but **every write call fails with `403 Resource not accessible by integration`** — `create_or_update_file`, `push_files`, and issue-write tools alike. This matches two open, unresolved public reports, not a one-off account issue: [anthropics/claude-ai-mcp#822](https://github.com/anthropics/claude-ai-mcp/issues/822) and [anthropics/claude-code#80874](https://github.com/anthropics/claude-code/issues/80874). Disconnecting/reconnecting the connector, revoking and re-authorizing on GitHub's side, and widening a fine-grained PAT's scopes do not fix it — the token the connector actually receives just isn't granted write capability, and the connector UI has no field to substitute a PAT directly.
>
> The obvious fallbacks from inside a Cowork cloud sandbox are blocked too, deliberately: a plain `git clone` over Bash is refused by the session's own action classifier (even with no credentials involved), and a direct GitHub REST API write (`PUT .../repos/{o}/{r}/contents/{path}`) over HTTPS with a manually-supplied PAT is blocked at the network-egress-proxy layer with an explicit `"Write access to this GitHub API path is not permitted through this proxy"` — which itself points at `docs.anthropic.com/en/docs/claude-code/github-actions`. Reads over the same proxy (GET requests, `curl` to public endpoints) succeed, so this is a deliberate write-path restriction, not a general network block.
>
> **Net effect:** as of this date, there is no working way for a Cowork session or scheduled task to write to a user's GitHub repo — not via the official connector, not via raw git/API calls. Don't spend time re-debugging PAT scopes, connector permissions, or GitHub App installs when this exact error shows up — it isn't a configuration problem on the user's side. See §7 for the path that does work today. Worth a quick re-check periodically whether [issue #822](https://github.com/anthropics/claude-ai-mcp/issues/822) has shipped a fix — if it has, the connector becomes the simpler option again for one-off writes from chat/Cowork.

## 5. Claude Design

Multi-artboard visual design published as an editable canvas — UI mockups, landing pages, marketing graphics, print pieces.

- Anchor the prompt in one concrete subject, its audience, and the artboard's single job — vague briefs ("make it look nice") reliably regress to templated, generic layouts.
- State a treatment, not just a format: "utilitarian and information-dense" versus "editorial, one bold risk" changes typography, color, and layout decisions throughout, not just decoration.
- Name real constraints (brand colors, existing type system, must work in both light and dark) up front — Design applies an existing system when one is described and only invents one when none exists.
- For iteration, refer to elements by what they are ("the hero headline," "the pricing table") rather than by position — canvas elements move during refinement, so positional references go stale fast.

## 6. Model-specific notes

Notes on how current-generation models (Claude Opus 5, Claude Sonnet 5, Claude Fable 5.1, and their predecessors still in rotation) respond differently to older prompting patterns.

- **Prefill** (seeding the start of the assistant turn to force a format) is a shrinking technique on current-generation models called via the API directly — explicit `<output_format>` instructions now get comparable compliance without it. Keep prefill for the narrow cases where only a hard format lock will do.
- **Verification instructions** ("double check your work," "explain your reasoning first") land differently on higher-effort/extended-thinking configurations, which already verify internally — an explicit instruction is more useful for lower-effort or non-reasoning configurations, where it's doing real work rather than being redundant.
- **Subagent delegation** patterns (Claude Code's `Agent`/Task tool, workflow scripts) are markedly more reliable on current models at following a self-contained brief with no shared memory of the parent conversation — lean on this rather than trying to carry context across agent boundaries.
- When the user names a specific model in a request, match your prompting pattern to it rather than defaulting to whatever pattern worked for the model you're most familiar with — capability and default verbosity shift release to release.

### Claude Fable 5.1 (and Mythos 5.1) — correct for these defaults

> **Official guidance — platform.claude.com, updated for the 2026-09-01 5.1 refresh.** Fable 5.1 sits above Opus 5 in Anthropic's lineup (see §8) and is prompted differently enough from Sonnet/Opus that Anthropic publishes a dedicated guide. The seven corrections below are the ones worth applying by default; treat them as defaults to override, not rules to explain to the user unless asked.

- **Effort is the primary control.** The `effort` parameter (`low`/`medium`/`high`/`xhigh`/`max`) matters more for Fable 5.1 than model choice does for cost/quality tradeoffs. Default is `high`; `low`/`medium` are often competitive with Opus/Sonnet at lower cost, and `xhigh`/`max` need a larger `max_tokens` buffer. Benchmark down from `high` rather than assuming more effort is always better.
- **Progress updates are hidden by default** — short inter-tool notes arrive as `thinking` blocks with `display:"omitted"`. Set `thinking.display:"updates"`, or add:
  > Before you start, say in a line what you're about to do; brief updates while you work help the user follow along. Close with a short recap.
- **Agent loops issue one tool call per turn** by default instead of batching independent calls in parallel. Nudge with:
  > First privately list what you need next; then request every item that doesn't depend on another's result in this one response.
- **Conversation history should be append-only.** For accounts created after 2026-08-31, thinking blocks are bound to their conversation context — editing earlier messages invalidates cached thinking. Use server-side compaction/context-editing instead of client-side summarization.
- **Denser prose, less formatting by default** — fewer paragraph breaks, less bold/headers/lists than Claude 5 defaulted to. If you want either behavior corrected:
  > Mannered prose substitutes metaphor and flourish for direct statement. The fix is to say what you mean. When a literal phrase is available, use it.
  >
  > Use lists and bullet points when asked to, or when the content is multifaceted enough that they help with clarity.
- **May describe next steps instead of finishing long autonomous tasks.** For unattended/long-horizon work, add explicit autonomy framing:
  > You are operating autonomously. The user is not watching in real time and cannot answer questions mid-task, so asking "Want me to…?" will block the work. For reversible actions that follow from the original request, proceed without asking.
  >
  > Before ending your turn, check your last paragraph. If it is a plan, analysis, or promise about work you have not done ("I'll…"), do that work now. End your turn only when the task is complete or you are blocked on input only the user can provide.
- **Subagent coordination:** have subagent tools return immediately rather than blocking the lead agent, pass results back in a later `user` message, and give the lead a separate explicit "wait" tool if it needs one — cuts wall-clock time without hurting quality.
- **Vision on dense images/charts:** provide a crop/zoom tool rather than relying on a single full-image pass — iterative cropping delivers most of the accuracy uplift on complex figures.

## 7. API / Claude Developer Platform

Direct API and platform-level agent-building capabilities: Messages API, tool use, computer use, Skills API, Files API.

> **Official — Reached GA, 2026-08-20.** Computer use, the Skills API, and the Files API are now production-ready (previously beta). Notable additions:
>
> - A **browser-use tool**: reads page structure alongside screenshots, so agents can target web elements reliably instead of relying on pixel coordinates. Prefer it over raw computer-use for web-specific tasks.
> - **Multiple actions per model turn** for computer use — fewer round-trips, materially faster task completion (one case study reported claims-workflow time dropping from 32 to 13 minutes).
> - Computer use now qualifies for **HIPAA-regulated workloads under a BAA**.
> - Files API: automatic expiration, 5× higher rate limits, 1 TB storage per organization.

> **Official — Added 2026-09-03, Developer Platform v1.30.0.** The **`ant` CLI** gained an **`ant apply`** command: create and update agents, environments, skills, memory stores, and deployments from files, with approval workflows and lockfile management for reproducible deployments — an infrastructure-as-code pattern for platform resources rather than clicking through a console.

> **Confirmed working path, 2026-09-09 — Claude Code GitHub Actions for scheduled repo writes.** For "Claude commits to my repo on a schedule" — the thing that doesn't work today from claude.ai/Cowork (see §4 known issue) — the supported, working path is the **Claude Code GitHub Action** (`anthropics/claude-code-action`). It runs inside the target repo's own GitHub Actions, not inside a claude.ai/Cowork session, authenticated with an `ANTHROPIC_API_KEY` or `CLAUDE_CODE_OAUTH_TOKEN` stored as an *encrypted GitHub repo secret* — proper secrets hygiene, nothing pasted into a prompt or chat. It ships a documented "run on a schedule" cron mode built for exactly this kind of daily job, including a worked example (daily commit/issue summary at a fixed UTC time).
>
> - **Setup, quick:** run the `claude` CLI locally inside a clone of the target repo, run `/install-github-app`, follow the prompts — it installs the Claude GitHub App, adds the secret, and opens a PR with the workflow file to merge.
> - **Setup, manual (no local CLI needed):** install the [Claude GitHub App](https://github.com/apps/claude) on the repo from a browser, add the secret under repo Settings → Secrets and variables → Actions, then commit a workflow file (starting point: [examples/claude.yml](https://github.com/anthropics/claude-code-action/blob/main/examples/claude.yml)) into `.github/workflows/`.
> - **Tradeoff to flag before recommending this:** it's a separate automation surface from a Cowork scheduled task — the job runs as its own Claude Code session inside GitHub's runners and has no access to a claude.ai Artifact (Actions has no way to authenticate to a user's Claude account). So a repo file, not an Artifact, has to be the source of truth for whatever this writes; keeping both in sync requires deliberate design, not something to assume comes for free.
>
> Source: code.claude.com/docs/en/github-actions — checked 2026-09-09.

### Prompting implications

- Package a repeated procedure as a versioned **skill** (instructions + scripts + templates) rather than re-explaining it in every prompt — this is now a first-class, production pattern, not a beta workaround.
- For browser/web-agent tasks, prompt for goals and let the browser-use tool handle element targeting, rather than hand-specifying pixel coordinates or brittle selectors.
- Since multiple computer-use actions can execute per turn, batch a task's actions into one instruction ("open settings, toggle X, save") rather than one action per prompt turn — it now maps to how the tool actually executes.
- For teams managing several agents/skills/deployments on the Developer Platform, prefer defining them as files and applying with `ant apply` over ad hoc console changes — it's now the reproducible, reviewable path, similar to infra-as-code.
- The `effort` parameter (`low`–`max`) is now the primary cost/quality lever on Fable 5.1 specifically — see §6 before assuming Opus/Sonnet's cost-control patterns (mostly model choice) transfer directly.
- When a task needs Claude to write to an external system on a recurring schedule (not just this session), check whether that system has its own scheduled/CI-native Claude integration (like GitHub Actions here) before defaulting to "give the Cowork scheduled task a stored credential" — a platform-native secrets store beats a token embedded in a prompt whenever one is available.

## 8. New & emerging products

Filled in as Anthropic ships new Claude surfaces or model tiers that need their own prompting guidance.

> **Official — New model tier: Claude Fable 5.1 / Mythos 5.1, announced 2026-09-01.** Fable and Mythos are a "Mythos-class" tier that sits above Claude Opus 5, first introduced as Fable 5 / Mythos 5 in June 2026 and refreshed to 5.1 on 2026-09-01. Both versions are the same underlying model at different safeguard levels: **Fable 5.1** (API id `claude-fable-5-1`) is generally available to everyone; **Mythos 5.1** has safeguards relaxed for vetted cyberdefense and life-sciences researchers through a trusted-access program and is not reachable through the standard API.
>
> - Positioned for demanding reasoning and long-horizon agentic work — coding, research, computational biology — where Opus 5 evaluations still fall short. Anthropic's own recommendation: start with Opus 5 for most workloads, reach for Fable 5.1 when a task needs more headroom.
> - 1M-token context, 128K max output; $10/$50 per million input/output tokens, cache reads at $0.25/MTok (a 75% discount) — roughly 25–45% cheaper per typical-to-highly-agentic workload than the original Fable 5.
> - Selectable today as a subagent model in Claude Code's/Cowork's `Agent` tool (alongside sonnet/opus/haiku). It needs several explicit corrections to its default behavior to prompt well — see §6 for the full list and copy-paste fixes.

**Claude Academy** (academy.claude.com, launched 2026-08-20) remains a free courses-and-tutorials learning hub — a resource to point users toward, not a promptable surface, so it doesn't get its own dedicated section here.

## 9. Community & open-source guidance — where it diverges

Reliable community prompting resources are useful, but a few widely-repeated patterns lag current official guidance.

> **Divergence.** Community guides (e.g. promptingguide.ai and many GitHub prompt-pattern collections) still lead with heavy prefill and "act as X" role-play framing as default techniques. Official current guidance treats explicit `<output_format>`/role instructions as sufficient for most cases and reserves prefill for hard format locks — see §6.

> **Divergence.** Some community agent-framework guides recommend maximal upfront context-stuffing ("give the agent everything it might need"). Official Claude Code guidance instead favors a lean `CLAUDE.md` plus on-demand file reads and subagent delegation — large unconditioned context tends to dilute attention rather than help.

## 10. Changelog

**2026-09-09** — First-hand finding, not routine research: the user asked to mirror this document to a personal GitHub repo with daily automated pushes. Confirmed two independent, currently-broken write paths and one working-but-heavier alternative, written up as new callouts in §4 and §7 for future reference. §4: connecting GitHub as a claude.ai custom MCP connector authenticates and reads fine but every write call returns `403 Resource not accessible by integration` — matches open reports [anthropics/claude-ai-mcp#822](https://github.com/anthropics/claude-ai-mcp/issues/822) and [anthropics/claude-code#80874](https://github.com/anthropics/claude-code/issues/80874), not fixable by reconnecting/re-authorizing/PAT scoping. Also confirmed the Cowork cloud sandbox independently blocks the fallbacks: plain `git clone` is refused by the session's action classifier, and a direct GitHub REST API write over HTTPS is blocked at the network-egress-proxy layer ("Write access to this GitHub API path is not permitted through this proxy"), which itself points at the GitHub Actions integration. §7: documented that working alternative — the Claude Code GitHub Action's cron/schedule mode, authenticated via an encrypted GitHub repo secret rather than a token in a prompt — including setup steps and the tradeoff that it can't reach a claude.ai Artifact directly. Net decision: no protocol change to this document's own daily maintenance (still Cowork scheduled task + Artifact, no GitHub push); did a one-time manual markdown/HTML export instead for the user to upload by hand. Re-check periodically whether issue #822 has shipped a fix. Sources: first-hand testing this session; github.com/anthropics/claude-ai-mcp/issues/822; github.com/anthropics/claude-code/issues/80874; code.claude.com/docs/en/github-actions — 2026-09-09.

**2026-09-07** — Added a new §4 callout on Cowork/Claude-desktop-app changes (v1.44121.1–v1.46388.4, 2026-09-02 to 09-05) that earlier runs of this document had missed, because they only checked the Claude Code CLI changelog (code.claude.com) and not the separate desktop-app changelog (claude.com/docs/cowork/changelog): device-bridge folder access widened to whole home folders/Documents/AppData/Library/entire drives (credential and shell-startup locations, plus Claude's own config, stay permanently excluded); scheduled tasks now auto-retry on failure; Code sessions inside the desktop app gained message queueing across the 5-hour usage limit plus a keep-awake option; and Claude was added to the OS "Open with" menu, with artifact-sharing defaults now following org-level settings. Checked and found unchanged since 09-06: Claude Code CLI (still v2.1.263), Claude Fable/Mythos 5.1 details, Claude Design, and API/Platform beyond the already-recorded `ant apply` release. Sources: claude.com/docs/cowork/changelog — checked 2026-09-07.

**2026-09-06** — Added Claude Fable 5.1 and Claude Mythos 5.1 (announced 2026-09-01) as a new "Mythos-class" model tier above Opus 5 — new §8 entry covering GA status, restricted Mythos access, context/pricing, and its availability as a Claude Code/Cowork subagent model. Added a substantial new §6 subsection, "Claude Fable 5.1 — correct for these defaults," from Anthropic's official prompting guide: the `effort` parameter as the primary cost/quality control, hidden-by-default thinking updates, one-tool-call-per-turn agent loops, append-only conversation history for cached thinking, denser/less-formatted prose by default, a tendency to describe next steps instead of finishing long autonomous tasks, subagent coordination patterns, and vision/cropping guidance — each with a copy-pasteable fix snippet. Cross-referenced from §3 and §7. Sources: anthropic.com/claude-fable-and-mythos-5-1, anthropic.com/news/claude-fable-5-mythos-5, platform.claude.com/docs/.../prompting-claude-fable-5-1, platform.claude.com/docs/en/models/overview — checked 2026-09-06.

**2026-09-06** — Added Claude Code v2.1.259–2.1.261/2.1.263 (2026-09-02 to 09-06) to §3: `/skill-doctor` for skill-usage/context-cost auditing, fullscreen `/diff` panel, prompt-cache-miss diagnostics in `/cost`/status line, `bashOutputMaxChars`/`taskOutputMaxChars` (up to 128K chars), `--append-subagent-system-prompt-file`, headless `/reload-plugins` and `--permission-prompts none`, and the `managedMcpServers` setting. Added Developer Platform v1.30.0's `ant apply` CLI command to §7 (agents/environments/skills/memory stores/deployments as files, with approval workflow and lockfiles). Source: code.claude.com/docs/en/changelog and corroborating changelog aggregators, checked 2026-09-06. Also fixed the underlying scheduled task: it previously ran research daily but was never wired to write findings back into this document — it now reads this artifact, updates it in place when something material changes, and republishes to the same URL every run.

**2026-08-22** — Rebuilt this reference as a Claude.ai Artifact (previously a Google Doc). Google Drive read/write access was gated behind a permission prompt that kept timing out in the scheduled-task environment, so per the user's request this file now lives here and is updated in place by the same daily scheduled task. Content refreshed from official sources current as of this date: Claude Code v2.1.236–2.1.239 (`ANTHROPIC_DEFAULT_MODEL`, Concise output style, output-style drift fix, subagent memory-release fix, plugin `headersHelper`); Computer Use / Skills API / Files API reaching GA on the Claude Developer Platform; Claude Academy learning-hub launch (noted as a resource, §8).

---

*Maintained by the daily "Claude & Claude Code best-practices watch" scheduled task. Re-checked each run against official docs (platform.claude.com, code.claude.com, claude.com/blog, support.claude.com) plus community prompting resources; only rewritten here when something material changes. The canonical, interactive version lives at the Claude.ai Artifact linked above — this file and the accompanying `prompting-standards.html` are a portable mirror for tools/agents that can't authenticate to claude.ai.*