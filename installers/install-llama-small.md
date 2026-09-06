# install-llama-small

Turns a fresh node into one that serves text generation. Qwen2.5-1.5B on
llama.cpp, sized for a 4GB-class GPU.

## What it does

1. Asks for the account alias and unlocks the wallet
2. Applies the `llama/small` profile, which points `MODEL` at the file step 3 fetches
3. Downloads Qwen2.5-1.5B-Instruct-Q4_K_M (about 940MB) from HuggingFace
4. Wakes WSL and dockerd, then creates and starts the llama container
5. Starts station and marks it to autostart, so the node can answer
6. Asks for a rendezvous URL, saves it, and registers the node there

After it finishes the node is reachable and serving.

## Requirements

`vram: 4G+`. The recipe checks this before it starts, so a node that cannot run
the model is turned away rather than left half-installed.

## Why the profile comes before the download

`profile use` writes the engine's `.env`, and `MODEL` in that file has to match
the folder `model pull --name` creates. Applying the profile first means the two
agree by construction instead of by luck.

## Install

```
isann recipe pull <this asset>
isann recipe exec install-llama-small
```

It is fetched, not run. Look at the file before executing it if you like: a
recipe drives your own CLI, so it can do anything you can.

## See also

- `install-llama-medium` - Qwen2.5-14B, needs 12G+
- `install-sd-small` - image generation instead of text
- `install-passenger` - a node that only calls others

## Source

https://github.com/isannai/recipes
