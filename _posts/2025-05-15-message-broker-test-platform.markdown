---
layout: post
title:  "Building a Reproducible, Plugin-Based Test Harness for Tactical Networks"
date:   2025-05-15 10:00:00 +0000
categories: blog
---

## The problem
I have had to explain this platform to several people and since the wiki repo is internal to my organization I decided to write this short post to provide an introduction to the protocol
test harness I developed to evaluate message brokers in constrained environments.

We needed a system that could:

* Support protocol implementations from different organizations (without sharing code)
* Let researchers collaborate without having to disclose internals of their proprietary solutions
* Keep experiments consistent, reproducible, and easy to extend

## The Solution: Plugins
We built a plugin-based architecture. Every protocol lives in its own module, completely isolated from the main test logic. The name of the protocol is specified at runtime and a class that implements the harness interface is loaded from a default directory.

At first, I used an open-source plugin library—but it quickly broke with newer Gradle versions. So, I ended up writing a simple reflection-based 
plugin loader. It turned out to be faster, easier, and way more maintainable.

The core test driver doesn’t care what protocol it’s running—each one just plugs in, runs, and logs results through a shared interface.

## What the Harness Does
We designed the harness to:
* Generate messages of custom size, type, and frequency
* Log all message delivery events for later analysis
* Work with EMANE, a well known tactical network emulator
* Scale to 24–96 concurrent nodes (realistic small deployment sizes)
* Stream logs to a central StatServer
* Automatically generate post-test summary stats

Exceptions: For some very experimental protocols (such as our last experiment with Hyperledger) we had to implement the driver logic inside the "hyperledger nodes" themselves,
we still recicled the stat server logic saving a big chunk of time. 

## Architecture Overview

![Test Harness Architecture](https://github.com/rFronteddu/rfronteddu.github.io/blob/main/img/test_harness.png?raw=true)


At the heart of the system is a Java platform that handles:
* Plugin loading and setup
* Message scheduling and injection
* Event logging (send/receive/timestamp) via NATS
* Consistent configuration across all test runs

Each protocol is treated like a black box. The same message schedule and network setup is reused for every run—so comparisons are fair, apples-to-apples.

All logs are timestamped and sent to the StatServer using NATS through a separate network interface, which ensures low-latency delivery with minimal jitter. Times are
recorded on the NATS VM.

### Message Types
We modeled our messages after common military communication patterns:

* BF (Blue Force) – 128 bytes: Periodic status updates broadcast to everyone
* SD (Sensor Data) – 1 KB: Medium-sized messages sent to a subset of nodes
* HQR (Headquarters Reports) – 1 MB: Big reports from HQ to a few gateway nodes

Each node runs a Java tool that generates and sends these messages on a schedule. UUIDs help correlate events and measure delivery.

### Emulated Tactical Network
We didn’t just simulate the messages—we simulated the network itself.

Using EMANE (Extendable Mobile Ad-hoc Network Emulator), we recreated wireless conditions like mobility, interference, and packet loss. On top of that, we used the [Anglova](https://anglova.net/) scenario to define realistic battlefield movement and communication patterns.

Node roles:
* **Leaders** send SD: HQ sends HQR
* **Gateways** receive HQR and SD
* **Everyone** broadcasts BF

We also captured raw packet traces with tcpdump and pulled logs from each node after tests to generate richer post-experiment metrics and visuals.

### What We Measured
Each protocol run was analyzed for:
* **Delivery Ratio** – % of messages that made it to their destination
* **Latency** – How long it took messages to arrive
* **Bandwidth Usage** – Total bytes over the network
* **Cost per Message** – Network + protocol overhead per payload byte

All runs used the exact same network conditions and message schedules. So when we compared protocols, we knew the results were fair and reproducible.

### Tested Protocols
* NATS
* MQTT
* Disservice
* GDEM
* RabbitMQ
* Kafka
* NDN
* OpenDDN
* Hyperledger Fabric
* NATS
* MQTT
* XMPP
* RTI DDS
* ZeroMQ
* Redis
* JGroups
* TamTam
* Edgware

