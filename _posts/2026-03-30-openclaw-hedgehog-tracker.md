---
layout: post
title: "AI for Hedgehogs"
subtitle: "Tracking hedgehog activity via sensors, local LLMs, and OpenClaw"
date: 2026-04-29 09:00:00 -0400
background: '/asset/images/reggie-background.jpg'
tags:
  - homelab
  - openclaw
  - home
---

## Why?

If you have a pet hedgehog, or really any small nocturnal animal, you probably know the strange feeling of going to bed while they are just starting their day. Reggie, my hedgehog, spends most of the daytime tucked away asleep, but at night she wakes up, explores, and runs on her wheel. Recently, as a sensing and systems person, I've been laying in bed thinking: how can i build a system to capture and recap what she does at night?

Through this course, that idea has grown into:  could I track Reggie’s wheel activity using a simple sensor and then ask questions about it later? And then through the rise of AI and agents: how can I turn raw sensor signals into meaningful, queryable information about behavior? And how can we set this up with a agent in between the sensor <> the interaction to retrieve information? 

In my research, this type of problem shows up in mobile and environmental sensing. Sensors collect noisy, indirect signals from the real world, and the challenge is to transform those signals into something interpretable. Here, the environment just happens to be a hedgehog wheel at 2 a.m. The scale is smaller, but the pipeline is similar: **sensing → representation → reasoning → interaction**

The final system uses a magnetic hall effect sensor to detect wheel rotations, a Jetson Orin Nano to log activity, a local PC running Ollama for language model inference, with OpenClaw to expose the whole thing through a Discord bot. Everything runs locally, without cloud APIs or subscription costs.

But first, let me show you Reggie. In this picture she's enjoying her bath from the safety of her float... not a fan of water!

<img src="{{ '/assets/reggie-bath.jpg' | relative_url }}" width="400">

---

## Main Ideas

This project has three core components:

1. A magnetic hall effect sensor on a hedgehog wheel to capture activity data.
2. Local LLM inference through Ollama, making it possible to ask natural language questions without sending data to the cloud.
3. OpenClaw as the glue between local models, memory, tools, and a Discord interface.

The result is a small, local sensing system that lets you ask questions like:

- “Was Reggie active last night?”
- “When did she start running?”
- “How much did she run compared to yesterday?”
- "What is the Reggie's weekly mileage?" 

and get a response based on real sensor data rather than guesswork.

---

## System Overview

At a high level, the system takes a physical event — the wheel rotating — and turns it into something conversational.

```text
Wheel rotation
    ↓
Magnet passes hall effect sensor
    ↓
GPIO pulse detected by Jetson
    ↓
Timestamped event logged in CSV
    ↓
OpenClaw indexes / retrieves relevant information
    ↓
Discord bot answers natural language questions
```

The important part is that the LLM is not directly reasoning over raw electrical pulses. Instead, the sensor data is first converted into structured activity information: timestamps, rotation counts, and activity windows. Then Openclaw creates and runs a python script to get the data that the language model reasons for to get the targeted summary based on the question.


---

## Hardware Setup


![Implementation Diagram]({{ '/assets/implementation.png' | relative_url }})


### Wheel Sensor

The core hardware is simple: a small magnet and a magnetic hall effect sensor.

The magnet is glued to the hedgehog wheel. The hall effect sensor is nearby on a 3D printed holder, close enough that each full wheel rotation causes the magnet to pass the sensor. Every time the magnet passes, the sensor produces a signal that can be detected by the Jetson’s GPIO pins.

From the rotation count, we can estimate:

- how often Reggie runs,
- when she starts and stops running,
- how long her active periods are,
- approximate distance traveled,
- and rough running intensity.

This approach is intentionally minimal. I didn't need a camera, computer vision model, or complicated wearable device. The wheel already constrains the motion into a repeated circular event, which makes it a perfect target for a simple magnetic sensor.

![teaser image]({{ '/assets/teaser.png' | relative_url }})

### Why Not a Camera?

A camera would provide richer data, such as whether Reggie is eating, exploring, or running. But it also introduces problems:

- it needs lighting or infrared illumination,
- it produces much more data,
- it raises more privacy concerns,
- and it requires more compute to interpret.

This is a useful sensing lesson in general: the best sensor is not always the richest one. Sometimes the best sensor is the one that captures the specific behavior you care about with the least ambiguity.

