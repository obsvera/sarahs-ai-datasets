# Claude Prompting Standards

A compiled reference of current best practices for writing prompts across every Claude product. Hand this file to any agent — Claude or otherwise — so it can generate high-quality, standards-compliant prompts for the user to run, rather than the user writing them by hand. Sections 0–9 state what's true _now_ ; nothing in them should be read as "changed recently" or "used to be different." §10 is a separate log for provenance and for open questions still worth double-checking.

**Maintained by** — daily scheduled task, "Claude & Claude Code best-practices watch" **Last updated** — 2026-09-20 **Storage** — Claude.ai Artifact

0

## How to use this file

If you are an agent that has been handed this file, and the user asks you to write, improve, or generate a prompt for a Claude product:

  * Identify which capability the request needs — a quick answer, a deliverable (agentic/Cowork-style work), coding in a repo, visual design, or direct API/Platform use. Claude Chat and Cowork share one interface and Claude decides on its own which kind of handling a request needs, so this is about picking the right section below, not picking a UI tab.
  * Apply the Universal Principles (§1) as a baseline, always.
  * Layer on the product-specific section's techniques and patterns.
  * Check the Model-Specific Notes (§6) if the user names a specific model — current-generation models respond differently to some older patterns, and Claude Fable 5.1 in particular needs several explicit corrections to its default behavior.
  * If community guidance (§9) conflicts with official docs, default to official guidance, and only surface the conflict if it's relevant to the user's case.
  * Check §10 for anything still open or unresolved that bears on the task (e.g. a known bug, an unconfirmed rumor) before treating a §0–9 statement as the final word on an edge case.
  * Produce the actual prompt text the user can copy and run — not commentary about prompting.



1

## Universal principles

From Anthropic's official prompt-engineering documentation. Hold regardless of which product surface is in use.

  * **Be explicit and direct.** State exactly what you want — treat Claude like a capable new hire with no institutional context. Spell out what "done" looks like.
  * **Give context and motivation** , not just instructions. Explaining _why_ something matters lets Claude generalize correctly to edge cases you didn't enumerate.
  * **Use examples.** Three to five, relevant and diverse, covering edge cases. Wrap them in `<example>`/`<examples>` tags in longer prompts.
  * **Structure complex prompts with XML tags** — `<instructions>`, `<context>`, `<document>`, `<output_format>`. Nest tags when content has hierarchy.
  * **Assign a role when it sharpens focus** — a short system-level framing ("you are a senior security engineer reviewing this diff") tightens tone and attention.
  * **Say what to do, not just what to avoid.** "Write in flowing prose" beats "don't use bullet points."
  * **Put long documents first** , above your instructions or question — this alone can lift accuracy meaningfully on long-context tasks. Wrap in `<document>` tags with source/metadata.
  * **Allow uncertainty.** Explicitly permitting "say so if you're not sure" measurably reduces hallucination.
  * **Match your prompt's formatting to your desired output.** Markdown-heavy prompts pull toward markdown-heavy answers.
  * **Shortest reliable prompt wins.** Add structure only where it fixes an observed failure mode — don't over-engineer.



2

## Claude Chat

