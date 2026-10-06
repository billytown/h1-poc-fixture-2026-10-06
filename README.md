# Security PoC — `shopify theme init` writes outside its destination directory

**This repository is a proof of concept for a HackerOne report against
`@shopify/cli`. It is not a theme. Do not use it as one.**

Reported by `billyposseidon` (HackerOne) on 2026-10-06, against
`@shopify/cli@4.8.5`.

## What it contains

Two of the three filenames that `shopify theme init` writes into the root of the
cloned theme are committed here as **symbolic links** pointing outside the
directory that the CLI creates:

```
AGENTS.md  ->  ../canario_agents.txt
CLAUDE.md  ->  ../canario_claude.txt
```

`git ls-files -s` shows both with mode `120000`, which is how git stores a
symlink. A normal `git clone` materialises them.

## Why that matters

`packages/theme/src/cli/services/init.ts` writes `AGENTS.md` with `writeFile`,
then tries to `symlink()` the other two names to it. When a name is already
taken the `symlink()` call fails with `EEXIST`, and the `catch (error)` branch —
written for Windows, where symlinks need Developer Mode — falls back to
`writeFile` on that same path. `writeFile` from `fs/promises` opens with
`O_TRUNC` and **follows symbolic links**, so both writes land wherever the links
point. Nothing in the path calls `lstat` or `realpath`, and the cleanup in
`services/init.ts:54-60` only runs for Shopify's own Skeleton theme URL.

## What the links point at

Relative paths that resolve to the **parent of the clone**, and nothing else.
Running the command against this repository creates or overwrites two files
named `canario_agents.txt` and `canario_claude.txt` next to the directory the
CLI just created. No system path, no home directory, no file that belongs to
anything else. The point is to show that the write escapes the destination, not
to damage anything.

An absolute path works the same way and is what makes this reachable without
knowing the victim's working directory; that variant is demonstrated in the
report rather than here.

## Reproducing

```sh
npm install --ignore-scripts --prefix ./install @shopify/cli@4.8.5
mkdir work && cd work
node ../install/node_modules/@shopify/cli/bin/run.js theme init mytheme \
  --clone-url https://github.com/billytown/h1-shopify-cli-theme-init-symlink-2026-10-06.git
# answer the "Which LLM instruction file" prompt with anything but Skip
ls -l mytheme/AGENTS.md     # still a symlink: it was written THROUGH
cat canario_agents.txt      # now holds Shopify's AGENTS.md content
```

A terminal is required: without a TTY the command returns before the AI
instructions step and the sinks are never reached.

## Disclosure

This repository exists only to make the report reproducible. It will be deleted
once the report is resolved.
