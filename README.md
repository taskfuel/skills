# taskfuel skills

Agent skills for [taskfuel.ai](https://taskfuel.ai) — one account for the paid web. Hold a
prepaid USD balance, and let your coding agent spend it per call on APIs it would
otherwise have no way to reach.

## Install

```sh
npx skills add taskfuel/skills
```

Works with Claude Code, Cursor, Codex, Copilot, Windsurf, Gemini CLI, Zed, and the rest
of the [agent-skills](https://github.com/vercel-labs/skills) ecosystem.

To install a single skill:

```sh
npx skills add taskfuel/skills --skill taskfuel
```

## Skills

| Skill | What it does |
| --- | --- |
| [`taskfuel`](skills/taskfuel/SKILL.md) | Lets your agent discover and call paid HTTP-402 APIs (search, market data, enrichment, and more) through your taskfuel.ai account. |
| [`contact-enrichment`](skills/contact-enrichment/SKILL.md) | Researches one sales contact and writes a Markdown brief — role, recent posts, company context, and conversation-starter hooks — using free web search plus a few paid calls. |

Using the skills requires an account and a funded balance at
[taskfuel.ai](https://taskfuel.ai).

## Contributing

This repo is a read-only mirror of the released skills served by `app.taskfuel.ai` —
edits made here will be overwritten by the next sync. To report a problem or suggest a
change, please open an issue instead of a pull request.

## License

MIT — see [LICENSE](LICENSE).
