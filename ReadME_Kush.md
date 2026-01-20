# LiveKit Intelligent Interruption Handling (Backchannel Filter)
**Submitted by:** Kush Tokas | **Roll No:** 2023296 | **College:** IIITD


This repository implements a **context-aware interruption handling layer** for a LiveKit voice agent.

## Problem

LiveKit’s default VAD (Voice Activity Detection) is very sensitive.
When the agent is speaking and the user says short **backchannel acknowledgements** like:

* `yeah`, `ok`, `aha`, `hmm`, `uh-huh`, `right`

…the agent incorrectly treats it as an **interruption** and stops speaking abruptly.

This creates a poor real-time conversational experience.

---

## Goal

Implement a logic layer that distinguishes between:

### ✅ Passive Acknowledgement (Backchannel)

While the agent is speaking:

* `"yeah"`, `"ok"`, `"hmm"`
  ➡️ **Ignore** and continue speaking seamlessly (**no pause / no hiccup**)

### ✅ Active Interruption

While the agent is speaking:

* `"stop"`, `"wait"`, `"no"`
  ➡️ **Interrupt immediately** and listen to user

### ✅ Normal Behavior When Agent is Silent

When the agent is silent:

* `"yeah"` is treated as valid input
  ➡️ Agent responds normally

---

## What Was Implemented

### 1) Configurable Ignore List (Backchannel Words)

A configurable list of **soft inputs** was added (example):

```python
["yeah", "ok", "hmm", "right", "uh-huh"]
```

These are treated as *non-interrupting* **only when the agent is actively speaking**.

---

### 2) State-Based Filtering

The filter checks the agent state:

* If `agent is speaking` → backchannels are ignored
* If `agent is silent` → backchannels are treated as valid user input

This ensures correct conversational flow.

---

### 3) Semantic Interruption Handling (Mixed Sentences)

If the user says something like:

> `"Yeah okay but wait"`

Even though it starts with backchannel words, it contains a real command (`wait`) → **interrupt immediately**.

So mixed inputs are correctly classified as **active interruptions**.

---

### 4) No VAD Kernel Modification

⚠️ The low-level VAD kernel is NOT modified.

Instead, this is implemented as a **logic layer** in the agent event loop using STT transcript signals.

---

## Hiccup / False Interruption Prevention (Key Production Fix)

### Why hiccups happen

VAD triggers faster than STT.

So the sequence becomes:

1. User says `"yeah"` while agent is speaking
2. VAD fires instantly → agent pauses/stops
3. STT arrives later → we realize it was only `"yeah"`
4. Too late → agent already paused → **hiccup occurs**

### Fix Strategy

A **deferred interrupt guard** was added:

* If VAD fires but STT transcript is not ready yet,
  we wait for a **tiny real-time window** (example ~250ms).
* If STT resolves to a backchannel → ignore interruption entirely.
* If STT resolves to a real command → interrupt normally.

This ensures:

✅ No stutter
✅ No pause
✅ No resume glitch
✅ Seamless speech continuity

---

## Files Added / Modified

### ✅ `intterrupt_audio_backend.py`

Contains the interruption filter logic:

* token normalization
* backchannel ignore list
* semantic interruption detection
* punctuation-safe matching

### ✅ `interrupt_tests.py`

Contains test cases that validate all core requirements:

* ignore backchannels while speaking
* interrupt on real commands
* respond to backchannels while silent
* handle mixed input like `"yeah wait"`

### ✅ `agent_activity.py`

Minimal integration changes:

* adds `InterruptionHandler`
* uses the filter during interruption decision logic
* includes deferred interrupt guard to prevent hiccups

---

## How To Run Tests

From the project directory:

```bash
python3 interrupt_tests.py
```

Expected output:

* All tests should pass (`116/116`)

---

## Behavior Matrix (Expected)

| User Input           | Agent State | Expected Behavior                     |
| -------------------- | ----------- | ------------------------------------- |
| "Yeah / Ok / Hmm"    | Speaking    | IGNORE (agent continues seamlessly)   |
| "Wait / Stop / No"   | Speaking    | INTERRUPT immediately                 |
| "Yeah / Ok / Hmm"    | Silent      | RESPOND normally                      |
| "Start / Hello"      | Silent      | RESPOND normally                      |
| "Yeah wait a second" | Speaking    | INTERRUPT (semantic command detected) |

---
## Summary

This solution prevents unwanted interruptions from filler words **without modifying VAD**, by using:

* state-aware filtering
* semantic interruption detection
* deferred interruption guard to avoid false-stop hiccups

This ensures a smooth, real-time voice conversation experience.

---


