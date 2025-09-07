---
date: 2025-08-16
categories:
  - devops
  - slackbot
  - kubernetes
  - python
  - hackathon
---
# Reviving Doraemon: A Slack Bot’s Second Life in Kubernetes
##### originally posted at [LinkedIn](https://www.linkedin.com/pulse/reviving-doraemon-slack-bots-second-life-kubernetes-janus-chung-nzync/) at Aug 16, 2025

![Automation bots have evolved. What’s next?](../../assets/blog/doraemon/banner.jpg)

Some projects stick with you. For me, it was a little Slack bot I hacked together at a previous job—something that could talk to our infrastructure and give quick answers without switching tools. I never learned what happened to it. Layoffs came. From what I later heard, it wasn’t adopted. It felt like watching a small idea I cared about slowly disappear.

Fast-forward to Flagler. I mentioned the bot almost off‑hand, unsure if anyone would care. My boss immediately supported the idea, and that gave me the energy to bring it back. This post is about reviving that project—this time with intent, care, and a proper home in Kubernetes.

<!-- more -->

## Meet Doraemon

> Fun fact: Doraemon takes its name from a beloved Japanese cartoon robot cat who pulls out gadgets to solve everyday problems. Our bot carries the same spirit—always ready with a tool or an answer when you need it most.

Doraemon is a Kubernetes‑native Slack bot, built with the slack_bot library in Python 3.11. It lives right in Slack, where conversations already happen, and brings clarity without forcing people to change context. What started as a hackathon experiment is now something I maintain proudly as a senior DevOps engineer—small, sharp, and genuinely useful.

When someone greets the bot, it replies with a menu of its capabilities. It’s friendly, obvious, and lowers the barrier for anyone to use it.

## What Doraemon Can Do

### 1. Infrastructure Version Report

__Purpose:__ Show what’s deployed versus what’s available for key infrastructure components—Argo CD, Sealed Secrets, Airbyte, Argo Rollouts, and more.

__Impact:__ Doraemon queries live sources like GitHub Releases, Helm chart indices, and Kubernetes manifests, then shows the difference between current and available versions. No dashboards, no delays—just one straight answer in Slack. For me as a DevOps engineer, this means I can surface upgrade opportunities to the team without context switching, and for everyone else it means visibility they never had before.

### 2. App Version Check

__Purpose:__ Answer the everyday question: “What version is deployed?”

__Impact:__ Doraemon understands natural prompts—`app version?`, `what’s our app version?`—and replies with the deployed version number. It’s not flashy, but it saves time in standups and avoids unnecessary back‑and‑forth.

### 3. Restaurant Recommendations

__Purpose:__ Help teammates flying into the office and unfamiliar with the city feel at home.

__Impact:__ Ask `what to eat?` and Doraemon suggests nearby restaurant options with a map. It may sound lighthearted, but it has real value—when someone lands in town for a sprint, they don’t waste time scrolling review apps. It shows that even a DevOps‑driven tool can make the human side of work smoother.

### 4. Access Codes

__Purpose:__ Replace tedious, manual back‑and‑forth requests for access codes.

__Impact:__ Instead of waiting on IT or ops, users can request codes directly in Slack. Doraemon validates and delivers them securely, reducing interruptions. As someone who used to field those requests, this feature is a relief—it gives time back to ops while giving users what they need instantly.

## Why Slack?

Because that’s where the conversations are. By answering questions directly in‑thread, Doraemon keeps the loop tight. No context switching, no dependency on who has the right tool open—the information becomes part of the conversation and the audit trail.


## Finding My Closure

I built the original during a hackathon at a previous job. It was never adopted, and after the layoff, I thought that chapter was closed. Getting the chance to bring it back at Flagler—with real support—meant more than just recycling old code. It was personal. It was proof that even small, unfinished ideas can grow into tools the team actually uses.

To my boss and teammates at Flagler: thanks for backing this. From the outside, Doraemon may look like a small Slack bot. But from my perspective as a DevOps engineer, it’s also a story about resilience—about not letting useful ideas die, and about finding ways to make work better for everyone.

If my hackathon‑self could see this now, they’d recognize the heart—but also the rigor. And maybe they’d smile at the name too—just like the cartoon Doraemon always had the right gadget for the moment, our bot is there with the right answers when we need them.
