# Documentation maintenance

Use the documentation guide to find a workflow. Keep each procedure in one
canonical page and link to it from overview pages.

## Where material belongs

| Material | Location | Update policy |
|---|---|---|
| Project introduction and quickstart | Root README | Short overview and links |
| Installation, user workflows and reference | `docs/` | Describe supported behavior; add pages to MkDocs navigation |
| Contributor checks and repository conventions | `CONTRIBUTING.md`, `CLAUDE.md` | Keep executable commands and constraints current |
| Research decisions and operations | `research/README.md`, observer guides | Separate current decisions from dated evidence |
| Frozen protocols, results and amendments | `research/`, `research/results/` | Preserve original method, thresholds and failures; append explicit corrections |
| Implementation plans and acceptance evidence | `EPICS/` | Retain task IDs and acceptance history |
| Release and incident history | Changelog and migration history | Keep dated evidence; link to current procedures |
| Service-specific setup | Component README | Own local commands; link to shared manuals |

Public documentation is built from `docs/` and `mkdocs.yml`. Research archives,
private host observations and personal deployment decisions do not belong in
that published tree. The repository research index is an internal entry point.

## How to revise a page

1. Identify whether the text is a current instruction, a proposal or a dated
   observation. Label the latter two explicitly.
2. Verify commands and paths against the checkout. For service status, state the
   observation date; an enabled unit or old screenshot is not a fresh health check.
3. Replace repeated procedures with links to the canonical guide. Keep safety
   prerequisites close to the action they constrain.
4. Preserve heading anchors when possible. A changed filename needs updated
   links or a compatibility entry. Code deleted from a historical design should
   be identified as historical evidence rather than presented as current source.
5. Do not edit generated release blocks or frozen research outcomes as prose
   cleanup. Correct interpretation with a dated amendment.

Use English for user-facing UI and manuals. Preserve source quotations and
historical records in their original language when their wording is evidence.

## Validate and publish

From the repository root:

```bash
python3 -m venv .venv-docs
.venv-docs/bin/pip install -r requirements-docs.txt
.venv-docs/bin/mkdocs build --strict
```

Also check repository-relative links outside the published documentation;
MkDocs cannot validate every research or contributor page. Review the diff to
ensure commands, policy values and evidence have not changed accidentally.

Publishing is a separate operation through `scripts/publish-docs.sh`. That
script checks for clean publishable paths and runs the strict build. A local
build proves the site can be generated; it does not prove the public site has
been updated. See [code signing](code-signing.md) and
[migration history](release-history.md) for release-specific constraints.
