---
title: "NeoHanega Post-Mortem: The Cost of Reinventing the Wheel"
description: Why ignoring file permissions and skipping reconnaissance turned a simple launcher into a lesson in humility.
date: 2026-02-03
image: cover.jpg
categories:
    - Reverse Engineering
    - Post-Mortem
---

So, while I wait for the hardware I need for my current projects to actually arrive, I wanted to tell you a story. It is a story about why you should strictly do your research before you start coding, and not just on the technical stack.

It’s a good one. Let's dive right in.

## What is this all about?
The idea for this project kicked off when my friends and I took an interest in **Genshin Impact** modding. For those out of the loop, Genshin Impact is an Action-RPG gacha game made by Hoyoverse. Now, they are not exactly *kind* regarding people modding their game. It ships with a kernel-level, driver-based anticheat that immediately detects any process trying to touch the game's memory space. It strips the process of the permissions needed to write to those addresses and then terminates it with a kernel function. That last property actually led to a rather unfortunate incident[^1] a while back, but we aren't here to discuss kernel drivers today.

When we say "modding" here, we don't mean cheats or god-mode. We mean client-side asset swapping. Think Counter-Strike: Source weapon skins, but for anime characters.

{{< figure
src="images/mindbog.jpg"
alt="Picture of a Genshin Impact mod, swapping a character with a model of Goku from Dragonball."
link="https://gamebanana.com/mods/471706"
caption="Genshin Impact modding can produce mind-boggling effects. | © cybertron231 CC BY-NC-ND 4.0"
class="article-image"
>}}

Essentially, these mods **swap models and textures** for characters, weapons, and the UI. This sits in a bit of a gray area. The Terms of Service technically forbid it, but Hoyoverse doesn't seem to enforce this rule strictly unless you do something blatant like posting a screenshot with your UID clearly visible. The anticheat generally ignores this layer of modification, though recent reports suggest this might be changing[^2].

This entire scene is possible thanks to a fantastic piece of software called **GIMI**.

## How the magic works
GIMI (Genshin Impact Model Importer)[^3] is a tool developed by SilentNightSound. It is basically a specialized version of 3DMigoto[^4], a DirectX 11 modding suite originally designed for 3D stereoscopic fixes.

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

### RTFM, but actually
I started by building a proof of concept: a simple CLI-based Python script to test the waters and figure out exactly which libraries I was going to need.
Spinning up GIMI and Genshin in the correct order was easy enough. The only real "gotcha" to keep in mind is that the programs must be spawned as **independent processes**, not child processes that die when your script closes. Python's `psutil` module (specifically `psutil.Popen`) made this trivial, giving me easy handles to manage the processes without attaching their lifecycles to my script.

Then came the actually difficult part: **The UAC Prompt.**

{{< figure
src="images/code1.jpg"
alt="Picture of the code that handles the launching of Genshin and GIMI."
link="https://github.com/ProsD03/Hanega/blob/master/hanega.pyw"
caption="Snippet of code from the lastest version of Hanega that handles launching GIMI and Genshin."
class="article-image"
>}}

Windows handles Administrator privilege requirements in a few ways, but the two big ones are the **Application Manifest**[^5] and the **Registry**.
The manifest is a file embedded inside the `.exe` itself that basically shouts, *"I need Admin rights!"*.
The Registry, however, can override these shouts. Specifically, the key `HKEY_CURRENT_USER\Software\Microsoft\Windows NT\CurrentVersion\AppCompatFlags\Layers` acts as a compatibility list. You can map an executable path to a set of flags, and if you map the Genshin executable to `~ RUNASINVOKER`, you are essentially telling Windows: *"Ignore the manifest. Run this with the privileges of the user who clicked it (me)."*

So, I devised a "clever" plan. I wrote a helper script called **Musou**. It would run with Admin rights *once* during setup, inject this registry key, and solve the UAC problem forever. I patted myself on the back and considered it solved.

*It was not solved.*

{{< figure
src="images/code2.jpg"
alt="Picture of the code that changes the registry key."
link="https://github.com/ProsD03/Hanega/blob/master/hanega.pyw"
caption="Snippet of code from the lastest version of Musou that overwrites the Registry subkey."
class="article-image"
>}}

It turns out this approach had three massive holes in it, one of which was fatal:

