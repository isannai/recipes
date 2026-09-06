# install-sd-small

Turns a fresh node into one that generates images. Stable Diffusion 1.5 at
512x512 through sd.cpp, sized for a 4GB-class GPU.

## What it does

1. Asks for the account alias and unlocks the wallet
2. Applies the `sd/small` profile, which sets `ARCH=sd15` and points `MODEL` at the checkpoint
3. Downloads v1-5-pruned-emaonly (about 4GB) from HuggingFace
4. Wakes WSL and dockerd, then creates and starts the sd container
5. Starts station and marks it to autostart
6. Asks for a rendezvous URL, saves it, and registers the node there

## Requirements

`vram: 4G+`, checked before the download starts.

## Two fields have to agree

SD keeps models per architecture, so the pull and the profile both carry it:

```
model pull ... --engine sd --kind model --arch sd15 --name v1-5-pruned-emaonly
                                        ARCH ┘             MODEL ┘
```

Change `ARCH` and the whole assembled view swaps to that architecture's tree, so
a mismatch here means the container starts and then cannot find the checkpoint.
The recipe sets both, which is the point of using it.

## Before you open this one publicly

An image node is a different exposure than a text node. The prober grades text
only, so serving images earns nothing from the faucet, and opening image
generation to anonymous callers is a decision worth making deliberately. The
node ships protected; leave it that way unless you meant otherwise.

## Install

```
isann recipe pull <this asset>
isann recipe exec install-sd-small
```

It is fetched, not run. Read it first if you like: a recipe drives your own CLI.

## See also

- `install-llama-small` / `install-llama-medium` - serve text instead
- `install-passenger` - a node that only calls others

## Source

https://github.com/isannai/recipes
