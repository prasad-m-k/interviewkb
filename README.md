# interviewkb

A persistent, LLM-maintained knowledge base for technical interview prep, built as an Obsidian vault. Claude Code reads source material and maintains the wiki; the human curates sources and asks the questions.

See [LLMWiki.md](LLMWiki.md) for the pattern this repo implements, and [INTERVIEW_NAVIGATOR.md](INTERVIEW_NAVIGATOR.md) for a role/company-based checklist of what to study.

## Layout

Each top-level folder is a self-contained wiki with its own `index.md`, `log.md`, and (where applicable) `topics/`, `concepts/`, `patterns/`, `problems/`, `companies/`, and `sources/` subdirectories, per the schema in `CLAUDE.md`.

| Folder | Covers |
|---|---|
| `dsa/` | Data structures & algorithms |
| `sre/` | Site reliability engineering |
| `devops/` | DevOps, IaC, Terraform, Ansible |
| `k8s/` | Kubernetes |
| `java/` | Java / backend engineering |
| `ml/` | Machine learning |
| `mlops/` | MLOps |
| `obs/` | Observability |
| `agents/` | Agentic AI |
| `solution-arch/` | Solution architecture |
| `figure/` | Figure AI (Staff SRE) prep |
| `git/` | Git |
| `amazon/` | Amazon leadership principles (STAR/SOAR) |
| `google/` | Google Googliness & leadership |
| `sources/` | Raw ingested source documents (read-only) |

## Conventions

- Internal links use Obsidian wikilink syntax: `[[path/page-name]]`.
- Each wiki folder is append-only in its `log.md`; content pages are updated in place, with contradictions noted rather than silently overwritten.
- `sources/` is immutable — read from, never edited.

Full schema and workflows (ingest / query / lint) are documented in `CLAUDE.md`.
