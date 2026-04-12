# Rails-with-AI

A `CLAUDE.md` template that teaches [Claude Code](https://docs.anthropic.com/en/docs/claude-code) modern Rails 8+ best practices. Drop it into any Rails project and Claude will follow production-grade conventions for Ruby style, ActiveRecord, Hotwire, security, testing, and more — automatically, on every interaction.

## Why

Claude Code reads `CLAUDE.md` from your working directory and applies its rules to every conversation (including sub-agents). Without guardrails, AI-generated Rails code can drift: symbol-form callbacks, `after_create` for jobs instead of `after_create_commit`, fat controllers with business logic, Devise in new projects, unscoped queries that leak data, and more. This template codifies the conventions experienced Rails teams (and DHH) enforce in code review, so Claude follows them from the first line.

## Two Versions

Pick the one that fits your project.

| File | Size | Contains | Best for |
|---|---|---|---|
| [`CLAUDE.md.FULL`](./CLAUDE.md.FULL) | 39,682 chars | All rules **plus BAD/GOOD code examples** | Projects where you want Claude to pattern-match from examples, or onboarding human devs alongside Claude |
| [`CLAUDE.md.SHORT`](./CLAUDE.md.SHORT) | 28,699 chars | All rules, **no code examples** | Context-sensitive projects, large monorepos, or when you already trust Claude to apply Rails idioms correctly |

Both files cover the **same rules** — the only difference is whether each rule is illustrated with a BAD/GOOD code block. Rules that lived inside code-only examples in the FULL version were folded into prose in SHORT, so no guidance is lost.

<details>
<summary><strong>What's inside (click to expand)</strong></summary>

| Area | Key rules |
|---|---|
| **Output & Efficiency** | Token-efficient responses (adapted from [claude-token-efficient](https://github.com/drona23/claude-token-efficient)), no sycophantic fluff, ASCII-safe output |
| **Ruby Style & Code Quality** | Expanded conditionals over guard clauses (DHH), method ordering (class → public → private), lambda-only callbacks, double quotes, files under 200 lines, `frozen_string_literal` |
| **Controllers** | REST purity (new resources over custom actions), thin controllers, `params.expect`, `before_action` setup, authentication concerns, `head :status` responses |
| **Models** | Rich models + concerns, alphabetical includes, nested concern paths, `normalizes`, `encrypts`, `generates_token_for`, `has_secure_password`, enums with `prefix: true` |
| **ActiveRecord & Queries** | `inverse_of`, `touch`, `counter_cache`, `dependent`, `includes`/`preload`/`eager_load`, `find_each`, `pluck`, `exists?` over `present?`, `where.missing`/`where.associated` |
| **Callbacks & Jobs** | MANDATORY lambda syntax, `after_*_commit` for async work, `_later`/`_now` pair pattern, jobs as shallow wrappers, `retry_on`/`discard_on`, `config/recurring.yml` |
| **Hotwire (Turbo + Stimulus)** | `turbo_frame_tag`, `turbo_stream_from`, broadcast from commit callbacks, Stimulus structure (targets/values/outlets, private `#fields`, Lifecycle/Actions/Private), server-rendered HTML only, native `<dialog>` |
| **Views & ERB** | Partials, collection rendering, `dom_id`, `tag.` helpers, `link_to`/`button_to`, never `.html_safe`, `sanitize`, i18n with `t()` |
| **Error Handling** | `rescue_from`, specific exception classes, never bare `rescue`, `Rails.error.report` |
| **Performance & Caching** | N+1 prevention, fragment/Russian-doll caching, Solid Cache, `counter_cache`, indexes, `pluck`/`select` |
| **Security & Auth** | Rails 8 auth generator (no Devise), `Current` attributes, CSRF, CSP, encrypted credentials, `allow_browser`, never `.html_safe` |
| **Project Structure** | Canonical `app/` layout, concerns, no `app/services/` by default, `lib/` minimalism |
| **Testing** | Minitest + fixtures (DHH defaults), `assert_difference`, `assert_enqueued_with`, system tests with Capybara, parallel tests |
| **Tooling** | `rubocop-rails-omakase`, `brakeman`, `bundler-audit`, `bin/importmap audit`, `bin/ci` |
| **Migrations & Database** | Reversible migrations, zero-downtime patterns, concurrent indexes, optional UUID primary keys |
| **Rails 8+ & Deployment** | Solid Trio (Queue + Cache + Cable, no Redis), Importmap + Propshaft, Kamal, `/up` health check |
| **Maintenance** | Dependency hygiene, dead code removal, prefer-simpler-alternatives table (Devise → auth generator, Sidekiq → Solid Queue, etc.) |

Both versions also include a **new-project setup checklist** (14 steps), **existing-project rules** (including the centralized escape hatch for RSpec/FactoryBot/Devise projects), a **pre-completion checklist**, and a customizable **Project-Specific Overrides** section at the bottom.

</details>

## Installation

1. **Download** the version you want into your Rails project root:

   ```bash
   # FULL (with code examples)
   curl -o CLAUDE.md https://raw.githubusercontent.com/maxart/Rails-with-AI/main/CLAUDE.md.FULL

   # Or SHORT (rules only)
   curl -o CLAUDE.md https://raw.githubusercontent.com/maxart/Rails-with-AI/main/CLAUDE.md.SHORT
   ```

   The `curl` commands above already rename the file to `CLAUDE.md` — if you download manually, rename it yourself.

2. **Customize** the `Project-Specific Overrides` section at the bottom of `CLAUDE.md` with your stack details — authentication approach, background job setup, testing framework, deployment rules.

Claude Code picks it up automatically on every conversation. No slash commands, no configuration.

## How It Works

```
your-rails-project/
  CLAUDE.md          <-- Claude reads this automatically
  app/
  config/
  Gemfile
```

Claude Code loads `CLAUDE.md` from the working directory (and parent directories) at the start of every conversation. The instructions become part of Claude's system context and apply to the main agent and all sub-agents.

<details>
<summary><strong>How scope works</strong></summary>

| Placement | Effect |
|---|---|
| Project root | Applies to all work in the project |
| `src/` subdirectory | Applies only when working in `src/` |
| Multiple levels | Rules stack — deeper files add to (or override) parent rules |

</details>

## Customization

The template is a starting point, not a straitjacket.

- **Remove** rules that conflict with your project (e.g., delete the Minitest section if you use RSpec).
- **Add** project-specific rules anywhere — API conventions, git strategy, design system usage, feature flag patterns.
- **Override** defaults in the `Project-Specific Overrides` block at the bottom.

Rules with code examples (FULL version) are followed more reliably than abstract guidelines. If Claude keeps missing a rule, make it more specific or add an example.

<details>
<summary><strong>Example: Project-Specific Overrides</strong></summary>

```markdown
## Project-Specific Overrides

### Authentication
- This project uses Devise (pre-existing). Follow existing Devise conventions.
- OAuth via OmniAuth for Google and GitHub

### Background Jobs
- Sidekiq with Redis (pre-existing). Do not migrate to Solid Queue.

### Testing
- RSpec + FactoryBot (pre-existing). Do not migrate to Minitest.
- Minimum 80% line coverage on new code

### Deployment
- Kamal deploys to Hetzner. config/deploy.yml is the source of truth.
- Do not modify .github/workflows/ without approval
```

</details>

## Companion File

| File | Description |
|---|---|
| [`docs/RAILS_BEST_PRACTICES_GUIDE.md`](./docs/RAILS_BEST_PRACTICES_GUIDE.md) | ~2,000-line tutorial covering each rule in depth with explanations, alternatives, and gotchas. For learning or onboarding — not meant to be loaded as `CLAUDE.md`. |

## Compatibility

- **Claude Code**: CLI, desktop app, web app, IDE extensions (VS Code, JetBrains)
- **Rails**: 8.0+
- **Ruby**: 3.2+
- **Database**: SQLite, PostgreSQL, MySQL
- **Frontend**: Hotwire (Turbo + Stimulus), Importmap, Propshaft, or any approach

## FAQ

**Which version should I pick?**
Start with **FULL** if you're unsure. The BAD/GOOD examples make Claude more consistent at applying each rule, especially on unfamiliar patterns. Switch to **SHORT** if context size matters (large monorepo, heavy sub-agent use) or once your team has settled on conventions and Claude reliably follows them.

**Does this work with existing RSpec / Devise / Sidekiq projects?**
Yes. Section 20 ("When Working on an Existing Rails Project") is a centralized escape hatch: it tells Claude to follow whatever conventions the project already uses, even when they diverge from the DHH defaults in this template. Customize the `Project-Specific Overrides` section to make it explicit.

**Will this slow Claude down?**
No. `CLAUDE.md` is loaded once per conversation and is negligible in the context window.

**Can I use this with other AI coding tools?**
The file targets Claude Code, but the content is standard Rails best practices and portable to other agents:

- **Agents that read `AGENTS.md`** (Codex, OpenCode, and others adopting that convention): download it directly as `AGENTS.md`.

  ```bash
  curl -o AGENTS.md https://raw.githubusercontent.com/maxart/Rails-with-AI/main/CLAUDE.md.FULL
  ```

- **Both Claude Code and an `AGENTS.md`-based agent in the same repo**: save the file as `AGENTS.md`, then create a one-line `CLAUDE.md` that imports it. Claude Code expands `@path` imports into context at session start, so both tools read identical instructions from a single source — no duplication, no drift.

  ```bash
  curl -o AGENTS.md https://raw.githubusercontent.com/maxart/Rails-with-AI/main/CLAUDE.md.FULL
  echo '@AGENTS.md' > CLAUDE.md
  ```

- **Cursor, Windsurf, Copilot**: adapt the rules into `.cursor/rules/`, `.windsurfrules`, or `.github/copilot-instructions.md`.

**Claude is ignoring a rule.**
`CLAUDE.md` is high-priority but not absolute. If a rule is consistently ignored, make it more specific or add an example. Rules with code examples are followed more reliably — the main reason to prefer FULL over SHORT.

## Further Reading

- [Ruby on Rails Guides](https://guides.rubyonrails.org/) — especially [Getting Started](https://guides.rubyonrails.org/getting_started.html) and [Active Record Basics](https://guides.rubyonrails.org/active_record_basics.html)
- [Hotwire](https://hotwired.dev/) — Turbo + Stimulus documentation
- [Solid Queue](https://github.com/rails/solid_queue) — database-backed Active Job backend
- [Kamal](https://kamal-deploy.org/) — containerized deployment
- [rubocop-rails-omakase](https://github.com/rails/rubocop-rails-omakase) — DHH-blessed RuboCop configuration
- [claude-token-efficient](https://github.com/drona23/claude-token-efficient) — the rules behind our "Output and Efficiency" section

## License

MIT