---

## Edge Device: Jetson Orin Nano

The Jetson Orin Nano, named “Og” in my homelab, acts as the edge device (all of my homelab devices are named from the *Ready Player One* series). It handles the physical sensing side of the system.

Its responsibilities are:

- reading the wheel sensor signal,
- timestamping rotation events,
- writing logs locally,
- running OpenClaw,
- and connecting to Discord.

The Jetson sits on my local homelab subnet - intentionally kept on a separate local network to make the setup easier to manage and keep the system contained.

For the local models, I tested on the Jetson since it can technically run small models itself, but it would timeout during most of the responses with the discord bot and the message itself had to be minimal. So instead, it acts as the always-on sensing and orchestration node, while my local PC with a nice GPU handles the LLM workload.

---

## Local LLM Inference with Ollama

The language model runs on my main PC using Ollama. The Jetson talks to this PC over the local network.

The two main models are:

- `qwen2.5:7b` for natural language reasoning,
- `nomic-embed-text` for semantic embeddings and memory search.

This split-compute architecture works well for this type of project. The Jetson stays focused on sensing and logging, while the PC handles the heavier model inference.

The benefit is that everything remains local. I do not need to send Reggie’s activity data to a cloud API, and I do not pay per request. The main tradeoff is that the system depends on my local network and the PC being available.

![teaser image]({{ '/assets/models.png' | relative_url }})

---

## From Rotations to Behavior

A wheel rotation by itself is just a pulse. The more interesting question is how those pulses become behavior.

The raw log might look something like this:

```text
2026-04-28 01:12:03 → rotation
2026-04-28 01:12:04 → rotation
2026-04-28 01:12:05 → rotation
2026-04-28 01:12:07 → rotation
```

That is useful, but not very readable. So the next step is to aggregate raw events into higher-level features.

From the timestamped pulses, we can compute:

- **rotation count** over a time window,
- **rotations per second** or rotations per minute,
- **activity bouts**, which are continuous periods of wheel motion,
- **inactive gaps**, where no rotations occur for some threshold of time,
- **estimated distance**, using wheel circumference,
- and **nightly summaries**.

A summarized activity bout might look like this:

```text
Activity bout:
start: 01:12
end: 01:47
duration: 35 minutes
average speed: 1.8 rotations/sec
estimated distance: 320 meters
```

This representation is what makes the AI layer useful. The LLM is not trying to infer behavior from individual GPIO events. It receives a cleaner representation of activity and can answer questions about patterns over time.

![mile chart]({{ '/assets/chart.png' | relative_url }})

---

## Software Stack

OpenClaw connects the sensor logs, local models, memory, and Discord interface.

The Jetson runs OpenClaw and points to the Ollama server on my PC. A simplified version of the configuration looks like this:

```json
{
  "models": {
    "providers": {
      "ollama": {
        "baseUrl": "http://192.168.8.226:xxxxx",
        "models": [
          { "id": "qwen2.5:7b" },
          { "id": "nomic-embed-text:latest" }
        ]
      }
    }
  },
  "channels": {
    "discord": {
      "enabled": true,
      "dmPolicy": "allowlist",
      "guilds": {
        "[your-guild-id]": {
          "requireMention": false,
          "channels": {
            "[your-channel-id]": { "enabled": true }
          }
        }
      }
    }
  }
}
```

With `requireMention: false`, I can type naturally in the Discord channel without tagging the bot every time.

The full software flow is:

```text
Sensor pulse
    ↓
Python logger
    ↓
CSV / local activity store
    ↓
OpenClaw tool or memory layer
    ↓
Embedding search with nomic-embed-text
    ↓
Reasoning with qwen2.5:7b
    ↓
Discord response
```

![discord bot]({{ '/assets/discord-bot.png' | relative_url }})


---

## Why Discord?

Since I use discord pretty much 24/7, having the interact in my personal server makes it easy to interact with (and later we can have her overnight stats autogenerate every morning).  With discord, I can ask questions from my phone or desktop, and the system responds in the same place I already use for other homelab bots and experiments.

This is where the project becomes more than a data logger. The point is not just to collect data, but to make the data easy to query.

---

## Design Decisions and Tradeoffs

### Hall Sensor vs. Camera

