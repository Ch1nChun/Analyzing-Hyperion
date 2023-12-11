# Analyzing Hyperion - Part 1: Starting Out

## Introduction

> [!IMPORTANT]
> Attention: Part 2 is available here [Analyzing Hyperion - Part 1: Starting Out](Part%202.md)

Hello everyone. this is gocart98. I apologize for my recent inactivity, I've been quite busy preparing for this series.

I'm excited to announce that I'm launching a new multi-part series titled "*Analyzing Hyperion,*" where I'll walk you through the step-by-step process of reverse engineering Hyperion.

A link to the next part will be posted under this thread once it's available, allowing you to easily pick up where you left off. Stay tuned for more in-depth explorations of Hyperion's inner workings.

## Credits

Before we start, here is a list of this thread's authors.

- [Dr McDingle](https://v3rmillion.net/member.php?action=profile&uid=618689)
- [stretchnuts](https://v3rmillion.net/member.php?action=profile&uid=350742)

## Table of Contents

- [Analyzing Hyperion - Part 1: Starting Out](#analyzing-hyperion---part-1-starting-out)
  - [Introduction](#introduction)
  - [Credits](#credits)
  - [Table of Contents](#table-of-contents)
  - [Overview](#overview)
  - [High-level View of Hyperion](#high-level-view-of-hyperion)
    - [The basics](#the-basics)
    - [Outline](#outline)
    - [Final Thoughts](#final-thoughts)

## Overview

Hyperion is typically referred to as an "anti-cheat," while this terminology is correct, you must understand the difference between Hyperion's "AC" and traditional anti-cheats.

Hyperion is more accurately described as an "anti-tamper;" it differs from traditional anti-cheats.

Traditional anti-cheats pivot towards cheat detection. To protect their detection vectors and halt reverse engineers, typically, they obfuscate the code belonging to their anti-cheat.

On the other hand, Hyperion simply attempts to make reverse engineering of the game itself harder.
It encrypts the game code and decrypts it only when needed (when execution occurs), and also protects the imports. It aims to make reverse engineering harder in general.

When Hyperion was used in other games, it was accompanied by another anti-cheat. The goal of Hyperion was to make reverse engineering of the game harder, as well as development on cheats. It isn't meant to work as a full anti-cheat by itself, but rather an anti-tamper to be paired with anti-cheats.

## High-level View of Hyperion

In this section, I will describe the main features of Hyperion. I will go more in depth later in this series. The final goal of this series is to know enough to be able to decrypt Roblox and achieve injection.

A key thing I want to mention here: Many developers in this community think that the current Hyperion version is somewhat final and weak; really the current version of Hyperion isn't anything special.

What you must remember is that in it's current state, Roblox's main goal is to fix core compatibility issues and ensure it works across different computers. Only after that, can they begin to expand to their full potential. So if you are one of them, prepare for huge surprises in the future.

### The basics

The first thing most reverse engineers would do is to simply try to disassemble Roblox. Well when you do that, the first thing you will notice is the entry point of Roblox is just an int 3; a debug trap. If you look at the imports, you will notice that there is only one import and its the 'run' export of RobloxPlayerBeta.dll, this is where Hyperion resides. You will also quickly notice that the entire .text section of Roblox (where the executable code resides) is just junk. So what's happening here?

Well, let's first start with the basics.
As you may have assumed, Hyperion runs before Roblox's entry point. How is this possible?
You need to think about how the Windows loader works. Before attempting to run any code in the executable, it must first resolve all imported DLLs, load them into the process, and call their entry point (if they have one).
When an imported DLL has an entry point and is called by the Windows loader, it's done in the main thread, meaning it runs before the entry point of the process itself, and the process entry point cannot run until all DLL entry points are finished.
Hyperion abuses this Windows feature to run code before Roblox's entry point, making the Roblox executable useless if there isn't something to handle the exception.

This is done so Hyperion can do critical initialization before anything else, for example they have an internal table storing the original imports of Roblox; before they give control to the original entry point, they will manually load all DLLs that Roblox is using imports from. They also protect some of these imports, but that's a topic for a future part.

Now that we've covered the essentials, let's look at some of their "dynamic" features.
As I stated earlier, our main goal is to achieve injection. To do that, you need to allocate executable memory for your module and also create a thread to begin execution.

To do this, you will need a full-access process handle open to Roblox, however, Roblox is actually checking for open handles to its process, especially from unsigned programs.
If we naively try to open a handle, it will not stay open for long, this means that external cheats will have to continuously re-open new handles, or use an existing handle opened by something else.

If you try to remotely create a thread in Roblox, you'll quickly notice that the thread won't start at all.
As I said, detailed reversals about those features will come in future parts.
Finally, if you try to remotely allocate executable memory, you may notice that after about 5 seconds, the protection will change.

You should also know that Hyperion is constantly scanning all running processes on your computer, if they detect public reverse engineering tools like Cheat Engine, ReClass, IDA, etc, they will make Roblox crash.
In a future part, we will write a tool to hide them, so we can analyze Hyperion easier. They also have various detections for hypervisors; that will be covered too.

### Outline

Let's reiterate what we now know. Hyperion encrypts the game's code, hides the imports, redirects the entry point, monitors running processes in real-time for known reverse engineering tools, tracks the working set of the process, monitors the threads, checks for the presence of a "malicious" hypervisor, and more, which is beyond the scope of this part.

### Final Thoughts

This should give you a basic idea of how Hyperion operates at high level.

I didn't cover absolutely everything here, only the key features.

In the next part, I will go into detail on analyzing Hyperion, the obfuscations they use in their module, the code encryption over Roblox, how to decrypt it, and finally, how to write a dumper.
