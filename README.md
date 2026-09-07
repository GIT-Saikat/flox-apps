# Flox Apps

Flox Apps is a Bun-managed Turborepo containing the web application, TypeScript
services, a Python FastAPI agent service, and shared packages.

## Repository layout

```text
apps/
	agent/       Python FastAPI service managed with uv
	backend/     TypeScript/Bun backend service
	web/         Next.js web application
packages/
	db/          Prisma client and PostgreSQL configuration
	ui/          Shared React UI components
	eslint-config/
	typescript-config/
```

The `backend` and database entry points are currently minimal Bun/TypeScript
services. The implemented agent service is the Python application in
`apps/agent`.

## Prerequisites

- Bun 1.4 or later
- Python 3.14 or later
- [uv](https://docs.astral.sh/uv/)
- PostgreSQL, when using the database package

The root workspace uses Bun. The agent has its own Python environment and lock
file, so JavaScript and Python dependencies are installed separately.

## Installation

From the repository root:

```sh
bun install
cd apps/agent
uv sync
cd ../..
```

If `bun` resolves to a Windows executable while working in a Linux environment,
install and use the native Linux Bun binary. Verify with:

```sh
which bun
bun --version
```

## Development

Start the Next.js web application:

```sh
bun run dev
```

The web application is available at [http://localhost:3000](http://localhost:3000).

Start the FastAPI agent in a second terminal:

```sh
cd apps/agent
uv run fastapi dev main.py
```

The agent currently exposes the FastAPI application defined in
`apps/agent/main.py`.

## Checks and builds

Run the repository tasks through Turborepo:

```sh
bun run build
bun run lint
bun run check-types
```

Format TypeScript, TSX, and Markdown files with:

```sh
bun run format
```

For a web-only task, use the workspace filter:

```sh
bun run --filter web build
bun run --filter web lint
bun run --filter web check-types
```

Python dependencies can be checked from the agent directory with:

```sh
cd apps/agent
uv run python -m compileall main.py
```

## Database

The database package uses Prisma 7 with PostgreSQL. Its schema is in
`packages/db/prisma/schema.prisma`, and the generated client is configured for
`packages/db/generated/prisma`.

Configure the PostgreSQL connection required by your Prisma setup before
running database commands. Prisma commands should be run from `packages/db`.

## Workspace commands

The root `package.json` provides these scripts:

- `bun run dev` runs the Turbo development task.
- `bun run build` builds packages with a `build` task.
- `bun run lint` runs lint tasks.
- `bun run check-types` runs TypeScript type checks.
- `bun run format` formats TypeScript, TSX, and Markdown files.

Turbo task configuration is in `turbo.json`. The Python agent is intentionally
run with `uv` because it is not a Bun workspace package.