1.  **The "Admin" Fallacy:** I was writing to `HKEY_CURRENT_USER`. As the name implies, this hive belongs to the *current user*. You **do not need Admin rights**[^6] to edit it. My entire "Musou" helper script, and the UAC prompt I forced the user to accept to run it, was completely theatrical and unnecessary.
2.  **The Clobbering:** My script blindly set the value of the registry key. If a user had already set other compatibility flags (like "Disable Fullscreen Optimizations" or Windows 8 compatibility), my script would have ruthlessly overwritten them, potentially breaking their game configuration.
3.  **The Fatal Flaw:** This is the big one. Genshin Impact is usually installed in `C:\Program Files\Genshin Impact\Genshin Impact game`. `Program Files` is a protected system directory. Standard users have **Read** access, but not **Write** access[^7]. When the game tries to update itself, it needs to write new files to this folder. By forcing it to run as a standard user with `RUNASINVOKER`, I successfully stripped it of the ability to update. I had created a launcher that would slowly break the game it was meant to launch. Oops.

In short, it only worked because I committed the cardinal sin: *"It works on my machine."* My game was on a secondary drive, sidestepping the strict permissions of `C:\Program Files`. For the vast majority of users, my script would have stripped the game of the write access it needs to patch itself, silently breaking updates. And the cherry on top? The `Musou` helper was completely useless, as `HKEY_CURRENT_USER` never required Admin rights to begin with.

**Lesson Learned:** Before you rush to implement a "clever" hack, make sure you understand the permissions model underneath it. Read the documentation, test your edge cases, and never assume you need Administrator Rights for something that belongs to User.

### VirusTotal is (not) your friend
Moving on, once I had the basic PoC working, I started building a proper GUI using **PyQt**. PyQt is a wrapper of the famous Qt framework for Python. It is easy to use and has a massive amount of documentation, so it seemed like a good choice for me—a poor backend developer with zero knowledge or taste in UI design. It didn't take me long to have a working prototype.

{{< figure
src="images/hanegaui.jpg"
alt="Picture of the Hanega UI made with PyQt."
link="https://github.com/ProsD03/Hanega/blob/master/hanega.pyw"
caption="PyQt makes it quite easy to build a cute-y (lol) UI."
class="article-image"
>}}

With a working UI, proofread text, and a "functional" backend, I was ready to release my project to the world. Being a Python application, I needed a way to package it so users wouldn't have to install Python and all the dependencies manually. After some (way too little!) research, I settled on **PyInstaller**. It is a popular tool that bundles Python applications into standalone executables or folders by extracting a Python Embeddable Package[^8] and all necessary libraries into a single `.exe` file. It sounded perfect, so I whipped up a build script, generated the executable, and published it on GitHub, GameBanana, and Reddit without a second thought.

**Big Mistake.**

It's been over a year, and I've since deleted the original submissions, so I can't show you the carnage. But trust me when I say this: **VirusTotal absolutely crucified my project.**

It turns out that **PyInstaller** is notoriously hated by antivirus engines. The reason is simple: malware authors love Python too, and they frequently use PyInstaller to package their payloads[^9]. Because every PyInstaller executable shares the same pre-compiled "bootloader" stub (the code that unpacks and runs your Python script), antivirus heuristics often flag *anything* using that bootloader as suspicious or malicious by association.

The result? My launcher was flagged by dozens of engines. GameBanana automatically withheld the submission and soft-banned my account. Reddit mods nuked my post and warned me about distributing unchecked software.

{{< figure
src="images/withhold.jpg"
alt="Picture of the Withhold advisory on GameBanana."
link="https://github.com/ProsD03/Hanega/blob/master/hanega.pyw"
caption="Hanega was withheld immediately after launch, and has been like this ever since."
class="article-image"
>}}

I tried to salvage it. I attempted to use **InstallForge** to create a standard installer wrapper, hoping to achieve the same result without using PyInstaller, but VirusTotal remained unimpressed, as the behavior it saw was identical to PyInstaller. The project was toast. I had two options: abandon the packer and force users to install Python manually (a UX nightmare), or abandon Python entirely and rewrite the project in a compiled language. Since code signing certificates cost money I didn't have[^10], I chose the latter. I simply didn't want to deal with the headache of packaging interpreted languages for desktop use anymore.

**Lesson Learned:** Always check the reputation of your distribution tools, and choose the right language for the job. All of this pain could have been avoided if I had just written the software in C++ or C# from the get-go, instead of trying to shoehorn a Python script into a binary executable.

