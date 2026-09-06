# install-llama-medium

The same shape as `install-llama-small`, one size up. Qwen2.5-14B on llama.cpp,
for a 12GB-class GPU.

## What it does

1. Asks for the account alias and unlocks the wallet
2. Applies the `llama/medium` profile (larger context, `MODEL` pointed at the 14B file)
3. Downloads Qwen2.5-14B-Instruct-Q4_K_M (about 8.4GB) from HuggingFace
4. Wakes WSL and dockerd, then creates and starts the llama container
5. Starts station and marks it to autostart
6. Asks for a rendezvous URL, saves it, and registers the node there

## Requirements

`vram: 12G+`, checked before anything is downloaded. On a smaller card use
`install-llama-small` instead; 8.4GB is a long download to discover it will not
load.

## What differs from small

Only the profile and the model. The 14B answers noticeably better and is
noticeably slower, and it holds a larger context. Everything else about the node
is identical, so switching later is a profile swap plus a restart rather than a
reinstall.

## Install

```
isann recipe pull <this asset>
isann recipe exec install-llama-medium
```

It is fetched, not run. Read it first if you like: a recipe drives your own CLI.

## See also

- `install-llama-small` - Qwen2.5-1.5B, needs 4G+
- `install-sd-small` - image generation instead of text
- `install-passenger` - a node that only calls others

## Source

https://github.com/isannai/recipes
