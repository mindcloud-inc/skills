# Maintaining the MindCloud agent skills

The source of these skills is `projects/public/agentSkills` in MindCloud's monorepo. `github.com/mindcloud-inc/skills` is a copy that `.github/workflows/agentSkillsPublish.yml` pushes on every merge to master: never edit that repo directly, the next sync overwrites it.

Skills live under `skills/<skill-name>/SKILL.md`. Both skills are model-invoked: the agent may reach for them on its own when the task fits their `description`. Do not add `disable-model-invocation` to the frontmatter or `policy.allow_implicit_invocation: false` to `agents/openai.yaml`; that would stop agents from using them without an explicit slash command.

## Generated, never by hand

- Each skill's endpoint reference, between the `BEGIN GENERATED ENDPOINT REFERENCE` and `END GENERATED ENDPOINT REFERENCE` markers, is built from the v2 endpoint definitions and their docs blocks, the same source as mindcloud.co/docs/api. Change an endpoint's docs in `projects/publicApi/v2`, then run `npm run docs:website:generate` in `projects/publicApi`.
- The same command writes the copies mindcloud.co serves (`projects/website/public/skills/**`) and the discovery index with their digests (`projects/website/public/.well-known/agent-skills/index.json`).
- publicApi `docs:check` (CI · api-public) fails when any of these differ from what the generator writes. After editing anything in a SKILL.md, run the generator and commit its output.

## Rules when you change a skill

1. Keep `name` in frontmatter equal to the directory name. `metadata.requires.env` lists required env vars.
2. Keep hand-written guidance to how to use the API; facts about endpoints belong in the endpoint docs, which feed the generated reference.
3. If a `description` changes, mirror it in `.claude-plugin` manifests if quoted there. The discovery index takes it from the frontmatter.
4. Bump `version` in both `.claude-plugin/plugin.json` and `.claude-plugin/marketplace.json` on user-visible changes.
5. Validate manifests after touching them: `claude plugin validate . --strict`.
