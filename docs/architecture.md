# Architecture Decisions

This document tracks the structural technical decisions taken on this project,
in **ADR** format (*Architecture Decision Record*, Michael Nygard, 2011).

Each entry captures **why** a decision was made at a given point in time, what
was on the table, and what it commits us to. The goal is not to be exhaustive,
but to keep enough trace so that future-me (or a contributor, or a reviewer in
an interview) can reconstruct the reasoning without having to ask.

> 💡 If this file grows past ~20 entries, consider splitting it into one file
> per decision under `docs/adr/NNNN-short-title.md`. For a project this size,
> a single file stays scannable.

## How to add a new decision

Append a new section at the bottom using the next sequential number.
Never rewrite history: if a decision is superseded, mark it with status
`Superseded by [ADR-NNNN]` and add a new ADR for the new direction.

Template:

```markdown
## ADR-NNNN: Short Title

- **Date:** YYYY-MM-DD
- **Status:** Active | Proposed | Superseded by [ADR-NNNN] | Deprecated
- **Context:** Why we are deciding now, what triggered it.
- **Decision:** What we chose.
- **Alternatives:** What was on the table, why rejected.
- **Consequences:** What this implies (positive and negative, short and long term).
```

---

## Index

| #   | Title                                                              | Date       | Status |
|-----|--------------------------------------------------------------------|------------|--------|
| 001 | [Container orchestration: Docker Compose](#adr-0001-container-orchestration-docker-compose)                           | 2026-05-26 | Active |
| 002 | [Visual identity & CSS stack](#adr-0002-visual-identity--css-stack)| 2026-05-26 | Active |
| 003 | [Web stack: Python 3 + Django](#adr-0003-web-stack-python-3--django)| 2026-05-26 | Active |
| 004 | [Code hosting & CI/CD: GitHub + Actions](#adr-0004-code-hosting--cicd-github--actions)| 2026-05-26 | Active |
| 005 | [Control plane / Data plane split](#adr-0005-control-plane--data-plane-split)| 2026-05-26 | Active |
| 006 | [Mono-repo layout](#adr-0006-mono-repo-layout)                     | 2026-05-26 | Active |
| 007 | [uv: one project per service](#adr-0007-uv-one-project-per-service)| 2026-05-26 | Active |
| 008 | [Reverse proxy: custom Nginx with Brotli](#adr-0008-reverse-proxy-custom-nginx-with-brotli)| 2026-05-26 | Active |
| 009 | [Secret management: SOPS + age](#adr-0009-secret-management-sops--age)| 2026-05-26 | Active |
| 010 | [Configuration management: Ansible](#adr-0010-configuration-management-ansible)| 2026-05-27 | Active |

---

## ADR-0001: Container orchestration: Docker Compose

- **Date:** 2026-05-26
- **Status:** Active
- **Context:** Runtime containers on Linux VM with multi-services (Python/Nginx). Env: WSL2 in dev and single Linux VM in prod.
- **Decision:** Docker Compose for orchestration. The Portfolio is a small project, and Docker Compose lets us run in the same environments for dev and prod.
- **Alternatives:** K8s deliberately deferred to a separate learning project. Not deployed in the portfolio infra to avoid the résumé-driven complexity.
  Swarm is in maintenance mode since Mirantis acquisition (2019). Nomad is simpler that K8s but what if chosen between Nomad and K8s, the learning budget goes to K8s for career leverage.
- **Consequences:** Capitalises on existing Docker knowledge. Important
  security note: the Docker socket exposure required by the deployer service
  is a high-risk surface — see ADR-0006 and `docs/security.md`.

---

## ADR-0002: Visual identity & CSS stack

- **Date:** 2026-05-26
- **Status:** Active
- **Context:** A styled site with smooth, professional animations is
  consistently more memorable than a plain static page. The visual layer is
  part of the portfolio signal, not decoration.
- **Decision:** Tailwind CSS + HTML5, adjustable if the web framework
  decision (ADR-0003) shifts.
- **Alternatives:** Hand-written CSS — fast initial setup but generic look,
  and harder to keep consistent without a design system.
- **Consequences:** Significantly slower delivery on the visible site, but
  the artefact is more representative of current frontend practice.

---

## ADR-0003: Web stack: Python 3 + Django

- **Date:** 2026-05-26
- **Status:** Active
- **Context:** The project's positioning is DevOps, not web. The web
  framework is a vehicle, not the subject. Choosing it should optimise for
  not being a distraction.
- **Decision:** Default to Python 3 + Django + `uv`. Revisitable if another
  stack serves the project better — e.g. if real-time log streaming for the
  container deployment feature becomes painful, reconsider FastAPI + HTMX or
  stay on Django Channels.
- **Alternatives:** FastAPI (lighter, native async, less "batteries
  included"); Go + Gin (aligned with ongoing Go learning, but doubles the
  learning load on a project where web is not the focus).
- **Consequences:** Capitalises on existing Python skills. Django provides
  ORM, admin, auth, templating out of the box. Possible friction on
  real-time log streaming — will require Channels or SSE. See ADR-0006 for
  how this is mitigated by splitting the streaming concern into a separate
  service.

---

## ADR-0004: Code hosting & CI/CD: GitHub + Actions

- **Date:** 2026-05-26
- **Status:** Active
- **Context:** The portfolio must be visible to recruiters. Learning new
  tooling is itself a goal of the project.
- **Decision:** GitHub for the repository, GitHub Actions for CI/CD.
- **Alternatives:** GitLab + GitLab CI — already mastered, fast start, but
  yields no new skill and lower recruiter visibility.
- **Consequences:** New tooling to learn (Actions syntax ≠ GitLab CI), but
  denser marketplace and stronger portfolio surface.

---

## ADR-0005: Control plane / Data plane split

- **Date:** 2026-05-26
- **Status:** Active
- **Context:** The meta feature — deploying containers from the site with
  live log streaming — requires native async I/O and access to the Docker
  daemon. Mixing this with the portfolio web service (classic CRUD) would
  blur responsibilities and merge two very different security surfaces into
  one process.
- **Decision:** Two separate services communicating over HTTP API.
  - **Data plane:** `portfolio-web` (Django + Tailwind + PostgreSQL) —
    serves content, manages CV and blog.
  - **Control plane:** `container-deployer` (FastAPI async, SSE log
    streaming, Docker SDK).
- **Alternatives:** Single Django monolith with Channels — simpler to start
  but constrained on async and mixes data/control concerns. Two Django
  services — loses FastAPI's native async edge.
- **Consequences:** Clean separation of concerns. Defensible in an interview
  (the *control plane / data plane* pattern, borrowed from Kubernetes).
  Stricter security isolation around the Docker-facing service. Slight
  overhead of running both Django and FastAPI — limited because Python is
  shared.

---

## ADR-0006: Mono-repo layout

- **Date:** 2026-05-26
- **Status:** Active
- **Context:** Need to visually separate runtime code (services) from
  supporting concerns (infra, docs), while keeping everything in a single
  repository.
- **Decision:** Mono-repo with the following top-level layout:
  - `services/portfolio-web/` (Django)
  - `services/container-deployer/` (FastAPI)
  - `infra/` (docker-compose, nginx, terraform)
  - `docs/` (architecture, deployment, security)
  - `.github/workflows/` (CI/CD)
- **Alternatives:** Polyrepo (one repo per service) — common in large
  organisations (Amazon, Netflix), but adds coordination overhead with no
  payoff for a personal project. Services at the repository root, no
  `services/` folder — fine while there is little else, gets messy as soon
  as `infra/` and `docs/` appear.
- **Consequences:** Single `git clone`, unified CI/CD filtered by `paths:`
  triggers, easy cross-service search. More complex permissions management
  if the project later becomes public and collaborative.

---

## ADR-0007: uv: one project per service

- **Date:** 2026-05-26
- **Status:** Active
- **Context:** Each service will be built into its own Docker image with its
  own dependency closure.
- **Decision:** Each `services/<name>/` directory owns its own
  `pyproject.toml`, `uv.lock`, `.python-version` and `.venv`.
- **Alternatives:** `uv workspace` — single root-level `uv.lock` with
  multiple member `pyproject.toml` files. Useful if a shared library gets
  extracted later. Revisit if that need emerges.
- **Consequences:** Strong isolation. Smaller Docker images. No accidental
  cross-service coupling. Slight overhead managing two local environments
  during development — mitigated by a single `Makefile` target that syncs
  both at once.

---

## ADR-0008: Reverse proxy: custom Nginx with Brotli

- **Date:** 2026-05-26
- **Status:** Active
- **Context:** Likely need to serve Brotli-compressed WebGL/WebGPU assets;
  the `ngx_brotli` module is not in the official Nginx image.
- **Decision:** Custom Nginx image (locally built) with `ngx_brotli`,
  TLS via certbot. Single reverse proxy in front of both services.
- **Alternatives:** Stock Nginx without Brotli — less efficient asset
  compression. Caddy — Brotli included, simpler config, but less
  representative of professional stacks. Traefik — auto-discovery oriented,
  overkill here.
- **Consequences:** Full control over the config. Slightly longer Docker
  build. Consistent with existing Nginx experience.

---

## ADR-0009: Secret management: SOPS + age

- **Date:** 2026-05-26
- **Status:** Active
- **Context:** Need to manage secrets (API keys, DB credentials,
  certificates). Explicit learning goal: move past `.env` files and GitLab
  CI variables toward a workflow representative of current pro practice.
- **Decision:** Adopt **SOPS** (Mozilla) + **age** (modern alternative to
  GPG) to encrypt secrets and commit the encrypted versions to Git. Decrypt
  on demand at deployment time. GitOps-friendly.
- **Alternatives:** 1Password CLI — useful as a complement for local
  scripts, not as the primary store. HashiCorp Vault — oversized for this
  project. AWS Secrets Manager / Azure Key Vault — cloud-bound.
- **Consequences:** Workflow more representative of professional practice.
  Initial learning curve. `age` keys must be protected locally — ideally
  inside a personal vault.

---

## ADR-0010: Configuration management: Ansible

- **Date:** 2026-05-27
- **Status:** Active
- **Context:**  The VM requires initial setup and must be re-configurable without full reprovisioning.
- **Decision:** Altough Ansible isn't the best solution in this case, it is widely adopted by the companies and this project is an opportunity to build that skill. Ansible is robust and exist since 2013.
- **Alternatives:** Chef and Puppet are less widely used by DevOps teams and the overhead is too much for a little projet like that. cloud-init is a correct choice, but not idempotent and prefer to learn Ansible
- **Consequences:** It ensures configuration consistency (state reconvergence) and facilitates reproducibility (idempotent). Learning courbe on the new tools is important and use Ansible on one VM is inabituel