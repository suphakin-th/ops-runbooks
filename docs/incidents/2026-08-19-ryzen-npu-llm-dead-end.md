---
date: 2026-08-19
tags:
  - windows
  - ai
  - ollama
  - amd
  - hardware
---

# AMD NPU Won't Run LLMs — A Full Day Chasing a Hardware-Generation Wall

**Date:** August 19, 2026  
**Device:** ASUS Vivobook S14 (AMD Ryzen 7 260, Hawk Point / XDNA1 NPU)  
**Severity:** 🟡 Low (no data/security impact — just a lot of wasted setup time)

---

## What happened?

Started the day setting up a local LLM pipeline (`news_prefilter.py`) to summarize
AI-bubble/market-crash headlines on-device before feeding them into the JARVIS
morning brief — using Ollama on the laptop's integrated GPU (Radeon 780M). That
part worked cleanly: `OLLAMA_IGPU_ENABLE=1` was needed to stop Ollama from
silently dropping the iGPU and falling back to CPU (it detects Vulkan support but
disables it by default for integrated GPUs), and once set, `ollama ps` confirmed
`100% GPU`.

Then came the follow-up question: *this laptop also has a dedicated NPU sitting
right next to the CPU — can that run the LLM instead, to free up the GPU or save
power?*

What followed was a full afternoon of chasing that NPU down three different
toolchains, each one dying for a different reason, ending in a PATH-propagation
nightmare that needed two full system restarts — only to discover at the very end
that the answer was "no" from the start, baked into AMD's own software stack, not
fixable by more config.

---

## Root Cause — three dead ends, then a hardware ceiling

### Dead end 1 — Lemonade SDK requires XDNA2, this chip has XDNA1

