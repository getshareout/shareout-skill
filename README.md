# ShareOut Skill

Agent skill for [ShareOut](https://github.com/getshareout/shareout) — teaches coding
agents how to build, publish, and manage live web artifacts.

## Important: where to edit

| Role | Location |
|------|----------|
| **Canonical source (edit here)** | [`getshareout/shareout`](https://github.com/getshareout/shareout) → [`skills/ShareOutSkill/`](https://github.com/getshareout/shareout/tree/main/skills/ShareOutSkill) |
| **This repo** | **Publish / install mirror** for agents that only need the skill tree |
| **Runtime (preferred for use)** | `GET {your-instance}/v1/skill` — versioned zip served by the Worker |

Do **not** open PRs here as the primary workflow. Open them against the product
repo so the skill stays in lockstep with the platform. Maintainers mirror into
this repo on skill releases.

Full policy: [skills/README.md in shareout](https://github.com/getshareout/shareout/blob/main/skills/README.md).

```text
EDIT:  github.com/getshareout/shareout/skills/ShareOutSkill/
  │
  ├─► sync:skill  →  $ORIGIN/v1/skill   (what agents should load at runtime)
  └─► release     →  this repo            (clone / marketplace install)
```

## How agents should load the skill

### 1. Against a running instance (best)

```text
GET https://shareout.site/v1/skill          # hosted
GET https://your-worker.example/v1/skill   # self-host
GET …/v1/skill/version                     # staleness check
```

Point credentials at that origin:

```json
// ~/.shareout/credentials
{ "token": "so_…", "origin": "https://your-worker.example" }
```

### 2. From this mirror (offline / skill marketplace)

Clone this repo and have the agent read **`SKILL.md`**, then **`INDEX.md`**.

If the frontmatter `version` is behind `GET $ORIGIN/v1/skill/version`, prefer the
live zip — this mirror may lag between releases.

### 3. From the product monorepo (deploy + use)

```text
git clone https://github.com/getshareout/shareout.git
# Install/deploy on Cloudflare:
#   shareout/skills/ShareOutSkill/deploy/SKILL.md
# Use an instance:
#   shareout/skills/ShareOutSkill/SKILL.md
```

Deploy docs (Cloudflare, domain, D1/R2/DO) live under `deploy/` in the monorepo
skill tree — use that path when the user wants **ShareOut installed**, not only
the publish API.

## Start here (this checkout)

- **[SKILL.md](SKILL.md)** — use skill: token, publish, SDK, routing  
- **[INDEX.md](INDEX.md)** — intent → file map  

For **install on Cloudflare**, clone
[getshareout/shareout](https://github.com/getshareout/shareout) and open
`skills/ShareOutSkill/deploy/SKILL.md` (may be newer than this mirror).

## License

[MIT](LICENSE) © ShareOut
