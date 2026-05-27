# Conventions

Project-specific terminology, coding rules and repository layout.
Items marked **TBD** are intentionally unresolved and will be settled
during the repo bootstrap step.

- [Glossary](#glossary)
- [Code conventions](#code-conventions)
- [Git conventions](#git-conventions)
- [Naming conventions](#naming-conventions)
- [Project structure](#project-structure)

---

## Glossary

| Term              | Definition                                                                                                                  |
|-------------------|-----------------------------------------------------------------------------------------------------------------------------|
| Control plane     | Service that orchestrates, configures or deploys other components. Here: `container-deployer` (FastAPI).                    |
| Data plane        | Service that serves data and business requests. Here: `portfolio-web` (Django).                                             |
| Mono-repo         | A single Git repository containing several services or components.                                                          |
| Polyrepo          | Opposite approach: one Git repository per service.                                                                          |
| IaC               | *Infrastructure as Code* — describing infrastructure in versioned files (Terraform, Ansible, Pulumi…).                      |
| ADR               | *Architecture Decision Record* — short document tracing one structural technical decision. See [`architecture.md`](./architecture.md). |
| SOPS              | *Secrets OPerationS* — Mozilla tool for encrypting secret files committed to Git.                                           |
| age               | Modern encryption library, a simpler alternative to GPG.                                                                    |
| SSE               | *Server-Sent Events* — unidirectional server-to-client HTTP streaming protocol.                                             |
| DinD              | *Docker-in-Docker* — running a Docker daemon inside a container.                                                            |
| Twelve-factor app | Cloud-native app design methodology ([12factor.net](https://12factor.net)).                                                 |
| Reverse proxy     | Front-line server that routes requests to backend services.                                                                 |
| Brotli            | HTTP compression algorithm, more efficient than gzip for static web content.                                                |
| ngx_brotli        | Nginx module adding Brotli support.                                                                                         |
| WSL2              | *Windows Subsystem for Linux v2* — Linux running in a lightweight VM integrated with Windows.                               |

---

## Code conventions

### Python

- **Style:** PEP 8 — **TBD:** enforcement via `ruff` or `black` + `isort`?
  To be settled during repo setup.
- **Type hints:** required on public functions and dataclasses.
- **Comments:** English only.
- **Imports:** stdlib → third-party → local, alphabetical within each group.
- **Docstrings:** **TBD** — Google, NumPy or reStructuredText style?

### Django

- **Models:** singular names (`User`, not `Users`). Always define `__str__`
  and a `Meta` class (`verbose_name`, `ordering` where relevant).
- **Views:** thin views. Business logic lives in services or managers, not
  inside views.
- **Templates:** naming convention **TBD**.

### Frontend

- **CSS:** Tailwind utility-first. No custom CSS.
- **Naming:** **TBD**.

---

## Git conventions

### Branches

Format: `<type>/<short-description>`

- `feat/` — new feature
- `fix/` — bug fix
- `chore/` — maintenance, dependencies, configuration
- `refactor/` — refactor with no behaviour change
- `docs/` — documentation only

### Commits

- **Format:** [Conventional Commits](https://www.conventionalcommits.org)
  (`feat: add login`, `fix: handle null user`).
- **Granularity:** one concern per commit.
- **Squash before merging to `main`.**

---

## Naming conventions

| Where                          | Convention         | Example             |
|--------------------------------|--------------------|---------------------|
| Python variables / functions   | `snake_case`       | `get_user_by_id`    |
| Python classes                 | `PascalCase`       | `UserProfile`       |
| Python constants               | `UPPER_SNAKE_CASE` | `MAX_RETRIES`       |
| Python files / modules         | `snake_case.py`    | `user_service.py`   |
| URL routes                     | `kebab-case`       | `/user-profile/`    |
| Environment variables          | `UPPER_SNAKE_CASE` | `DATABASE_URL`      |

---

## Project structure

Mono-repo organised in runtime (`services/`) + support
(`infra/`, `docs/`, `.github/`).

```text
portfolio-meta/
├── README.md
├── CLAUDE.md                   # Claude Code instructions
├── .gitignore
├── .editorconfig
├── .env.example                # env variables (no secrets)
├── Makefile                    # common commands
│
├── services/
│   ├── portfolio-web/          # Django (data plane)
│   │   ├── pyproject.toml
│   │   ├── uv.lock
│   │   ├── .python-version
│   │   ├── Dockerfile
│   │   ├── manage.py
│   │   ├── portfolio/          # Django "project" (config)
│   │   ├── apps/               # Django "apps" (blog, projects, cv…)
│   │   ├── static/
│   │   ├── templates/
│   │   └── tests/
│   │
│   └── container-deployer/     # FastAPI (control plane)
│       ├── pyproject.toml
│       ├── uv.lock
│       ├── .python-version
│       ├── Dockerfile
│       ├── src/
│       │   ├── main.py
│       │   ├── api/
│       │   ├── core/
│       │   └── docker_client/
│       └── tests/
│
├── infra/
│   ├── docker-compose.yml         # local dev
│   ├── docker-compose.prod.yml    # prod overlay
│   ├── nginx/                     # custom Brotli image
│   └── terraform/                 # added later if relevant
│
├── docs/
│   ├── architecture.md            # ADRs (mirror of Notion)
│   ├── conventions.md             # this file
│   ├── deployment.md
│   └── security.md                # deployer threat model
│
└── .github/
    └── workflows/
        ├── portfolio-web.yml
        ├── container-deployer.yml
        └── infra.yml
```
