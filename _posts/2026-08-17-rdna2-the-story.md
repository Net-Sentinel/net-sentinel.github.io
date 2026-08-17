---
layout: post
title: "We spent months fighting AMD RDNA2 on Windows. Here is what actually works."
date: 2026-08-17
author: NetSentinel
---

The first time it worked, it was not impressive.

A small Qwen model running on an RX 6800. Home Assistant asking it a question. The house answering — slowly, uncertainly, but locally. No cloud. No subscription. No round-trip to someone else's server. We called that a win and decided to push harder.

We should have paused to understand what we had before trying to make it bigger.

---

## Getting ambitious: two cards, a larger model, and a family that noticed

Sixteen gigabytes of VRAM on an RX 6800 is enough for smaller models. It is not enough for the ones that start sounding properly intelligent. The obvious move, when you have a second card sitting in the same machine — an RX 6900 XT, same RDNA2 generation, another sixteen gigabytes — is to split a larger model across both. Ollama will do it. You point both cards at the runner and a 27 billion parameter Qwen model loads.

It ran. For about three minutes.

The problem with voice in a house full of people is that "it runs" is not the same as "it works." Conversations are not continuous. Someone asks a question, the assistant answers, fifteen minutes pass, someone else has a question. In those fifteen minutes, the model was gone. Not gracefully suspended. Evicted. Ollama would reload it when the next question arrived. That reload is not fast.

When your family member is standing in the kitchen asking something and the house replies with forty-five seconds of silence followed by an answer, the family member stops talking to the house. They use their phone. They are polite about it. You feel it regardless.

We had built something impressive on paper and unusable in practice.

---

## The keep-alive lie

`OLLAMA_KEEP_ALIVE=-1`. Tell Ollama to never unload the model. Problem solved.

It was not solved.

Ollama and Windows are not the only parties in this conversation. Windows WDDM — the graphics driver memory manager — has its own view on which GPU memory pages deserve to stay in the fast lane. Pages that have not been touched recently get quietly moved out of dedicated VRAM into system memory. No warning. No API notification. The Ollama health endpoint still says the model is loaded. The OS-level dedicated memory counter on the GPU tells a very different story.

We spent longer than we would like to admit diagnosing this. The API was lying by omission. The GPU dashboard was lying by aggregation — it shows committed memory, not dedicated. The only truth was the OS dedicated memory counter, and nobody thinks to check that until they have already wasted a week on forum posts.

The two-card split made this worse. Sixteen gigabytes on each card, a 27B model split across both via PCIe, WDDM doing its memory accounting on two devices simultaneously. Every question involved negotiation between cards. The latency was down from "obviously broken" to "noticeable." For a family voice path, noticeable is still broken.

---

## Murray's first batch file

The real fix for WDDM eviction is not a flag. It is a heartbeat. A small script that touches the model regularly enough to keep the pages warm in dedicated memory. If the model has been evicted, reload it. If it is present, ping it with a tiny request. Wait. Check again.

Murray wrote the first version of this. Here it is, bugs and all:

```batch
@echo off
:loop
ollama ps | findstr "gemma4" >nul
if errorlevel 1 (
    echo Reloading...
    start "" ollama run gemma4:26b
    timeout /t 30 /nobreak
)
curl -s -X POST http://localhost:11434/api/generate ^
  -d "{\"model\":\"gemma4:26b\",\"prompt\":\"hi\"}" >nul
timeout /t 300 /nobreak
goto loop
```

It has two bugs that cost real time to find.

The first is `timeout /t`. In a Windows Scheduled Task running in Session 0 — which is the context your task runs in when no user is logged into the console — `timeout /t` returns immediately instead of waiting. The loop becomes a tight spin. Every few hundred milliseconds, a generate request fires. The log grows to tens of megabytes overnight. The GPU sits at constant low compute load. Idle wattage climbs. Nothing looks obviously wrong because nothing reports an error. The fix is `powershell -NoProfile -Command "Start-Sleep -Seconds 300"`. Never `timeout /t` in a scheduled task that needs to sleep.

The second bug is the generate request itself. `"prompt":"hi"` with no token limit produces an unbounded response. Ollama will generate until it decides it is done — which, for a home assistant model asked "hi", could be a paragraph. The CPU runs at twenty-five percent during this time. The fix is `"options":{"num_predict":1}` in the request body. Note: top-level `num_predict` is ignored by Ollama. It must be under `options`.

There is a third problem that is not in the script itself. Windows Scheduled Tasks have an `ExecutionTimeLimit`. The default is 72 hours. At exactly 72 hours, the task will be killed silently — healthy log right up to the moment, then nothing. The result code is `267014` (0x41306) if you check Task Scheduler afterward. The fix is to set Stop Task If Runs to Disabled. Verify with `schtasks /Query /TN "Ollama Server" /V` and look for that line. Do not trust that it is set correctly. Check.