Lemonade (AMD's Ollama-style NPU server) and FastFlowLM both hard-require an
XDNA2 NPU — that's the Ryzen AI 300/400-series and Strix Halo. The Ryzen 7 260 is
**Hawk Point**, a Zen 4 refresh chip with the older **XDNA1** NPU (16 TOPS).
AMD's own docs list Ryzen AI 7000/8000/**200-series** as explicitly unsupported
for this toolchain. Confirmed via Lemonade's own compatibility page — no
ambiguity, no workaround.

### Dead end 2 — Not a Copilot+ PC either

Windows Copilot+ PC features (on-device Phi Silica, etc.) require a **40+ TOPS**
NPU. This chip has 16 TOPS. Not close enough to qualify for any of the
NPU-accelerated Windows system APIs either.

### Dead end 3 — AMD's own Ryzen AI Software installs, then hits a missing firmware file

Ryzen AI Software 1.8.0 (the official ONNX Runtime VitisAI EP toolchain) does
list Hawk Point as a supported chip, so this looked like the real path. Full
install:

- Visual Studio 2022 (C++ desktop workload), CMake, Miniforge — all via `winget`
- NPU driver — already had `32.0.203.314`, above the `32.0.203.280` minimum
- `ryzen-ai-1.8.0.exe` (3.5GB) — required a manual login at `account.amd.com`
  to download; account/login actions are something I don't do on the user's
  behalf, so this step waited for them

Installer kept failing at the conda check:

```
Conda not found. Please install conda...
```

...even after Miniforge was installed and `conda --version` worked fine in a
terminal. This turned into its own multi-round debugging chase (see below).
Once past that, the installer actually finished cleanly: conda env created,
VitisAI EP wheel installed, ONNX Runtime GenAI DirectML wheel installed, all
exit code 0.

Ran the official `quicktest.py` to verify NPU inference. It printed
`Test Finished` — but the log was full of:

```
E: xclbin_fingerprint.cpp:71 Failed to open xclbin
```

`.xclbin` is the NPU firmware/bitstream file the runtime needs to actually talk
to the hardware. Checked the install directory:
`voe-4.0-win_amd64\xclbins\phoenix\` — the folder existed, but was **empty**.
The `4x4.xclbin` file for Phoenix/Hawk Point was simply missing from the
installer package. Web search turned up other users hitting the same empty
`phoenix` xclbin folder — a known AMD packaging gap, not something wrong with
this machine.

### The real answer, found in AMD's own release notes

Even setting the missing xclbin aside, AMD's Ryzen AI Software 1.8.0 release
notes state plainly: **Phoenix/Hawk Point (XDNA1) only supports CNN INT8
models.** LLM and BF16 workloads are restricted to the newer Strix/Krackan
Point (XDNA2) chips. This NPU generation was built for lightweight always-on
CNN inference — background webcam effects, not language models — and that's
enforced in AMD's software stack itself, not something a driver update or
config flag changes.

**Bottom line: this NPU cannot run LLM inference right now, full stop.** Not a
setup mistake, not a missing package — a hardware-generation limit.

---

### Side quest — the "Conda not found" PATH nightmare

Worth its own writeup because it burned the most wall-clock time and had a
genuinely non-obvious cause.

`conda --version` worked fine in every terminal tested. The installer still
couldn't find it. Root cause, found step by step:

1. `winget install CondaForge.Miniforge3` had installed to
   `C:\Users\jadek\miniforge3` — but an earlier, older Miniforge install already
   existed at a *different* path, and the first PATH edit pointed at the wrong
   (non-existent) `AppData\Local\miniforge3` location. Fixed by pointing PATH
   at the real install dir.
2. Even after fixing the User PATH, the installer (run **as administrator**)
   still failed. Elevated processes on Windows read from **System (Machine)
   PATH**, not User PATH — and `[Environment]::SetEnvironmentVariable(...,
   "Machine")` requires admin rights the automation shell didn't have. Had to
   walk through `sysdm.cpl` → Advanced → Environment Variables → System
   variables → add the three conda paths manually.
3. Windows does not propagate PATH changes to already-running processes
   automatically — not `explorer.exe`, not `cmd.exe` subprocesses spawned from
   an existing session, nothing already alive at the time of the registry
   write. Restarting `explorer.exe` alone wasn't enough, since the shell that
   launched the installer was itself a descendant of a long-running parent
   process. Only a **full system restart** — twice, since the first didn't
   fully take for every process chain — got every process to read the updated
   System PATH.

Lesson: when an installer says "X not found" right after X was just installed,
check *which* PATH scope (user vs. system) and *which process generation* is
actually being consulted before assuming the install itself is broken.

### Fix

```powershell
# The actual System PATH fix (via sysdm.cpl GUI, not scriptable without admin):
#   Advanced System Settings > Environment Variables > System variables > Path
#   Add: C:\Users\jadek\miniforge3\condabin
#        C:\Users\jadek\miniforge3\Scripts
#        C:\Users\jadek\miniforge3
# Then: full restart (not just explorer.exe restart) for it to fully propagate.
```

---

## Risk Assessment

### 🟡 Low — No security or data exposure, just time cost

Nothing here touched sensitive data or opened any attack surface — this was a
pure feasibility investigation. The "risk" was entirely about time: roughly a
full afternoon spent on prerequisite installs, two PATH-driven system restarts,
and a 3.5GB+6GB install cycle, all for a conclusion that was knowable from
AMD's public spec sheet before any of it started (16 TOPS NPU, no LLM support
in the software stack for this chip generation).

**Fix:** for future hardware-capability questions, check the vendor's official
supported-workload matrix for the exact chip *before* installing anything —
Hawk Point's "CNN INT8 only" restriction was documented in AMD's own release
notes the whole time.

---

## Summary

| Issue | Severity | Fixed |
|---|---|---|
| Lemonade SDK requires XDNA2, chip has XDNA1 | 🟡 Low | ❌ Hardware limit, not fixable |
| Not Copilot+ PC eligible (16 vs 40+ TOPS) | 🟡 Low | ❌ Hardware limit, not fixable |
| Ryzen AI installer "Conda not found" despite conda working | 🟡 Low | ✅ System PATH + full restart |
| Missing `4x4.xclbin` firmware file in installer package | 🟡 Low | ⏳ Known AMD packaging gap, not fixed (moot — see below) |
| Hawk Point (XDNA1) has no LLM support in Ryzen AI 1.8.0 software stack | 🟡 Low | ❌ Hardware-generation limit, unfixable |

**Net result:** NPU path abandoned entirely. The Ollama-on-iGPU pipeline
(`news_prefilter.py`, Radeon 780M via Vulkan) remains the correct and only
viable local-LLM setup on this hardware, and was left untouched throughout.

---

## Cleanup

Since the NPU path has no path forward on this hardware, everything installed
specifically for it was removed to reclaim disk space:

```powershell
conda env remove -n ryzen-ai-1.8.0 -y          # ~2-3GB
Remove-Item "C:\Program Files\RyzenAI" -Recurse -Force   # 6.05GB, required admin elevation
Remove-Item "C:\Users\jadek\Downloads\ryzen-ai-npu" -Recurse -Force  # ~3.5GB installer + driver zip
```

Kept installed: Visual Studio 2022, CMake, Miniforge — general-purpose dev
tools unrelated to Ryzen AI specifically, no reason to remove them.

---

## Timeline

| Time | Event |
|---|---|
| ~02:20 | Ollama + iGPU pipeline set up and verified working (`100% GPU`) |
| ~02:45 | Asked whether the NPU could run the same workload |
| ~02:50 | Confirmed hardware: Ryzen 7 260 = Hawk Point = XDNA1, 16 TOPS |
| ~02:55 | Lemonade SDK ruled out — requires XDNA2 |
| ~03:00 | Copilot+ PC ruled out — needs 40+ TOPS |
| ~03:00–03:15 | Verified Ryzen AI Software supports Hawk Point on paper; installed VS2022, CMake, Miniforge |
| ~03:19–03:29 | Four failed installer attempts, all "Conda not found," despite conda working standalone |
| ~03:18 | First explorer.exe restart — not sufficient |
| ~03:2x | First full system restart — still not sufficient (installer runs elevated, needs System PATH) |
| ~03:24 | Root cause found: System PATH missing conda, elevated process can't see User PATH |
| ~03:2x | Manually added conda to System PATH via `sysdm.cpl`, second full restart |
| ~03:30–03:40 | Installer completed successfully end to end |
| ~03:42 | `quicktest.py` ran but logged `Failed to open xclbin` — missing firmware file |
| ~03:45 | Found AMD release notes: Hawk Point limited to CNN INT8, no LLM support at all |
| ~03:46 | Decision: abandon NPU path, keep iGPU pipeline as-is |
| ~03:47 | Cleaned up ~9GB of NPU-specific installs |
