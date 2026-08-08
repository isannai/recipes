# Recipes

A **recipe** is a `.ian` script that runs a sequence of `isann` commands. It is
how a multi-step setup - unlock, start the backend, wait for it, pull a model,
start an engine - becomes one reproducible command.

```
example/<name>.ian     one file = one recipe (this repo)
        |
        | isann recipe pull
        v
<install-root>/artifacts/addon/recipes/<name>.ian
```

This repo has two folders:

| Folder | What is in it |
|---|---|
| `example/` | Recipes meant to be used. Start here. |
| `test/` | Conformance tests for the recipe language. **Many fail on purpose** - see below. |

## File format

This is `example/llama-start.ian`, with its header comment trimmed:

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
carries metadata shown by `isann recipe list`. `example/inproc-smoke.ian` and
`example/secure-unlock.ian` use it:

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

(from `example/inproc-smoke.ian` and `test/inproc.ian`)

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

## What recipes may not do

Recipe management commands are blocked when called from inside a recipe:

| Blocked | Why |
|---|---|
| `recipe exec` / `recipe ls` | stops a recipe fanning out into more recipes |
| `recipe rm` | data-loss vector |
| `recipe pull` | chain-fetch attack (A pulls B pulls C ...) |

The runtime refuses them before dispatch, so the recipe aborts with a clear
error rather than doing the damage.

## Install and run

Use the **raw** URL. A `github.com/.../blob/...` (or `/tree/...`) address serves
an HTML page, not the file.

```bash
# from this repo
isann recipe pull \
  https://raw.githubusercontent.com/isannai/recipes/main/example/llama-start.ian

# pin to a commit so the content can never change under you
isann recipe pull \
  https://raw.githubusercontent.com/isannai/recipes/<commit-sha>/example/llama-start.ian

# run a stored recipe by name
isann recipe exec llama-start

# run a file directly (no install)
isann recipe exec ./llama-start.ian

# with the passphrase supplied non-interactively
ISANN_PASSPHRASE=... isann recipe exec llama-start
```

A bare name resolves from the store; anything containing a path separator or
ending in `.ian` is read from disk.

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

## `example/` - the recipes here

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