Also: never start this bat from SSH or from WSL. Let an interactive Scheduled Task own the process tree at logon. SSH and WSL-spawned processes inherit Session 0, which is where `timeout` goes wrong and where "where did my window go" becomes a recurring mystery.

The crude batch file is shown here because it represents something real. This is what the beginning of a working system looks like. Imperfect. Fixable. Worth showing.

---

## The two-card setup, honestly assessed

With the keeper script running correctly and the bugs fixed, the two-card split was usable. The model stayed loaded. Requests got answered.

It was still not great for voice.

Splitting a 27B model across two consumer cards over PCIe has a latency cost that no script can remove. Every inference round-trip involves data moving between cards. Every response is the result of two GPUs coordinating their share of the layers. For a home voice path where the expectation is natural back-and-forth, the hesitation was still there. Smaller. Still present. Still the difference between "this is convenient" and "I'll just use my phone."

And then there were the ordinal flips. After a driver update, after a reboot, the device Ollama called device 0 might be the display card and not the compute card. The model would load with most layers on the wrong GPU and a handful in system RAM, and the API would report everything as healthy. We set environment variables. We wrote detection scripts. We wrote `ROCR_VISIBLE_DEVICES=0` into every start script. Some of it worked some of the time.

None of it was durable. Because `ROCR_VISIBLE_DEVICES` ordinals can flip. The variable is correct; the mapping it refers to has changed.

We had a workaround. A workaround is not a product.

---

## £500 and thirty-two gigabytes

Second-hand workstation cards do not get the same press as consumer GPUs. They sit quietly on eBay while everyone argues about RTX generations. The AMD Radeon Pro W6800 is an RDNA2 workstation board. It has thirty-two gigabytes of VRAM. It runs cooler and draws less at load than a gaming card in the same generation. It is not beautiful. It was around £500 used.

We bought one.

The change was not subtle. Thirty-two gigabytes is enough to hold a 26 billion parameter model resident with a 131,000-token context window. Not "loaded until Windows gets bored." Resident. The keeper script now has a real job: maintaining warm dedicated pages rather than fighting a mismatch between model size and card capacity.

One card. One job. The RX 6800 got its own role: display, Plex, speech-to-text via faster-whisper. Clean separation. No competition for VRAM. No PCIe negotiation between two devices. No ordinal flip to worry about when there is only one card in the Ollama job.

The question of whether the model is on the right GPU becomes much simpler when there is only one GPU that could be running it.

---

## The golden corner: Ollama 0.24.0, ROCm, gemma4:26b

"It works on AMD on Windows" depends entirely on which Ollama version and which AMD driver stack are on the box together. This is the part that the forums get wrong consistently, because people report results from one pairing and readers assume it generalises.

Our corner, found after a lot of trial and documentation: **Ollama 0.24.0**, unified AMD drivers from May 2026, the **ROCm backend**, and the HIP 6 runtime that version of Ollama bundles. Not HIP 7. Not the newest AMD installer's assumption. HIP 6, because that is what gfx1030 — the RDNA2 compute identity — needs on this build.

`HSA_OVERRIDE_GFX_VERSION=10.3.0` is required. Without it, the ROCm stack does not correctly identify the GPU generation. With it, the serve log shows `library=ROCm compute=gfx1030`, the card name, and all thirty-one layers offloaded to GPU. All thirty-one. Not twenty-eight. Not "most of them." All of them.

One naming mistake worth spelling out: on Windows, device selection is controlled by `ROCR_VISIBLE_DEVICES` and `HIP_VISIBLE_DEVICES`. Not `ROCM_VISIBLE_DEVICES`. Many Linux guides use `ROCM_` and it silently does nothing on Windows. The serve log is the truth. Not the environment variable you set. Not the API. The log.

The model we settled on was **gemma4:26b**. It fits in the W6800's dedicated VRAM with room for the context window. It answers questions properly. It understands what the house needs. It runs Home Assistant voice correctly — not three-second latency, not "wait while I reload," but responsive.

---

## When it started sounding like something you would pay for

This is the part that was genuinely surprising.

We had been using cloud AI models throughout this process — for comparison, for the more complex work. We knew what good answers looked like. When gemma4 started matching that quality on the local model, on our own hardware, that was a different kind of result.

A 131,000-token context window means the assistant holds a real conversation. Not five exchanges. Not "what were we talking about?" two minutes in. A real context. Ask it to plan something, come back to it, add a condition — it tracks.

The house does not have a search engine in the corner any more. It has something that reasons.

---

## It became part of something bigger

When a local model runs reliably enough — stays resident, responds quickly, handles real context — it stops being just a home assistant and starts looking like infrastructure.

We run an AI assistant platform alongside all of this. The W6800 box, with gemma4 at ROCm speed and the keeper script maintaining residency, became a worker node in that system. Bounded, specific tasks. Not the model you send your most complex reasoning chains to unsupervised. But for processing, classification, local inference where latency matters — it earns its keep.

