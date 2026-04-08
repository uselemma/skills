# Lemma Agent Skills

[Agent Skills](https://github.com/anthropics/skills) that teach AI coding assistants (Cursor, Claude Code, Windsurf, etc.) how to correctly integrate [Lemma](https://docs.uselemma.ai) — AI observability.

## Skills

| Skill | Description |
|---|---|
| [`lemma-tracing`](./lemma-tracing/SKILL.md) | Integrate Lemma tracing into any codebase — detects language, framework, and existing OTel setup to pick the right integration path |

## Installation

### skills CLI

```bash
npx skills add uselemma/skills --skill "lemma-tracing"
```

### Cursor

```bash
npx skills add uselemma/skills --skill "lemma-tracing" --target cursor
```

Or install manually into your project's `.cursor/rules/` directory:

```bash
mkdir -p .cursor/rules
curl -o .cursor/rules/lemma-tracing.md \
  https://raw.githubusercontent.com/uselemma/skills/main/lemma-tracing/SKILL.md
```

### Claude Code

```bash
npx skills add uselemma/skills --skill "lemma-tracing" --target claude
```

## Usage

Once installed, the agent will automatically use these skills when relevant — for example:

- Adding Lemma tracing to a new or existing project
- Choosing the right integration path (native SDK vs. OpenTelemetry)
- Debugging instrumentation issues

## Versioning

Skills are versioned alongside the Lemma SDK. When the SDK API changes, the skill is updated in the same PR so agents always generate up-to-date code.
