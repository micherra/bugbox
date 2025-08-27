# Tech Baseline

## Runtime Versions

- Python 3.13.6
- Node.js 24.5.0
- Terraform 1.12.2

## Tooling

The Python package in `bugbox/` uses [uv](https://github.com/astral-sh/uv) for environment management, locking, building, and publishing.

## Folder Intent

- `bugbox/` – PyPI package (CLI, decorator, core logic)
- `frontend/` – Next.js site (later)
- `backend/` – API layer (infra-agnostic; Cloudflare Worker today)
- `context/` – internal project docs

## Local Development

Sync the environment in `bugbox/` using `uv sync`.

## Continuous Integration

CI will run Python 3.13 and use uv within `bugbox/`.

## Versioning

Versioning policy will be defined in a later ticket.