### The Nail in the Coffin
Morale was at rock bottom, but I was determined to finish the project. I renamed it **NeoHanega** (since it was the "new" Hanega) and rewrote the entire stack in C#. I used **WinForms** for the UI, compiling it directly to a native executable that only needed the .NET 8 runtime. I tested it thoroughly, ran it through VirusTotal to ensure it was clean, and finally pushed the release. Weeks of coding, research, and debugging had finally paid off. I had a working, user-friendly launcher that didn't trigger a nuclear response from antivirus engines.

But as you can guess, this chapter is not called "Happy Ending." The project still ended up being a complete failure. Why?

Well, because of this:

{{< figure
src="images/fiasco.jpg"
alt="Picture of a comment saying another tool already exists."
link="https://gamebanana.com/tools/18519"
caption="Just get XXMI Launcher."
class="article-image"
>}}

**XXMI Launcher.**[^11] A tool that did exactly what NeoHanega did, but better. It supported newer versions of GIMI, had auto-updates, and featured a far more polished UI. Its first release dropped exactly two months before NeoHanega, and it had already reached version 1.1.4 by the time I showed up. It is actively maintained, has a thriving user base, and is generally considered the standard Genshin mod launcher in the community. As the name implies, it actually supports any game moddable with 3DMigoto—Honkai Star Rail, Zenless Zone Zero, and Wuthering Waves.

In short, it does more than NeoHanega ever did, or ever would.

{{< figure
src="images/XXMI Launcher.jpg"
alt="Picture of XXMI Launcher."
link="https://github.com/SpectrumQT/XXMI-Launcher"
caption="UIs like this are beyond my small backend brain. | © SpectrumQT GPL-3.0"
class="article-image"
>}}

**Lesson Learned:** Before you embark on a project, check, double-check, and triple-check that nobody has already done it—or at least ensure that what you are building is significantly different. This is especially true for "obvious" ideas like a game launcher. The last thing you want is to spend weeks building a solution, only for the community to point you to the superior, established alternative five minutes after you release. It is a massive blow to morale, and it forces you to ask the painful question: *was this worth the effort at all?*

## Conclusion
In the end, I can't really call NeoHanega a total failure. Sure, it was a bit of a dumpster fire—riddled with bugs, bad assumptions, and architectural mistakes (some of which I only realized while writing this very post). But it was a necessary fire.

I learned that **reconnaissance is non-negotiable**. Every hour you spend researching the ecosystem is an hour you don't waste building a tool that already exists or implementing a hack that breaks the user's system. I also learned the hard way that **you have to pick the right tool for the job**. Python is my comfort zone, but shoehorning it into a desktop application with PyInstaller was a mistake. C# and .NET were the correct tools all along, and I should have started there.

This experience fundamentally changed how I approach engineering. If sharing this post-mortem helps just one other dev avoid my rabbit holes, then the project wasn't useless after all.

---

Thank you for reading. This is the first entry in my digital garden, and I hope you enjoyed the ride. If you are curious (or morbidly fascinated), you can find the archived source code for the C# version on my GitHub[^12]. It is unmaintained, but it stands there as a monument to the lessons I learned.

I'll catch you soon with more projects—hopefully, ones that actually work.

See you, space cowboys. 🚀

[^1]: https://research.meekolab.com/analyzing-genshin-impacts-anticheat-module
[^2]: https://gamebanana.com/news/21910
[^3]: https://github.com/SilentNightSound/GI-Model-Importer
[^4]: https://github.com/bo3b/3Dmigoto  
[^5]: https://learn.microsoft.com/en-us/windows/win32/sbscs/application-manifests
[^6]: https://learn.microsoft.com/en-us/troubleshoot/windows-server/performance/windows-registry-advanced-users
[^7]: https://en.wikipedia.org/wiki/User_Account_Control#Tasks_that_trigger_a_UAC_prompt
[^8]: https://docs.python.org/3/using/windows.html#the-embeddable-package
[^9]: https://github.com/pyinstaller/pyinstaller/issues/6754
[^10]: https://www.reddit.com/r/electronjs/comments/17sizjf/a_guide_to_code_signing_certificates_for_the/
[^11]: https://github.com/SpectrumQT/XXMI-Launcher
[^12]: https://github.com/ProsD03/NeoHanega