The hall sensor is simple and reliable, but it only captures one type of behavior: wheel motion. A camera could capture more, but would require more compute, more storage, and more complicated interpretation.

For this first version, the hall sensor was the right level of complexity.

### Jetson vs. Microcontroller

A microcontroller could count wheel rotations easily. In fact, for only the sensing part, a microcontroller would be more than enough. I used the Jetson because this project was also about integrating sensing with an AI agent stack. The Jetson gives me a Linux environment, network access, GPIO, Docker support, and enough flexibility to run OpenClaw and related services.

### Local Models vs. Cloud APIs

Cloud APIs would probably provide stronger reasoning, but they add cost, latency, and external dependency. Local inference keeps the system private and free to run continuously.

The tradeoff is that local models are smaller and sometimes less reliable. A 7B model can answer many structured questions well, but it is not perfect and the message can't be super long. 

### Split Compute vs. All-in-One Edge

The Jetson could potentially run the whole stack, but offloading inference to the PC gives better performance and keeps the edge device lighter. The tradeoff is network dependency. If my PC is offline or unreachable, the sensing system can still log data, but the conversational interface will not work as there is not model.

![model unavailable]({{ '/assets/no-model.png' | relative_url }})


---

## What Worked Well

The most successful part of the project was the interaction layer.

Being able to ask:

```text
Was Reggie active last night?
```

or:

```text
When did she run the most?
```

is much more useful than manually reading logs.

The semantic search layer also helps. A normal keyword system would need queries to match the exact field names or log format. With embeddings, the system can connect a natural question like:

```text
Did she have a busy night?
```

to activity summaries involving duration, distance, and number of running bouts.

That moves the system from simple data retrieval toward data interpretation.

---

## Limitations and Failure Modes

This system works, but it is not perfect.

### Sensor Ambiguity

A wheel rotation does not always mean intentional running. Reggie could bump the wheel, step on it briefly, or start and stop without completing a normal running bout.

The system measures wheel motion, then infers activity from it. That distinction matters.

### Lack of Ground Truth

There is no direct validation that every detected activity bout corresponds to actual running. A camera or manual observation could provide ground truth, but I intentionally avoided that complexity in this version.

### LLM Reasoning Limits

The local model can occasionally overinterpret the data. For example, it might describe a night as “unusual” without enough historical context. This is why the system should provide summaries grounded in the logs rather than making unsupported claims.

### Temporal Resolution

Aggregating data into activity windows makes it easier to reason over, but it loses fine-grained detail. The right level of aggregation depends on the question being asked.

These are the same kinds of issues that appear in larger sensing systems. Sensors rarely measure the thing we care about directly. They measure a proxy, and the system has to carefully translate that proxy into an interpretation.

---

## What I Would Add Next

There are several directions that would make the system more useful.

### Better Activity Summaries

The next version could generate automatic daily summaries, such as:

```text
Reggie started running at 12:43 a.m., had 4 major activity bouts, and ran an estimated 1.2 km total. Her longest bout was 31 minutes.
```

### Visual Dashboard

Discord is good for questions, but plots are still useful. I would like a small dashboard showing nightly activity trends, distance estimates, and start/stop times.

### Additional Sensors

Possible additions include:

- load cell under food bowl,
- temperature and humidity sensor near enclosure (she needs 72f and has a heat lamp),
- passive infrared motion sensor.

### Stronger Behavioral Models

With more data, the system could learn typical patterns and flag deviations. For example, it could identify nights where Reggie runs much less than usual, possibly indicating health concerns.

---

## Wrap-up

This project started with a simple question: what does my hedgehog do at night?

The answer turned into a small local AI sensing system. A magnet and hall effect sensor capture wheel activity. A Jetson logs the data. A local PC runs the language model. OpenClaw connects everything to Discord.

More broadly, this project shows a different way to interact with sensor data. Instead of only building dashboards or manually checking logs, we can build systems that let us ask questions about real-world activity.

At larger scales, the same pattern applies to environmental sensing, smart homes, animal monitoring, and other cyber-physical systems: collect signals, structure them, reason over them, and make them accessible through an interface people actually want to use.

---

I hope you enjoyed reading about this project! Ideally, in the coming months I will make the code repository and a public-facing dashboard available. In reality, PhD life can get very busy at times -- stay tuned :) 
