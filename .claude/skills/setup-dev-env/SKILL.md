---
name: setup-dev-env
description: Set up the development environment for this project. Triggered by phrases like "setup env", "set up environment", "setup dev environment", "initialize environment", "bootstrap project", "get this running", "prepare dev env", or "configure environment".
allowed-tools: Read, Glob, Grep, Bash, Write, Edit
---

# Set Up Dev Environment

Inspect the repository to determine what kind of project this is, then set up the development environment end-to-end.

## Steps

1. **Detect project type** by checking for:
   - `pyproject.toml`, `requirements.txt`, `setup.py` → Python
   - `package.json` → Node.js / JavaScript / TypeScript
   - `Cargo.toml` → Rust
   - `go.mod` → Go
   - `Makefile` → check targets for setup instructions

2. **Set up the environment** based on what you find:

   **Python:**
   - Prefer `uv` if available: `uv venv && uv pip install -e ".[dev]"`
   - Fall back to: `python3 -m venv .venv && .venv/bin/pip install -e ".[dev]"` or `pip install -r requirements.txt`

   **Node.js:**
   - Run `npm install` (or `yarn` / `pnpm install` depending on lockfile present)

   **No project files yet (early-stage repo):**
   - Ask the user what language/stack they plan to use
   - Scaffold the appropriate project structure (pyproject.toml for Python, package.json for Node, etc.)
   - Set up `.gitignore` appropriate for the chosen stack

3. **Verify the setup** by running whatever check makes sense (e.g., `python --version`, `node --version`, import a key package, run a smoke test).

4. **Report** what was set up, any issues encountered, and what the user should do next.

## Notes

- This is a Bitcoin algorithmic trading/analysis project. If scaffolding from scratch, default to Python with these likely dependencies: `pandas`, `numpy`, `requests`, `python-dotenv`, and suggest `ccxt` for exchange connectivity.
- Never commit `.venv/`, `node_modules/`, or `.env` files.
- If a `.env.example` exists, copy it to `.env` and prompt the user to fill in API keys.
