# Downstream Fork Workflow

This guide documents a practical workflow for teams maintaining a downstream fork while preserving the ability to regularly ingest upstream changes.

## Goals

1. Keep `main` in your fork close to upstream.
2. Isolate environment-specific rollout work from `main`.
3. Ensure submodule commits referenced by parent repositories are always available remotely.

## Branch Strategy

Use these branch types in the fork repository:

1. `main`
   - Tracks upstream as closely as possible.
   - Avoid stacking environment-specific rollout changes directly here during active migrations.
2. `rollout/<initiative>`
   - Long-lived integration branch for a live migration window.
   - Example: `rollout/k3s-setup-2026q1`.
3. `feat/*`, `fix/*`, `chore/*`
   - Short-lived topic branches.
   - Open PRs into the active `rollout/<initiative>` branch.

## First-Time Setup (Controller)

Run from this repository root:

```bash
cd infra/roles/ansible-role-k3s
```

Ensure remotes:

```bash
git remote -v
# origin should point to your fork
# add upstream once if missing
git remote add upstream https://github.com/PyratLabs/ansible-role-k3s.git
```

Fetch all refs:

```bash
git fetch --all --prune
```

Sync fork `main` with upstream:

```bash
git switch main
git rebase upstream/main
git push origin main
```

## Start or Continue a Rollout Branch

Create rollout branch from the currently validated branch or commit:

```bash
git switch <validated-branch>
git switch -c rollout/k3s-setup-2026q1
git push -u origin rollout/k3s-setup-2026q1
```

If the rollout branch already exists:

```bash
git switch rollout/k3s-setup-2026q1
git pull --ff-only
```

## Day-to-Day Development Flow

1. Create topic branch from rollout branch:

```bash
git switch rollout/k3s-setup-2026q1
git pull --ff-only
git switch -c fix/<topic>
```

2. Implement, validate, and commit:

```bash
# example validation
ansible-playbook -i <inventory> <playbook> --syntax-check

git add -A
git commit -m "fix: <summary>"
```

3. Push and open PR into rollout branch:

```bash
git push -u origin fix/<topic>
```

4. Merge PR into `rollout/k3s-setup-2026q1`.

## Parent Repository Submodule Pointer Flow

When parent repositories consume this role as a submodule, always pin the tested commit and commit the pointer update in the parent.

From the parent repository root:

```bash
cd /path/to/parent-repo

git submodule update --init --recursive
git -C infra/roles/ansible-role-k3s fetch --all --prune
git -C infra/roles/ansible-role-k3s switch rollout/k3s-setup-2026q1
git -C infra/roles/ansible-role-k3s pull --ff-only

git add infra/roles/ansible-role-k3s
git commit -m "chore(submodule): bump ansible-role-k3s to <sha>"
```

## Upstream Ingestion During Rollout

Periodically ingest upstream changes:

```bash
cd infra/roles/ansible-role-k3s
git fetch upstream --prune

git switch main
git rebase upstream/main
git push origin main

git switch rollout/k3s-setup-2026q1
git rebase main
# resolve conflicts if needed
git push --force-with-lease origin rollout/k3s-setup-2026q1
```

If your team prefers no history rewriting on rollout branches, use merge instead of rebase.

## After Rollout Stabilizes

1. Decide what should remain fork-specific versus proposed upstream.
2. For generic fixes, open upstream PRs from clean topic branches based on `main`.
3. Merge `rollout/<initiative>` into fork `main` only when the rollout is stable.
4. Tag a release point in the fork if parent repositories depend on the stabilized behavior.

## Operational Guardrails

1. Never leave parent repos pinned to submodule commits that are not pushed.
2. Do not run long-lived rollout work directly on fork `main`.
3. Keep each PR scoped to one concern (safer rollback and review).
4. Run at least syntax checks before updating a submodule pointer in parent repos.
