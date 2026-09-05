# Research: Prior Art — Tools That Explain or Fix Failed Commands

Date: 2026-07-08
Status: Informative (feeds DESIGN.md §2, §4 and ADR-001/ADR-003)

## Purpose

Survey existing tools in the "my last command failed, help me" space to (a) position
whydid, (b) learn which capture mechanisms work in practice, and (c) avoid known
failure modes. Facts below were verified against public sources on 2026-07-08.

## Comparison table

| Tool | Core value | How it gets the failure context | Engine | Fix execution model |
|---|---|---|---|---|
| thefuck | Corrects the previous console command | Default: reads shell history, **re-runs the command** to capture output. Experimental "instant mode": logs terminal output with the Unix `script` utility + prompt markers, reads the log instead of re-running | Rule based (hundreds of built-in rules) | Prints corrected command; user confirms; thefuck **executes it** |
| GitHub Copilot CLI (`copilot`) | General agentic CLI; can explain commands | User works inside the Copilot session or pastes the command; no ambient shell hook that observes your normal prompt | LLM (GitHub-hosted) | Agent may run commands inside its own session |
| shell "AI helper" plugins (zsh-codex, ai-shell, etc.) | Generate a command from natural language | User explicitly invokes with a prompt; typically no failure capture at all | LLM (BYOK) | Insert into buffer or print |
| whydid (this project) | **Explain why the previous command failed** + propose fixes | Shell hooks record metadata continuously; real error output captured opportunistically (tmux) at invocation time | LLM (BYOK, provider-agnostic) | Prompt insertion only; whydid never executes commands |

## Key findings

### 1. thefuck's two capture modes bracket the design space

- **Re-run mode (default)**: simple, but re-executes the previous command to observe
  its output. This is unsafe for side-effectful commands (deletes, deploys, billable
  API calls) and slow for long-running ones. thefuck added `require_confirmation` and
  timeouts, but the class of risk is inherent to the approach.
- **Instant mode (experimental)**: wraps the session with the `script` utility, adds a
  marker to the shell prompt so the log can be segmented per command, then reads the
  log instead of re-running. It is faster and safer, but: it records **all** terminal
  output (including secrets) to disk continuously, requires prompt modification, is
  limited to bash/zsh + Python 3, and is still flagged experimental with known
  breakage reports (e.g. nvbn/thefuck#811).

**Implication for whydid (ADR-001):** both endpoints of the spectrum have documented
problems. whydid takes the middle path: always-available metadata via hooks (cheap,
no output on disk), plus opportunistic real-output capture from the terminal
multiplexer at invocation time (no continuous recording, nothing persisted).

### 2. Execution of fixes is the main safety divider

thefuck executes the corrected command after confirmation. That makes the tool itself
an execution engine for generated text, which widens its threat surface (rule bugs,
malicious rules from its plugin system). Copilot CLI has had real-world issues in this
area (e.g. misdiagnosing `posix_spawnp` launch failures as "command not found",
github/copilot-cli#2736), illustrating that agentic execution multiplies failure modes.

**Implication for whydid (ADR-003):** whydid never executes commands. The strongest
UX we allow is placing the chosen fix into the shell's input line (zsh editing buffer /
bash history recall), so the human presses Enter. This keeps the human physically in
the execution path.

### 3. Rule engines are a maintenance treadmill

thefuck's value comes from a large hand-maintained rule set. Rules are precise but
cover only anticipated failures; every new tool/version needs new rules. An LLM
inverts this: broad coverage, no rule maintenance, at the cost of network dependency
and payload privacy questions.

**Implication for whydid (ADR-002):** LLM-primary with BYOK. No built-in rule engine
in v1 (deferred; see ISSUE_PLAN "Deferred v2"). Privacy cost is addressed head-on with
mandatory redaction, minimal default payload, first-run consent, and `--show-payload`.

### 4. Naming collision check

Web searches for a "whydid" CLI tool (2026-07-08) surfaced no existing project with
this name; results were dominated by GitHub CLI documentation. GitHub search shows no
popular repository named `whydid` in the CLI-tools space. The name is considered safe
for an OSS release. (Re-verify immediately before first public release announcement.)

### 5. LLM provider landscape (for the BYOK abstraction)

- **OpenAI-compatible surface** is the de-facto interop standard. Ollama exposes
  `POST /v1/chat/completions` (plus `/v1/models`, `/v1/embeddings`) at
  `http://localhost:11434/v1`, accepting any non-empty API key. Many other local and
  hosted runtimes copy the same shape. Supporting `base_url` + OpenAI chat format
  therefore covers OpenAI, Ollama, and most self-hosted gateways with one client.
- **Anthropic Messages API** is a distinct surface: `POST /v1/messages` with
  `x-api-key` + `anthropic-version: 2023-06-01` headers and a different
  request/response schema. Supported as a second, separate provider implementation.

**Implication:** two concrete provider implementations (openai-compatible, anthropic)
behind one small interface give practical coverage of cloud + local without an SDK
dependency tree. Model IDs are user configuration, never hardcoded product behavior;
implementers must verify current model names at implementation time.

## Sources

- thefuck README and instant-mode description: https://github.com/nvbn/thefuck ,
  https://raw.githubusercontent.com/nvbn/thefuck/3.26/README.md ,
  https://deepwiki.com/nvbn/thefuck/3.3-instant-mode
- thefuck instant mode breakage report: https://github.com/nvbn/thefuck/issues/811
- GitHub Copilot CLI repo/issues: https://github.com/github/copilot-cli ,
  https://github.com/github/copilot-cli/issues/2736 ,
  https://github.blog/ai-and-ml/github-copilot-cli-101-how-to-use-github-copilot-from-the-command-line/
- Ollama OpenAI compatibility: https://docs.ollama.com/api/openai-compatibility ,
  https://ollama.com/blog/openai-compatibility
- Anthropic Messages API reference (headers, endpoint): https://platform.claude.com/docs/en/api