Questions, brainstorming, short-to-medium exchanges — handled in the same window as agentic/deliverable work, not a separate app.

  * Default to direct, conversational phrasing. §1's XML/example structuring earns its keep once a prompt gets genuinely complex — multi-part questions, style constraints, structured output.
  * State the desired output format explicitly (tables, JSON, tone) rather than relying on prefill (seeding the start of Claude's reply) — see §6 for when prefill is still worth reaching for.
  * Use Projects for recurring topics instead of re-explaining background every conversation.
  * Claude decides on its own whether a request needs conversational handling or agentic/Cowork-style capability (file work, multi-step tasks, a persisted deliverable). A clear, explicit statement of the desired deliverable and its format (doc, sheet, deck, scheduled routine, etc.) is what triggers the right handling — a vague request is more likely to get answered conversationally when a file was actually wanted, or vice versa.



3

## Claude Code

An agentic coding environment: reads files, runs commands, edits code, works autonomously. Prompting it well is mostly about managing context and giving it a way to verify its own work.

### Core practices

  * Front-load a `CLAUDE.md` with project conventions, build/test commands, and gotchas — it's read at the start of every session. If a repo has no `CLAUDE.md`/`CLAUDE.local.md`, Claude Code reads an existing `AGENTS.md` instead by default — a repo already set up for another coding agent works with zero extra setup. Configurable under `/config` → Project instructions: load both (CLAUDE.md first), ignore AGENTS.md entirely, or load only an org's managed CLAUDE.md. On Bedrock or in telemetry-disabled sessions, import AGENTS.md explicitly with an `@AGENTS.md` line in a CLAUDE.md instead.
  * Give it a verification loop: tests, a linter, a type-checker, or an explicit "run this and check the output" instruction. Claude Code does its best work when it can tell for itself whether it succeeded.
  * For large or multi-part work, ask for a plan first (plan mode), then approve before it edits — cheaper to redirect a plan than to unwind a large diff.
  * Delegate isolated sub-problems to subagents (the `Agent`/Task tool) rather than growing one context window indefinitely; hand each subagent a self-contained brief since it starts with no memory of the parent conversation. Model options include sonnet, opus, haiku, and **Fable** — reach for Fable on long-horizon research, writing, or a task where Opus/Sonnet evaluations fall short, but correct for its default behavior first (§6).
  * Run `/skill-doctor` periodically to see which loaded skills are going unused and what they cost in context — trim skills that never fire.
  * Use `/diff` (a fullscreen diff panel) to review uncommitted changes as you go, and check `/cost` or the status line if a session feels slower or pricier than expected — both surface likely causes of prompt-cache misses.
  * Task-tracking tools (`TaskCreate`/`TaskUpdate`/`TaskList`, `TodoWrite`) are offered by default only on older model versions. On current-generation models (Opus 5, Sonnet 5, Fable 5.1), set `CLAUDE_CODE_ENABLE_TODO_TOOLS=1` to get them — otherwise a prompt or skill that assumes task-list tooling exists should fall back to plain prose progress narration.
  * Select the built-in **"Concise"** output style (`/config` → Output style, or `/output-style [name]`) for automation-friendly sessions that lead with results and skip preamble/narration.
  * Set `ANTHROPIC_DEFAULT_MODEL` to pin which model new sessions start on for a team or CI default; a manual `/model` pick still overrides it.
  * For unattended/headless hosts, pair `--permission-prompts none` with a properly scoped permission config rather than using it to skip permissioning altogether.
  * Claude.ai account skills and plugins sync automatically into a terminal session signed in with that account — worth knowing before assuming a session's skill set matches only its local project config (opt out per-session with `syncClaudeAiSkills:false`/`syncClaudeAiPlugins:false`).
  * A scheduled or "Run now" routine can save data to, and republish the page of, an artifact it already owns without a permission prompt — a first publish, a public artifact, or a delete still asks.
  * A built-in in-session browser is available as a lighter-weight alternative to a full computer-use/browser-use setup for simple in-session browsing, where an organization enables it.



4

## Cowork

File and task automation, sharing one interface with Claude Chat — the right mental model for a request that ends in a deliverable file, a scheduled routine, or work spanning multiple connected apps.

  * **Rule of thumb:** if the request's success condition is "I have a file/artifact I can open, share, or keep," it needs agentic/Cowork-style capability. If it's "I have an answer," conversational handling is enough. If it's "the codebase changed," that's Code.
  * State the deliverable's format up front (doc, sheet, deck, persisted page) — this determines which output-format skill gets invoked and is what triggers correct auto-routing.
  * For anything recurring, say so explicitly ("do this every Monday") rather than asking for a one-off — the scheduled-task tooling produces a materially more maintainable setup than a one-shot prompt repeated by hand. A failed scheduled run retries automatically rather than silently failing once.
  * When a task needs the user's own connected data (mail, drive, chat tools), name the specific source rather than a generic "check my stuff" — this avoids an unnecessary connector search and a slower first turn.
  * For documents and decks, prefer **Claude Docs** (a real-time collaborative text editor — comments, revision tracking, export to Word/PDF/Google Docs/markdown) and **Claude Slides** (prompt-or-research-to-deck, presentable directly or exportable to PowerPoint/PDF) when the account has them. Both are in beta: available by default on Pro/Max, opt-in for Team/Enterprise, and unavailable to any org configured with customer-managed encryption keys, zero data retention, or a HIPAA-ready setup. Whether these fully supersede the older Pages/Slides ready-made artifact types isn't settled — if unsure which one a session will use, check directly rather than assuming.
  * A connected device folder can be a whole home folder, Documents/AppData, the macOS Library folder, or an entire drive, not just individual project folders — for a task spanning many folders it's reasonable to ask the user to attach one broad root instead of enumerating subfolders. Claude's own config/session data and credential or shell-startup locations stay off-limits regardless.



Known issue — GitHub connector cannot write

Connecting GitHub as a connector authenticates and reads fine, but write calls (creating/updating files, pushing) fail with `403 Resource not accessible by integration`. The obvious fallbacks from inside a Cowork sandbox are blocked too: a plain `git clone`/push is refused by the session's action classifier, and a direct GitHub REST API write is blocked at the network-egress layer. This isn't a configuration problem on the user's side and isn't fixed by reconnecting, re-authorizing, or widening a token's scopes — it's an open, unresolved issue (tracked publicly as `anthropics/claude-ai-mcp#822`). A partial community workaround exists (installing the Claude GitHub App directly on the target repo instead of relying on the connector's own auth), but results are inconsistent — verify with a real commit before relying on it.

**What actually works:** the Claude Code GitHub Action (see §7) — it runs inside the target repo's own GitHub Actions, authenticated with a repo secret, and has a built-in cron/schedule mode. Its limitation: it can't reach a claude.ai Artifact, so a repo file (not an Artifact) has to be the source of truth for anything it writes.

5

## Claude Design

Multi-artboard visual design published as an editable canvas — UI mockups, landing pages, marketing graphics, print pieces. Works inline inside any conversation, not only as its own surface.

  * Anchor the prompt in one concrete subject, its audience, and the artboard's single job — vague briefs ("make it look nice") reliably regress to templated, generic layouts.
  * State a treatment, not just a format: "utilitarian and information-dense" versus "editorial, one bold risk" changes typography, color, and layout decisions throughout, not just decoration.
  * Name real constraints (brand colors, existing type system, must work in both light and dark) up front — Design applies an existing system when one is described and only invents one when none exists.
  * For iteration, refer to elements by what they are ("the hero headline," "the pricing table") rather than by position — canvas elements move during refinement, so positional references go stale fast.



6

## Model-specific notes

How current-generation models (Claude Opus 5, Claude Sonnet 5, Claude Fable 5.1, and their predecessors still in rotation) respond differently to older prompting patterns.

  * **Prefill** (seeding the start of the assistant turn to force a format) is a shrinking technique on current-generation models called via the API directly — explicit `<output_format>` instructions now get comparable compliance without it. Keep prefill for the narrow cases where only a hard format lock will do.
  * **Verification instructions** ("double check your work," "explain your reasoning first") land differently on higher-effort/extended-thinking configurations, which already verify internally — an explicit instruction is more useful for lower-effort or non-reasoning configurations, where it's doing real work rather than being redundant.
  * **Subagent delegation** patterns (Claude Code's `Agent`/Task tool, workflow scripts) are markedly more reliable on current models at following a self-contained brief with no shared memory of the parent conversation — lean on this rather than trying to carry context across agent boundaries.
  * When the user names a specific model in a request, match your prompting pattern to it rather than defaulting to whatever pattern worked for the model you're most familiar with — capability and default verbosity shift release to release.



### Claude Fable 5.1 (and Mythos 5.1) — correct for these defaults

Official guidance — platform.claude.com

Fable 5.1 sits above Opus 5 in Anthropic's lineup (§8) and is prompted differently enough from Sonnet/Opus that Anthropic publishes a dedicated guide. The corrections below are worth applying by default; treat them as defaults to override, not rules to explain to the user unless asked.

  * **Effort is the primary control.** The `effort` parameter (`low`/`medium`/`high`/`xhigh`/`max`) matters more for Fable 5.1 than model choice does for cost/quality tradeoffs. Default is `high`; `low`/`medium` are often competitive with Opus/Sonnet at lower cost, and `xhigh`/`max` need a larger `max_tokens` buffer. Benchmark down from `high` rather than assuming more effort is always better.
  * **Progress updates are hidden by default** — short inter-tool notes arrive as `thinking` blocks with `display:"omitted"`. Set `thinking.display:"updates"`, or add: 
        
        Before you start, say in a line what you're about to do; brief updates while you work help the user follow along. Close with a short recap.

  * **Agent loops issue one tool call per turn** by default instead of batching independent calls in parallel. Nudge with: 
        
        First privately list what you need next; then request every item that doesn't depend on another's result in this one response.

  * **Conversation history should be append-only.** For accounts created after 2026-08-31, thinking blocks are bound to their conversation context — editing earlier messages invalidates cached thinking. Use server-side compaction/context-editing instead of client-side summarization.
  * **Denser prose, less formatting by default** — fewer paragraph breaks, less bold/headers/lists than Claude 5 defaulted to. If you want either behavior corrected: 
        
        Mannered prose substitutes metaphor and flourish for direct statement. The fix is to say what you mean. When a literal phrase is available, use it.
        
        Use lists and bullet points when asked to, or when the content is multifaceted enough that they help with clarity.

  * **May describe next steps instead of finishing long autonomous tasks.** For unattended/long-horizon work, add explicit autonomy framing: 
        
        You are operating autonomously. The user is not watching in real time and cannot answer questions mid-task, so asking "Want me to…?" will block the work. For reversible actions that follow from the original request, proceed without asking.
        
        Before ending your turn, check your last paragraph. If it is a plan, analysis, or promise about work you have not done ("I'll…"), do that work now. End your turn only when the task is complete or you are blocked on input only the user can provide.

  * **Subagent coordination:** have subagent tools return immediately rather than blocking the lead agent, pass results back in a later `user` message, and give the lead a separate explicit "wait" tool if it needs one — cuts wall-clock time without hurting quality.
  * **Vision on dense images/charts:** provide a crop/zoom tool rather than relying on a single full-image pass — iterative cropping delivers most of the accuracy uplift on complex figures.



7

## API / Claude Developer Platform

Direct API and platform-level agent-building capabilities: Messages API, tool use, computer use, Skills API, Files API.

  * **Computer use, the Skills API, and the Files API** are GA. Prefer the **browser-use tool** over raw computer-use for web-specific tasks — it reads page structure alongside screenshots, so an agent can target elements reliably instead of relying on pixel coordinates. Since multiple computer-use actions can execute per model turn, batch a task's actions into one instruction ("open settings, toggle X, save") rather than one action per prompt turn. Computer use qualifies for HIPAA-regulated workloads under a BAA. The Files API auto-expires files, has higher rate limits, and 1 TB storage per organization.
  * **On-demand compaction** (`compaction` parameter, beta): for a long-running agent loop built directly on the Messages API, trigger summarization on your own schedule — optionally in the background — instead of waiting for automatic threshold-based compaction. Fire it asynchronously between turns rather than blocking the loop on it. A **keep-tail** pattern (excluding recent turns from the summarization request) preserves those turns word-for-word, along with their extended-thinking chains where the model supports it. If system-prompt rules or tool availability need to stay active after compaction, restate them in a system message after the compaction block — the summary doesn't carry that forward automatically.
  * **Claude Managed Agents** (server-hosted agents with a managed sandbox): use the `auto` permission-policy option to let the server itself evaluate, run, deny, or pause each agent/MCP tool call, rather than pre-declaring a static rule or requiring human-in-the-loop approval for everything — write the evaluation criteria into the policy prompt/config rather than trying to enumerate allowed calls. Use `ant beta:sessions connect` to attach a terminal to a live session for spot-checking or approving pending calls.
  * Package a repeated procedure as a versioned **skill** (instructions + scripts + templates) rather than re-explaining it in every prompt. For teams managing several agents/skills/deployments, define them as files and apply with `ant apply` (agents, environments, skills, memory stores, deployments — with approval workflows and lockfile management) rather than ad hoc console changes.
  * The `effort` parameter (`low`–`max`) is the primary cost/quality lever on Fable 5.1 specifically — see §6 before assuming Opus/Sonnet's cost-control patterns (mostly model choice) transfer directly.
  * When a task needs Claude to write to an external system on a recurring schedule (not just this session), check whether that system has its own scheduled/CI-native Claude integration — e.g. the **Claude Code GitHub Action** for a GitHub repo, which runs inside the repo's own GitHub Actions authenticated with an encrypted repo secret and ships a documented cron/schedule mode — before defaulting to a stored credential inside a Cowork scheduled task. See §4's known-issue callout for why that matters for GitHub specifically. Setup: run `/install-github-app` from a local Claude Code session inside a clone of the target repo (installs the GitHub App, adds the secret, opens a PR with the workflow file), or install the [Claude GitHub App](https://github.com/apps/claude) manually from a browser and commit a workflow file yourself. Its tradeoff: it can't authenticate to a claude.ai Artifact, so a repo file — not an Artifact — has to be the source of truth for anything it writes.



8

## Products & model tiers

What currently exists beyond the core Chat/Code/Cowork/Design surfaces, and how to scope a prompt for each.

  * **Claude Fable 5.1 / Mythos 5.1** — a model tier above Claude Opus 5, positioned for demanding reasoning and long-horizon agentic work (coding, research, computational biology) where Opus 5 evaluations fall short. Fable 5.1 is generally available to everyone; Mythos 5.1 has safeguards relaxed for vetted cyberdefense/life-sciences researchers through a trusted-access program and isn't reachable through the standard API. Start with Opus 5 for most workloads, reach for Fable 5.1 when a task needs more headroom. 1M-token context, 128K max output. Selectable as a subagent model in the `Agent`/Task tool. Needs the corrections in §6 to prompt well.
  * **Life Sciences Verification Program** — academic labs, startups, pharma companies and other life-science organizations can apply for verification (research credentials, security standards, ethical-oversight review) to unlock relaxed safety classifiers for legitimate biology work, across Mythos, Opus, and Sonnet, and across Claude Science, claude.ai, Claude Code, and the API. If a life-sciences user hits a safety-classifier block on legitimate research, the fix is applying to this program, not prompt-engineering around the classifier.
  * **Claude for Financial Advisors** — an Enterprise Cowork plugin bundling workflow skills (meeting prep/summaries, portfolio rebalancing review, alternative-investment briefs, estate/tax plan review, prospect intake, compliance flagging) with connectors into custody/portfolio, alternatives, wealth-management, and CRM platforms. Frame requests as producing a draft/brief/flag for the advisor to review, not an instruction to execute a financial or compliance decision — critical outputs require advisor sign-off by design.
  * **Salesforce in Claude** ("Claudeforce") — a beta plugin bringing a seller's Salesforce accounts, opportunities, and pipeline into Claude chat and Cowork, using the seller's existing Salesforce permissions. Ships 37 prebuilt sales skills (account research, call prep, pipeline review/deal scoring, meeting follow-up, renewal prep, QBR decks, forecast narratives), paired with a Salesforce connector (read/write) and the Slack connector. Defaults to asking the seller to approve each proposed change before it's written — frame prompts as drafts for approval, not direct execution. Requires the latest Sales Cloud enterprise edition and Salesforce-side beta approval.
  * **Smart reports** — an Enterprise admin-analytics feature that analyzes a team's Claude usage transcripts (up to 28 days) into workstreams, costs, friction points (blocked connectors, approval gates, tool failures), and skill-packaging recommendations. It is explicitly not designed for evaluating individual performance or making employment decisions — don't scope a prompt that way. Unavailable to orgs with customer-managed encryption keys, a HIPAA configuration, Access Transparency, or Claude Code zero-data-retention.
  * **Claude Academy** — a free courses-and-tutorials learning hub (academy.claude.com). A resource to point users toward, not a promptable surface.



No corroborated model beyond Fable/Mythos 5.1 exists as of this writing — treat "Opus 5.1"/"Sonnet 5.1" claims from low-quality blogs or social media as unconfirmed rumor unless anthropic.com/news or platform.claude.com says otherwise.

9

## Community & open-source guidance — where it diverges

Reliable community prompting resources are useful, but a few widely-repeated patterns lag current official guidance.

Divergence

Community guides (e.g. promptingguide.ai and many GitHub prompt-pattern collections) still lead with heavy prefill and "act as X" role-play framing as default techniques. Official current guidance treats explicit `<output_format>`/role instructions as sufficient for most cases and reserves prefill for hard format locks — see §6.

Divergence

Some community agent-framework guides recommend maximal upfront context-stuffing ("give the agent everything it might need"). Official Claude Code guidance instead favors a lean `CLAUDE.md` plus on-demand file reads and subagent delegation — large unconditioned context tends to dilute attention rather than help.

10

## Log

Sections 0–9 are kept evergreen: they state current behavior as fact, and resolved issues or superseded techniques are removed from them entirely rather than marked "changed." This log exists for two things instead: a provenance trail (what changed, when, sourced), and a short watch-list of things that are unresolved enough to be worth an agent double-checking before relying on a §0–9 statement in an edge case.

### Open — worth double-checking before relying on it

  * Whether Claude Docs/Slides fully supersede the older Pages/Slides ready-made artifact types, or the two remain parallel, is still not confirmed by official documentation (open since the 2026-09-16 launch).
  * The GitHub connector write-403 issue (`anthropics/claude-ai-mcp#822`) remains open with no official fix; only a partial, inconsistent workaround exists (§4).
  * No new base model beyond Fable/Mythos 5.1 has been corroborated by Anthropic. "Opus 5.1"/"Sonnet 5.1" rumors keep circulating on low-quality sources — re-check anthropic.com/news and platform.claude.com before treating them as real.



### History

  * 2026-09-20

Rewrote §0–9 to state current practice directly rather than as a changelog of "recent changes" — dated framing, version-by-version CLI release notes, and resolved issues (e.g. the Windows file-access bug, Cowork's local→cloud migration rough edges, individual dated "Update, 2026-09-xx" callouts) were folded into plain current-state facts or dropped where fully superseded. Full prior history condensed into this log. Also folded in, from the prior run: **Salesforce in Claude** and **Smart reports** , both added to §8; §4's Cowork known-issues section trimmed to the one still-open item (GitHub write-403).

  * 2026-09-16 to 2026-09-19

Claude Chat and Cowork merged into one interface with auto-routing between conversational and agentic handling; Claude Design became available inline in any conversation. Claude Docs and Claude Slides launched in beta (real-time collaborative doc editor; prompt/research-to-deck generation), rolling out to Pro/Max first. Claude Code shipped native AGENTS.md support, a server-side auto-mode classifier (no extra billing), an agent-map view of subagents, and a long tail of prompt-cache-reuse fixes for the subagent-delegation pattern. Source: claude.com/blog/cowork-is-now-claude; code.claude.com/docs/en/changelog (v2.1.271–v2.1.278).

  * 2026-09-01 to 2026-09-17

Claude Fable 5.1 / Mythos 5.1 announced as a model tier above Opus 5, with a dedicated prompting guide (now §6). Life Sciences Verification Program launched, formalizing Mythos 5.1's trusted-access program into named grant tiers. Claude for Financial Advisors launched as an Enterprise Cowork plugin. Salesforce in Claude ("Claudeforce," 09-15) and Smart reports (09-10) launched in beta — both only surfaced via support.claude.com's release-notes page, now added to this routine's standing source list. Sources: anthropic.com/claude-fable-and-mythos-5-1; anthropic.com/news/life-sciences-verification-program; support.claude.com/en/articles/12138966-release-notes.

  * 2026-08-20 to 2026-09-14

Computer use, the Skills API, and the Files API reached GA with a browser-use tool and multi-action turns. Messages API on-demand compaction shipped in beta. Claude Managed Agents gained an `auto` permission-policy option and `ant beta:sessions connect`. Developer Platform's `ant apply` shipped for infra-as-code style agent/skill deployment. Claude Code shipped the Concise output style, `/skill-doctor`, `/diff`, and the task-tracking-tools default change on current-gen models. First-hand testing (2026-09-09) confirmed the GitHub connector write-403 issue and documented the GitHub Actions workaround, now §4/§7. Windows file-access bug (09-08) affecting locally-run Cowork sessions was later resolved by a Windows update (KB5129195, 09-14) — no Claude-side fix was needed.

  * 2026-08-22

Rebuilt this reference as a Claude.ai Artifact (previously a Google Doc), after Google Drive read/write access proved unreliable in the scheduled-task environment.




Maintained by the daily "Claude & Claude Code best-practices watch" scheduled task. Re-checked each run against official docs (platform.claude.com, code.claude.com, claude.com/blog, support.claude.com) plus community prompting resources; §0–9 rewritten in place when something material changes, §10 kept as the provenance/watch-list log.
