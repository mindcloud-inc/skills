# Maintaining this repo

Skills live under `skills/<skill-name>/SKILL.md`. Both skills are model-invoked: the agent may reach for them on its own when the task fits their `description`. Do not add `disable-model-invocation` to the frontmatter or `policy.allow_implicit_invocation: false` to `agents/openai.yaml` — that would stop agents from using them without an explicit slash command.

Each skill carries:

- `SKILL.md` — frontmatter needs `name` and `description` (both strings, required by installers). `metadata.source` points at this repo. `metadata.requires.env` lists required env vars.
- `agents/openai.yaml` — Codex-facing display metadata.

Rules when you change a skill:

1. Keep `name` in frontmatter equal to the directory name.
2. If the `description` changes, mirror it in `.claude-plugin` manifests only if quoted there, and in the website well-known index (below).
3. The website serves `https://mindcloud.co/.well-known/agent-skills/index.json` (in the gravity monorepo at `projects/website/public/.well-known/agent-skills/index.json`). Its entries pin a commit SHA of this repo and carry a `sha256:` digest of each SKILL.md. After changing any SKILL.md here, update that file: new pinned commit URL, new digest (`shasum -a 256 SKILL.md`). A stale digest makes `npx skills add mindcloud.co` fail verification.
4. The website docs page `content/docs/universal/introduction/build-with-ai.mdx` and `src/app/llms.txt/route.js` in the monorepo link here; keep install commands accurate.
5. Validate manifests after touching them: `claude plugin validate . --strict`.
6. Bump `version` in both `.claude-plugin/plugin.json` and `.claude-plugin/marketplace.json` on user-visible changes.
