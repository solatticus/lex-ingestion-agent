# Design Notes — Captured from working sessions

---

## Feb 16, 2026 — Architecture Decision: oscar.exe + fren.exe

### Context

While planning the lex ingestion agent, the conversation evolved into a broader architecture discussion about Oscar's machine presence tooling.

### Decision: Two Binaries, Shared Core

The system splits into two Rust binaries sharing a common framework:

| Binary | Role | I/O Model |
|--------|------|-----------|
| `oscar.exe` | Full UI experience — persistent session, rich TUI, interactive | Terminal UI (ratatui/crossterm or similar) |
| `fren.exe` | Headless, single-shot, pipe-friendly — takes a prompt, does the work, returns output | stdin/stdout, exit codes |

Think `curl` vs browser. Same engine, different interface.

### Shared Core Architecture

```
oscar-core/          (Rust lib crate — shared framework)
├── api/             # Claude API client, auth, token management
├── tools/           # Tool definitions, execution engine
├── prompts/         # Prompt templates, operator prompts
├── context/         # File reading, diff analysis, repo awareness
└── config/          # Shared configuration

oscar/               (binary crate)
└── tui/             # Rich terminal UI, session management, history

fren/                (binary crate)
└── cli/             # Arg parsing, stdin/stdout, single-shot execution
```

### Use Cases

**Oscar (interactive):**
- Debugging sessions
- Code exploration
- Conversational development
- Persistent context across turns

**Fren (headless):**
- CI/CD actions (lex ingestion, PR reviews, code checks)
- Git hooks (pre-commit analysis)
- Cron jobs (scheduled knowledge sync)
- Piped workflows: `git diff | fren --prompt "review this diff"`
- The lex ingestion workflow from PLAN.md becomes a fren use case

### Lex Ingestion Agent — Revised

With fren.exe, the lex ingestion agent from PLAN.md simplifies significantly. Instead of a standalone TypeScript app calling the Claude API, the GitHub/Gitea Action just calls:

```bash
git diff HEAD~1..HEAD | fren --prompt "sync .lex docs with these changes" --context .lex/
```

The action YAML becomes trivial — fren handles all the intelligence.

### Naming Notes

Names considered for the headless binary:
- rake — taken (Ruby Make)
- scrape/scraper — overloaded (web scraping)
- glean — Facebook code indexer exists
- sift, distill, trawl, comb — available but generic
- **fren** — chosen. Short, memorable, implies a helpful companion.

### Open Questions

- [ ] Should fren support streaming output or always batch?
- [ ] Does fren need a --session flag for multi-turn piped workflows?
- [ ] Where does the shared core live? Monorepo with oscar, or separate crate?
- [ ] Should fren have built-in tool definitions (file read, grep, git) or load them from config?
