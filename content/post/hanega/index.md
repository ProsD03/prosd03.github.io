---
title: "NeoHanega: Why you should do research before going down a rabbit hole"
description: The story of a failed project, and a memento for the future.
date: 2026-03-28
image: cover.jpg
categories:
    - Meta
---

So, while I wait for the hardware I need for my current project to actually arrive, I wanted to tell you a story. It is a story about why you should strictly do your research before you start coding, and not just on the technical stack.

It’s a good one. Let's dive right in.

## What is this all about?
The idea for this project kicked off when my friends and I took an interest in **Genshin Impact** modding. For those out of the loop, Genshin Impact is an Action-RPG gacha game made by Hoyoverse. Now, they are not exactly *kind* regarding people modding their game. It ships with a kernel-level, driver-based anticheat that immediately detects any process trying to touch the game's memory space. It strips the process of the permissions needed to write to those addresses and then terminates it with a kernel function. That last property actually led to a rather [unfortunate incident](https://research.meekolab.com/analyzing-genshin-impacts-anticheat-module)[^1] a while back, but we aren't here to discuss kernel drivers today.

When we say "modding" here, we don't mean cheats or god-mode. We mean client-side asset swapping. Think Counter-Strike: Source weapon skins, but for anime characters.

{{< figure
src="images/mindbog.jpg"
alt="Picture of a Genshin Impact mod, swapping a character with a model of Goku from Dragonball."
link="https://gamebanana.com/mods/471706"
caption="Genshin Impact modding can produce mind-boggling effects. | © cybertron231 CC BY-NC-ND 4.0"
class="article-image"
>}}

Essentially, these mods **swap models and textures** for characters, weapons, and the UI. This sits in a bit of a gray area. The Terms of Service technically forbid it, but Hoyoverse doesn't seem to enforce this rule strictly unless you do something blatant like posting a screenshot with your UID clearly visible. The anticheat generally ignores this layer of modification, though [recent reports suggest this might be changing](https://gamebanana.com/news/21910)[^2].

This entire scene is possible thanks to a fantastic piece of software called **GIMI**.

## How the magic works
[GIMI (Genshin Impact Model Importer)](https://github.com/SilentNightSound/GI-Model-Importer)[^3] is a tool developed by SilentNightSound. It is basically a specialized version of [3DMigoto](https://github.com/bo3b/3Dmigoto)[^4], a DirectX 11 modding suite originally designed for 3D stereoscopic fixes.

3DMigoto works by wrapping the **Direct3D 11** library, acting as a proxy between the game and the GPU. It features a **hash matching function** that can substitute game assets on the fly. It can swap **Vertex Buffers** (geometry), **Index Buffers** (instructions on how to connect vertices), and **Constant Buffers** (variables for special effects). GIMI modifies 3DMigoto to specifically handle Genshin's unique buffer formats.

{{< figure
src="images/broken.jpg"
alt="Picture of a broken mod for Genshin Impact, causing the model to spaghettify."
link="https://github.com/SilentNightSound/GI-Model-Importer/blob/main/Guides/Troubleshooting.md"
caption="The results of applying the wrong model can be... terrifying. | © SilentNightSound GPL-3.0"
class="article-image"
>}}

Using GIMI is not exactly hard, but it adds friction. You need to start the injector *before* running the game. You have to launch the game binary directly to bypass the launcher. If you mess up the order, you have to close everything and restart. It is just enough hassle to be annoying.

So, naturally, being a developer, I started to wonder if there was a way to automate this.

## The quest for an easier life
The objective was clear. I wanted to build a simple application to streamline the launch process. With a single click, it should start GIMI and then launch Genshin in the correct order.

I also wanted to fix a minor annoyance: the Genshin executable is flagged to require Administrator rights. In reality, the game runs perfectly fine without them. That UAC prompt was just another useless click standing between me and the game.

Now, I could have handled all of this with a simple Python script. But I wanted to make something user-friendly. This was going to be the first modding tool I ever released to the public, and I wanted it to be polished.

And that is exactly where the errors began.