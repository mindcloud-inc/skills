# MindCloud Agent Skills

[![skills.sh](https://skills.sh/b/mindcloud-inc/skills)](https://skills.sh/mindcloud-inc/skills)

Agent skills that let coding agents (Claude Code, Codex, Cursor, and [70+ more](https://github.com/vercel-labs/skills#supported-agents)) work with [MindCloud](https://mindcloud.co).

| Skill | What it does |
|---|---|
| [mindcloud-universal-api](skills/mindcloud-universal-api) | Read and write data in 3,000+ business apps (Slack, NetSuite, Shopify, QuickBooks, HubSpot, and more) through one REST API. |
| [mindcloud-api](skills/mindcloud-api) | Manage a MindCloud account: run and debug workflows, manage connections, members, roles, environments, and API keys. |

Both skills need a MindCloud API key. Get one at [app.mindcloud.co/user/api-keys](https://app.mindcloud.co/user/api-keys) and export it as `MINDCLOUD_API_KEY`.

## Install

With the [skills CLI](https://github.com/vercel-labs/skills) (works with every supported agent):

```bash
npx skills add mindcloud-inc/skills
```

Install one skill only:

```bash
npx skills add mindcloud-inc/skills --skill mindcloud-universal-api
```

### Claude Code plugin

```
/plugin marketplace add mindcloud-inc/skills
/plugin install mindcloud-skills@mindcloud
```

### Manual

Copy a skill folder into your agent's skills directory, for example `~/.claude/skills/`:

```bash
git clone https://github.com/mindcloud-inc/skills mindcloud-skills
cp -r mindcloud-skills/skills/mindcloud-universal-api ~/.claude/skills/
```

## Try it

After you install and export `MINDCLOUD_API_KEY`, ask your agent things like:

- "Send 'hello' to the #general channel in Slack"
- "List the NetSuite customers created this week"
- "Why did my last workflow run fail?"

## Learn more

- Universal API docs: https://mindcloud.co/docs/universal/rest/home/latest
- MindCloud API docs: https://mindcloud.co/docs/api/rest/introduction
- OpenAPI spec (no auth needed): https://connect.mindcloud.co/v2/openapi.json

## License

[MIT](LICENSE)