With caveats, and those caveats matter. But the same card that answers "set a timer for twenty minutes" in the kitchen is doing real work in a real AI system. We did not plan for that when we bought a second-hand workstation GPU to run Home Assistant voice. We find the outcome pleasing.

---

## Code 43 and the thing that actually fixed it

No honest AMD-on-Windows account omits Code 43.

We saw it. Multiple times. Hard crash during overnight inference, TDR event, hard reset, restart, Device Manager showing Code 43 on the compute card. PnP disable/enable did not reliably fix it. DDU in safe mode usually fixed it. We say usually because there was a period where nothing fixed it — and the reason, when we finally stopped adding variables and thought about it, was that the Windows install itself was corrupted.

Weeks of failed repairs: aggressive PnP cycling, driver reinstalls on top of driver reinstalls, system restore points, crash reboots under GPU load, bat surgery while the model was running. What we had accumulated was a poisoned C:\ drive. DDU removes driver files. It does not clean a Windows install that has had months of LLM-driven trauma layered into it.

The actual fix was a clean Windows reinstall. Takes an afternoon. Works.

If DDU has not resolved Code 43 after two clean attempts on a system with a long stress history, stop adding variables. The problem may not be the driver. The problem may be the operating system state underneath the driver. That diagnosis cost us more time than we would like to admit.

---

## Where we are now: Vulkan and what comes next

This story does not have a tidy ending, because it is not finished.

Ollama 0.24.0 on ROCm ran the house for months. It is still a valid corner for anyone on RDNA2 who wants a proven, working configuration. But newer models require newer Ollama builds, and newer builds return HTTP 412 against 0.24.0: "the model you are attempting to pull requires a newer version of Ollama." That is not a VRAM problem. The weights never reach disk. The gate is the server version.

`qwen3.8:27b` was waiting on the other side of that gate.

So this week, with a proper rollback kit in place, we moved to the latest Ollama build on the **Vulkan backend**. RDNA2 is no longer on the Windows ROCm support matrix for the newer stack — AMD's current Windows ROCm documentation covers RDNA3 — but Vulkan fills the gap. Different backend, same GPU, different rules.

The Vulkan rules are specific. Device enumeration is not the same as ROCm enumeration. On our box, Vulkan device 0 is the display card and device 1 is the compute card. `GGML_VK_VISIBLE_DEVICES=1`, not `0`. Print the device map after every driver bump or Ollama update. Assume nothing.

As we write this, qwen3.8:27b is loaded, resident in dedicated VRAM on the W6800 Pro, Vulkan backend, latest Ollama. The family do not know it changed. That is the test.

Results pending. We will update this post when the verdict is in.

---

## What you actually need

This is the part that matters for anyone reading this with a gaming card and a sceptical forum thread open in another tab.

A used **RX 6800 or RX 6900 XT** costs what it costs in your market. Sixteen gigabytes. For models in the 8–14 billion parameter range, that is enough — not a compromise, actually enough. Those models run quickly, stay hot with a keeper script, and handle everything a home assistant needs to do. Start here. Prove the basics before spending more money.

If you want a 26 billion parameter model resident for a family voice path, sixteen gigabytes will not do it cleanly on a single card. A used **W6800 Pro** at 32 GB is the move if you can find one. The used workstation market is quieter than the gaming GPU market. Worth looking.

You do not need an AI fleet. You do not need a cloud subscription. You do not need Nvidia. You need:

- A card with enough VRAM for the model you actually want to run — not the biggest model you can find, the right one for your card
- Ollama pinned to a known-good version for your hardware
- A keeper script that sleeps correctly, bounds its generate requests, and runs from the right session
- The patience to read the serve log, not the API, not the dashboard, not the environment variable you set

Murray built the foundation of this himself. A batch file, a second-hand GPU, months of stubborn iteration. The AI tooling that lives alongside it now makes it more capable — but the working system was built by one person on hardware that the forums said would not work.

It works.

---

## For anyone with an RDNA2 card in a drawer

The default narrative is that AMD on Windows is hopeless for local AI. That narrative is wrong — and it is also convenient for people selling new hardware.

RDNA2 is more work than it should be. The documentation trails the hardware. The community advice is inconsistent. WDDM behaviour is genuinely frustrating. None of that means it does not work.

The cards in this story are not exotic. RX 6800, RX 6900 XT, and used W6800-class boards are hardware people already own or can still buy without a startup budget. Sixteen gigabytes runs smaller models properly. Thirty-two gigabytes runs a mid-20B model resident if you do the software work.

Match the model size to the VRAM you have. Pin a known-good Ollama version and driver pair. Build a keeper script that actually sleeps. Read the serve log.

A family that talks to the house without a cloud round-trip for the everyday questions. A model doing real work in a real system. Months of evidence.

The card was not the villain. Wishful process monitoring was the villain — and buying too much model for the VRAM available was the other villain, for a while.

Fix those two things and RDNA2 on Windows is a rational local AI platform. We have the receipts.
