# MotifJS Agent Skills

Skills that teach AI coding agents how to build, modify and debug applications written with [MotifJS](https://motifjs.com) (`@motifx/core` + `@motifx/compiler`).

Each skill follows the open [Agent Skills](https://agentskills.io) format: a `SKILL.md` with frontmatter plus reference files that the agent loads on demand. The same folder works in Claude Code, Codex CLI, Gemini CLI, GitHub Copilot, Cursor and every other tool that reads Agent Skills.

## Skills

| Skill | What it covers |
|---|---|
| [`motifjs`](skills/motifjs/SKILL.md) | Components, reactivity, JSX bindings, conditionals and lists, events and forms, lifecycle and disposal, router, dependency injection, styling and transitions, virtualization, testing, mobile and Electron, error codes. |

## Install

**Claude Code**

```
/plugin marketplace add motifjsdev/skills
/plugin install motifjs@motifjs
```

**Any Agent Skills compatible tool** (Codex CLI, Gemini CLI, GitHub Copilot, Cursor, ...)

```
npx skills add motifjsdev/skills
```

**Manual**

Copy `skills/motifjs` into your agent's skills directory, for example `.claude/skills/motifjs` in a Claude Code project. See [agentskills.io](https://agentskills.io) for the directory each tool reads.

## Layout

```
.claude-plugin/marketplace.json   Claude Code marketplace manifest
skills/motifjs/SKILL.md           entry point: mental model, rules, reference map
skills/motifjs/references/*.md    detailed references loaded on demand
```

## Links

- Website and docs: https://motifjs.com
- Framework source: https://github.com/motifjsdev/motifjs
- npm: [`@motifx/core`](https://www.npmjs.com/package/@motifx/core), [`@motifx/compiler`](https://www.npmjs.com/package/@motifx/compiler)

## License

MIT
