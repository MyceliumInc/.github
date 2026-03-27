# Mycelium — Setup Guide

## Prerequisites

- Git
- Bun (or Node)
---

## 1. Clone the monorepo

Always clone with `--recurse-submodules` to pull in all three repos at once:

```bash
git clone --recurse-submodules git@github.com:MyceliumInc/Mycelium.git
cd Mycelium
```

If you already cloned without it, run:

```bash
git submodule update --init --recursive
```

---

## 2. Set up your `.env`

Create a `.env` file at the monorepo root from .env.example, filling required values.

```bash
cp .env.example .env
```

Now symlink it into each submodule so they all share the same env:

```bash
for repo in Agent MCP Site; do
  ln -sf "$(pwd)/.env" "$repo/.env"
done
```

> The symlinks are gitignored in each submodule — never commit them.

---

## 3. Install dependencies

```bash
bun install
```

---

## 4. Set up Git hooks

There are two hooks to install. Both live locally and are not committed to git.

### Hook 1 — Auto-bump from a submodule push

When you push from inside `Agent`, `MCP`, or `Site`, this hook automatically updates the monorepo's reference to point at your latest commit.

Run from the **monorepo root**:

```bash
for repo in Agent MCP Site; do
  cat > $repo/.git/hooks/post-push << EOF
#!/bin/sh
cd ..
git submodule update --remote --merge
git add .
git commit -m "AutoBump $repo to Latest"
git push
EOF
  chmod +x $repo/.git/hooks/post-push
done
```

### Hook 2 — Auto-sync submodules on monorepo push

When you push from the monorepo root, this hook stages, commits, and pushes any pending changes in all three submodules first.

```bash
cat > .git/hooks/pre-push << 'EOF'
#!/bin/sh
for repo in Agent MCP Site; do
  echo "Syncing $repo..."
  cd $repo
  git add .
  git diff --staged --quiet || git commit -m "AutoSync $repo"
  git push
  cd ..
done
EOF
chmod +x .git/hooks/pre-push
```

---

## 5. Verify everything

Check hooks are in place and executable:

```bash
ls -la Agent/.git/hooks/post-push MCP/.git/hooks/post-push Site/.git/hooks/post-push .git/hooks/pre-push
```

Check submodules are initialised:

```bash
git submodule status
```

Each line should start with a commit hash, not a `-` (which would mean uninitialised).

---

## How submodules work here

The monorepo pins each submodule to a specific commit SHA — this is the `@ abc1234` you see on GitHub. It does **not** auto-track `main`.

- **Pushing from a submodule** triggers the `post-push` hook, which bumps the monorepo's ref automatically.
- **Pushing from the monorepo** triggers the `pre-push` hook, which syncs all submodules first.
- If you ever need to manually bump all submodules to their latest: `git submodule update --remote --merge`

---

## Repo structure

```
Mycelium/
├── .env                  ← shared env (never committed)
├── .gitmodules           ← submodule registry
├── .github/
├── Agent/                ← MyceliumInc/Agent (submodule)
├── MCP/                  ← MyceliumInc/MCP (submodule)
├── Site/                 ← MyceliumInc/Site (submodule)
└── ...etc.
```
