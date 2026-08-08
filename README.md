# Recipes

A **recipe** is a `.ian` script that runs a sequence of `isann` commands. It is
how a multi-step setup - unlock, start the backend, wait for it, pull a model,
start an engine - becomes one reproducible command.

```
examples/<name>.ian     one file = one recipe (this repo)
        |
        | isann recipe pull
        v
<install-root>/artifacts/addon/recipes/<name>.ian
```

This repo has two folders:

| Folder | What is in it |
|---|---|
| `examples/` | Recipes meant to be used. Start here. |
| `test/` | Conformance tests for the recipe language. **Many fail on purpose** - see below. |

## File format

This is `examples/llama-start.ian`, with its header comment trimmed:

```ian
#pragma ISANN 0.1.20
# llama-start.ian - start the llama (coding) engine from a cold node.

requires:
  vram: 4G+

auth unlock --account me;
mesh start provider;
docker warmup;                # fire WSL + docker boot (async)
docker wait;                  # block until the docker backend is running
docker start llama;
docker wait --engine llama;   # block until llama's HTTP endpoint responds
```

**The first line must be `#pragma ISANN <version>`.** A file without it is
refused by the loader - that is the version gate, not a comment.

An optional `# name:` / `# author:` / `# description:` / `# version:` header
carries metadata shown by `isann recipe list`. `examples/inproc-smoke.ian` and
`examples/secure-unlock.ian` use it:

```ian
#pragma ISANN 0.1.20
# name: secure-unlock
# author: iSANN core
# description: prompt for the node's alias + passphrase safely, then boot the node
# version: 1.0.0
```

`requires:` is an optional precondition block checked before anything runs:

```ian
requires:
  vram: 4G+
  gpu: RTX 30 | RTX 40 | GTX 16
```

Every statement ends with `;`. Comments start with `#`.

## Language

### Variables

```ian
var engine = llama;                  # constant, no subprocess
var ttl = 60s;
var greeting = hello world;          # multi-token -> joined into one string
var combined = ${engine}-${ttl};     # expansion happens before storing
```

### Capture

`name := <command>` runs the command and stores its result. With `-json` the
result is a structure you can walk with `${path}`.

```ian
info := info -json;
echo "node=${info.node_id}";
echo "gpu=${info.hardware.gpu.name}";
```

(from `examples/inproc-smoke.ian` and `test/inproc.ian`)

Commands registered for in-process dispatch run inside the same `isann` process
(no fork), and return native values straight into memory. `isann recipe exec
-fork <file>` forces the subprocess path instead - both must produce identical
output.

### Environment variables

```ian
echo "env user=${env.ISANN_RECIPE_TEST_USER}";
require ${env.ISANN_RECIPE_TEST_USER}, "ISANN_RECIPE_TEST_USER required";
```

`env` is a reserved namespace, so a variable of your own named `env` cannot
shadow the OS lookup. Values are flat strings - `${env.PATH.sub}` is an error,
not an empty value.

### Assertions

```ian
require ${status} == ready, "status must be ready";
require ${status} != booting, "must not be booting";
require ${node.node_id}, "node must be registered";   # truthy check
assert ${status} == ready;
```

A failed `require` aborts the recipe with its message and a non-zero exit.

### Include

```ian
include "fragment.ian";     # path relative to THIS file
```

The fragment's statements are inlined at parse time and share the same memory,
so a variable it captures is visible afterwards. A fragment has no `requires:`
block of its own.

### Reading input

```ian
alias := func read "account alias: ";
pass  := func read -secret "passphrase: ";     # echo suppressed on a TTY
auth unlock --account ${alias};
```

`read` is a function, reached through the `func` namespace. This is how one
recipe serves several nodes without hard-coding a per-node alias in plain text.

For unattended runs, put the passphrase in the environment instead and drop the
`read` line - `auth unlock` inherits `ISANN_PASSPHRASE`.

## The policy gate - read this before your first run

A recipe is a script driving the operator's own CLI. A recipe from the market is
therefore **untrusted code with your command line in its hands**, so a node
starts with the gate **closed** and the operator opens what a recipe needs.

A fresh node refuses everything, including read-only commands:

```
[1/9] x recipe policy: `version -json` is blocked - default deny (0s)
isann: recipe exec: inproc-smoke.ian:18: ver := version -json -
       recipe policy: `version -json` is blocked - default deny
```

Nothing is wrong. Nothing has been configured yet.

### How the gate decides

```bash
isann policy list --rule recipe
```
```
   (pre)  deny   auth.mode                 immutable - no rule can override
      -    (no operator rules)
   (def)  deny   - nothing matched
```

Three layers, checked **top to bottom, first match wins**:

