# pbx3-docs

**How this was built:** PBX3 was **designed by humans** in mid/late 2025. Most of the code was **implemented in Cursor** under **human-directed AI** assistance — not unattended generation, and not a claim that AI is a panacea.

**Operator documentation (all PBX3 repos):** [pbx3-docs (MkDocs)](https://pbx3-oss.github.io/pbx3-docs/)

- **Way forward:** classical install / admin procedure pages and the shipped installers in these repositories.
- **Optional co-pilot:** if you already use an AI coding agent, start at [AI-assisted operations](https://pbx3-oss.github.io/pbx3-docs/getting-started/ai-assisted/) — same scripts and human gates. Not unattended; **AI is not a panacea**.

Published **installer and administrator** documentation for PBX3 (MkDocs Material).

Not developer workingdocs — those stay in the product repos under `workingdocs/`.

## Local review

```bash
cd pbx3-docs
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
mkdocs serve
```

Open http://127.0.0.1:8000/

## GitHub Pages

Remote: **https://github.com/pbx3-oss/pbx3-docs**.

Push to `main` with Actions enabled; workflow runs `mkdocs gh-deploy --force` (standard MkDocs Material + GitHub Pages pattern). Published site: **https://pbx3-oss.github.io/pbx3-docs/**.

## Content map

Nav and page inventory: `pbx3/workingdocs/USER_GUIDES_MKDOCS_CONTENT_MAP.md`.
