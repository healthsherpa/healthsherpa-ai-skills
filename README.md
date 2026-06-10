# HealthSherpa Skills

Teach your AI coding assistant how to build with HealthSherpa APIs.

Skills give AI agents the context they need to generate accurate integration code: correct endpoints, field names, payload structures, known quirks, and carrier-specific behavior. Our skills follow the open [Agent Skills](https://agentskills.io) standard.

## Quick Start

Clone or download this repo, then copy the skill into your agent's skills directory.

**Cursor**

```bash
mkdir -p .cursor/skills
cp -r ichra_platform .cursor/skills/ichra-platform-integration
```

**Claude Code**

```bash
mkdir -p .claude/skills
cp -r ichra_platform .claude/skills/ichra-platform-integration
```

**GitHub Copilot / VS Code**

```bash
mkdir -p .github/skills
cp -r ichra_platform .github/skills/ichra-platform-integration
```

The skill is now available in your project. Use `@ichra-platform-integration` to attach it as context or `/ichra-platform-integration` to invoke it directly (where supported by your tool).

For global availability across all projects, copy to the user-level directory instead (e.g., `~/.cursor/skills/`, `~/.claude/skills/`).

## Available Skills

| Skill | Description |
|-------|-------------|
| [ichra_platform](ichra_platform) | ICHRA platform integration: plan quoting, enrollment (EnrollConnect and Deeplink), payment, policy lifecycle, and webhooks. |

## How Skills Work

Each skill is a folder with a `SKILL.md` file and supporting references. Agents read the skill's name and description first, load the full `SKILL.md` when the task is relevant, and pull in reference files as needed for deeper detail.

```
ichra_platform/
  SKILL.md                          # Primary context (loaded first)
  references/
    data-model.md                   # Request/response schemas, field types
    quoting-and-plans.md            # Quoting API, plan lookup, FIPS resolution
    deeplink-enrollment.md          # Deeplink schema, 302 handling, flat field mapping
    payment-and-documents.md        # Payment flows, document upload
    webhooks-and-monitoring.md      # Webhook payloads, polling, reconciliation
  evals/
    evals.json                      # Test prompts to verify the skill works
```

## Evals

Each skill includes an `evals/evals.json` with test prompts and assertions you can use to verify the agent produces correct output.

Example eval: *"Build a quoting function for ICHRA"* should produce code that uses `POST /api/v1/quotes`, includes `off_ex: true`, reads `api_enrollment` and `deeplink_enrollment` flags, and uses `cost_sharing.medical_ded_ind` for deductible display.

## Official Documentation

These skills supplement the official HealthSherpa documentation. If there is ever a discrepancy, the official docs are authoritative.

| Audience | Documentation |
|----------|---------------|
| ICHRA Platforms | [docs.ichra.healthsherpa.com](https://docs.ichra.healthsherpa.com) |
| Agents and Agencies | [one.healthsherpa.com](https://one.healthsherpa.com) |

## Disclaimer

These skills are provided as-is. AI-generated code may contain errors and must be reviewed and tested before use in any environment. See [LICENSE.md](LICENSE.md) for full terms and the AI usage disclaimer.

## License

MIT. See [LICENSE.md](LICENSE.md).