| Layer | What it is | Can you change it |
|---|---|---|
| `(pre)` | pre-chain. Checked **before** your rules | **no** - compiled in |
| `1. 2. 3. ...` | your rules, in order | yes - `policy add` / `rm` / `move` |
| `(def)` | the answer when **nothing above matched** | yes - `--default` |

On a fresh node the middle layer is empty, so every command falls through to
`(def) deny`. That is why even `version` is refused.

### Reading a command as a pattern

Rules name commands as `namespace.verb`:

```
isann auth mode      ->  auth.mode
isann docker rm      ->  docker.rm
isann model pull     ->  model.pull
```

| Pattern | Covers |
|---|---|
| `docker.rm` | that one verb |
| `docker` | the whole namespace (`docker.*`) |
| `*.rm` | `rm` in every namespace |
| `*` | everything (a catch-all - anything below it is dead, and `policy list` marks it UNREACHABLE) |

### `(pre)` - the one thing nobody can open

```
(pre)  deny   auth.mode   immutable - no rule can override
```

`isann auth mode` switches the node's inference door between `public` (anonymous
callers allowed) and `protected`. A recipe that flipped it could open your node
silently - and **snapshot cannot undo it**, because `auth mode` writes
`conf/isannd.json`, which the snapshot scope deliberately excludes.

Every other reachable mutation lands under `artifacts/` and is recoverable. That
is the bar for the pre-chain, and it is why the list has exactly one entry.

`--allow auth.mode` is refused rather than accepted-and-ignored, and
`--default allow` does not reach it either. Calling `isann auth mode` yourself
from a shell is unaffected - only from inside a recipe.

## When a recipe is refused

### 1. Ask what it needs, without running it

```bash
isann recipe exec inproc-smoke.ian -dry-run
```

`-dry-run` prints the plan and **preflights every statement against the gate**,
listing only the ones that would be refused. One command tells you the whole
allow-list instead of discovering it one failure at a time.

It checks statements inside `IF` blocks too, without evaluating the condition -
warning about a line that may not run is cheaper than staying silent about one
that does.

### 2. Open what it needs

```bash
isann policy add --rule recipe --allow "version,info,list.nodes"
isann policy list --rule recipe
```
```
   (pre)  deny   auth.mode                 immutable - no rule can override
      1   allow  version
      2   allow  info
      3   allow  list.nodes
   (def)  deny   - nothing matched
```

Quote the patterns - `*` is a shell glob.

### Or open everything (a node where you only run your own recipes)

```bash
isann policy add --rule recipe --default allow
```

This does not add a rule. It changes the **fallback**: "nothing matched" now
means allow instead of deny.

```
before:  no rules  +  default deny   ->  everything refused
after:   no rules  +  default allow  ->  everything passes (except the pre-chain)
```

Then block the dangerous ones, remembering that **order decides**:

```bash
isann policy add --rule recipe --default allow
isann policy add --rule recipe --deny "*.rm" --at 1
```

`--at 1` puts the deny at the top. Without it the rule is appended below, and
`--default allow` never gets consulted for `*.rm` anyway - but a later `allow`
rule above it would win. Use `policy move <pattern> --to <N>` to reorder.

Back to a closed node:

```bash
isann policy add --rule recipe --default deny
```

### What is never gated

- **Builtins** - `echo`, `var`, `sleep`, `func read`. They touch only the
  recipe's own variables, so there is nothing to gate.
- **Everything else goes through the chain.** There is no read-only exemption.
  An earlier version waved `ls`/`info` through on the grounds that reads cannot
  damage a node; that was wrong twice - it made "deny by default" a half-truth
  the operator never asked for, and reads are not uniformly harmless
  (`cred list` enumerates credential names, `list nodes` exposes peer topology
  and owner addresses).

## What recipes may not do at all

Separately from the policy chain, recipe-management commands are refused inside
a recipe. No policy opens these:

| Blocked | Why |
|---|---|
| `recipe exec` / `recipe ls` | stops a recipe fanning out into more recipes |
| `recipe rm` | data-loss vector |
| `recipe pull` | chain-fetch attack (A pulls B pulls C ...) |
| `market install` | same chain-attack shape - use `app pull <url>` for files |

The runtime refuses them before dispatch, so the recipe aborts with a clear
error rather than doing the damage.

## Install and run

Paste the address straight from the GitHub page - both shapes work:

```bash
# a FOLDER: installs every .ian in it (one file = one recipe)
isann recipe pull https://github.com/isannai/recipes/tree/main/examples

# a FILE: installs just that one
isann recipe pull https://github.com/isannai/recipes/blob/main/examples/llama-start.ian
```

The ref is pinned to its commit before anything downloads, so a folder arrives
as one snapshot. Files that are not `.ian` are reported as skipped rather than
silently ignored, and sub-folders are not descended into.

```
recipe from https://github.com/isannai/recipes/tree/<commit-sha>/examples

NAME               RESULT
inproc-smoke       ok
interactive-smoke  ok
llama-start        ok
...

7 installed, 0 skipped, 0 failed
```

`--name` applies to a **single-file** source only; a folder takes each name from
its own file.

Then run:

```bash
# a stored recipe by name
isann recipe exec llama-start

# a file directly (no install)
isann recipe exec ./llama-start.ian

# with the passphrase supplied non-interactively
ISANN_PASSPHRASE=... isann recipe exec llama-start
```

A bare name resolves from the store; anything containing a path separator or
ending in `.ian` is read from disk.

### Re-running

`recipe pull` is idempotent: an already-installed recipe is skipped with its
reason and exits 0. Add `-force` to overwrite.

```
NAME         RESULT
llama-start  skip - already installed - artifacts/addon/recipes/llama-start.ian
```

```bash
isann recipe list             # what is installed, with content ids and source
isann recipe inspect <name>   # raw .ian + provenance
isann recipe rm <name>
```

## Writing a recipe that re-runs safely

Every `isann` mutation command is idempotent by design: already installed means
`[skip]` and exit 0, not an error. Lean on that instead of guarding by hand -
a recipe should be safe to run twice.

Two ordering rules matter in practice:

1. **`docker warmup` is asynchronous.** It fires the WSL + docker boot and
   returns immediately. Without a following `docker wait`, the next
   `docker start` races ahead and fails with "no docker backend available".
2. **A started container is not a ready engine.** `docker wait --engine <e>`
   blocks until the engine's HTTP probe actually responds. Run it before the
   first inference.

## `examples/` - the recipes here

| File | What it does | Needs isannd |
|---|---|---|
| `llama-start.ian` | Cold node to a serving llama engine | yes |
| `sd-bootstrap.ian` | Pull the SD base model, apply a profile, start | yes |
| `wait-llama.ian` | Wait until llama is genuinely ready | yes |
| `secure-unlock.ian` | Prompt for alias + passphrase, then boot | yes |
| `inproc-smoke.ian` | In-process dispatch, capture, `var` | yes |
| `interactive-smoke.ian` | `func read` and `${env.X}` | no |
| `smoke-vars.ian` | Variable expansion only | no |

## `test/` - conformance tests

These exercise the recipe language itself. **Nine of them are negative tests:
passing means the runner REFUSES them.** Do not read a non-zero exit here as a
broken recipe.

| File | Expected | Checks |
|---|---|---|
| `set.ian` | exit 0 | `var` assignment, `${path}` expansion |
| `inproc.ian` | exit 0 | in-process dispatch + capture |
| `include.ian` | exit 0 | `include` shares memory (pulls in `fragment.ian`) |
| `include-cmd.ian` | exit 0 | an included fragment runs real commands (`fragment-probe.ian`) |
| `require.ian` | exit 0 | `require` / `assert` pass path |
| `require-probe.ian` | exit 0 | probe a command, then assert on a field |
| `read.ian` | exit 0 | `func read` stdin capture |
| `read-capture.ian` | exit 0 | `name := func read ...` |
| `env.ian` | exit 0 | `${env.X}` lookup |
| `require-fail.ian` | **exit != 0** | a failed `require` aborts |
| `env-fail.ian` | **exit != 0** | `${env.X.sub}` has no nested traversal |
| `read-empty.ian` | **exit != 0** | bare `read` (no `func`) is rejected |
| `func-unknown.ian` | **exit != 0** | `func` only holds functions, not namespaces |
| `pragma-missing.ian` | **exit != 0** | no `#pragma ISANN` line -> refused |
| `docker-wait-timeout.ian` | **exit != 0** | `docker wait --engine` times out and aborts |
| `blocked-recipe-ls.ian` | **exit != 0** | `recipe ls` blocked inside a recipe |
| `blocked-recipe-rm.ian` | **exit != 0** | `recipe rm` blocked |
| `blocked-recipe-pull.ian` | **exit != 0** | `recipe pull` blocked |

`fragment.ian` and `fragment-probe.ian` are not run directly - they are included
by the two `include*` tests.

Tests whose name ends in `-fail`, starts with `blocked-`, or that print
`UNREACHABLE` are the negative ones: if you ever see `UNREACHABLE` in the output,
the guard did not fire and that IS a bug.

Some tests need input or environment:

```bash
printf 'alice\nblue\n' | isann recipe exec ./test/read.ian
ISANN_RECIPE_TEST_USER=daesob isann recipe exec ./test/env.ian
```

## Publish to the hub

```bash
isann recipe push    llama-start --version 1.0.0 --summary "cold start to serving"
isann recipe publish llama-start
```

`push` uploads as private; `publish` makes it public. Pass `-install` to mark a
recipe as an installer package. Run `isann auth unlock` first - the upload is
signed with your owner wallet.
