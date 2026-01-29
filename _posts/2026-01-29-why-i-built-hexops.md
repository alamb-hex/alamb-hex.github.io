---
layout: post
title: "Why I Built HexOps: A CVSS 10.0 Wake Up Call"
date: 2026-01-29
categories: [development, tools, security]
tags: [hexops, devops, patch-management, cve, react, nextjs, open-source]
author: Aaron Lamb
description: "A critical React vulnerability exposed a gap in developer tooling. Here's why I built HexOps, a free patch management dashboard for developers managing multiple projects."
---

# Why I Built HexOps: A CVSS 10.0 Wake Up Call

December 3rd, 2025. I'm checking the news over coffee when I see it: "Critical Security Vulnerability in React Server Components."

CVE-2025-55182. CVSS 10.0. Pre-authentication remote code execution. No workarounds. Patch immediately.

I manage 22 active projects. Most of them run Next.js with React Server Components. Some are client projects. Some are internal tools. Some are products generating revenue.

My morning just changed.

Within hours, CISA added CVE-2025-55182 to their Known Exploited Vulnerabilities catalog. Security researchers confirmed China-nexus threat groups were already exploiting it. By the end of the week, a botnet called RondoDox was actively scanning for unpatched servers.

This wasn't theoretical. This was happening.

I started the patch rotation. Open project. Check React version. Run pnpm update. Verify the build. Test critical paths. Commit. Deploy. Move to the next project.

Twenty-two times.

It took most of the day. Not because patching is hard. Because doing it twenty-two times is tedious, error-prone, and completely manual.

![HexOps Dashboard](https://alamb-hex.github.io/assets/images/hexops-dashboard.jpg)

## The Real Problem

Here's the thing: CVE-2025-55182 wasn't special. It was just loud.

Critical vulnerabilities drop constantly. Most don't make headlines. They show up in `pnpm audit` output that you ignore because you're busy shipping features. They accumulate in your node_modules like technical debt with an expiration date.

The React2Shell incident forced me to confront something I'd been avoiding: I had no system for managing patches across projects.

Each project lived in its own directory. Its own terminal tab. Its own mental context. To check if a project was vulnerable, I had to cd into it, run the audit command, read the output, decide what to do.

Multiply that by 22 projects and you get a full day of context switching just to answer the question: "Am I exposed?"

And that's just visibility. Actually applying patches meant repeating the process with update commands, build verification, and deployment. Every project. Every time.

This isn't a tooling problem unique to me. Every developer managing multiple projects faces this. The difference is most people manage 2 or 3 projects. I was managing 22. The pain was impossible to ignore.

## What I Built

Before I started building, I looked around. Surely someone had solved this.

PM2 handles process management for Node apps. It's solid for running production services but it's not what I needed for local development across dozens of projects with different frameworks and package managers.

I stepped back and thought about what I actually wanted.

My first job in tech was at a web hosting company. I spent years working with WHM and cPanel. Say what you want about those tools, but they gave you one interface to manage hundreds of sites. Check status, restart services, view logs. All without SSH-ing into individual accounts.

Later, as a sysadmin, I used Red Hat Satellite for patch management across enterprise Linux fleets. Scan for vulnerabilities, see what's affected, push updates in batches. Visibility and control at scale.

More recently, working in DevOps, I leaned on Portainer for container management. One dashboard to see all your containers, start and stop them, check resource usage.

Each of these tools solved a similar problem in different contexts. Centralized visibility. Batch operations. Reduced context switching.

Nothing combined these ideas for local development. No "cPanel for your projects folder." No Satellite for your node_modules. No Portainer for your dev servers.

So I built it. HexOps is the brain child of all those experiences. A control panel, dashboard, and patch management solution for developers managing multiple local projects.

The core idea is simple: one interface that shows every project, its status, and its patch health. No more cd-ing into directories. No more running audit commands in 22 terminals. No more spreadsheets tracking which projects got patched.

HexOps watches your project directories. It knows which projects are running, which have outdated packages, and which have known vulnerabilities. Everything in one view.

The patches dashboard was the feature I needed most. It pulls vulnerability data from npm audit and outdated package info from your package manager. Then it sorts everything by severity. Critical issues float to the top. You see the worst problems first.

From there, you can update packages individually or in batches. Select five projects with the same vulnerable dependency, update them all. HexOps handles the package manager commands, you review the results.

It tracks patch history too. Every update gets logged with timestamps, success status, and output. When your security team asks "when did we patch CVE-2025-55182?" you have an answer.

I added other things along the way. An integrated terminal so you don't need a separate window. System health monitoring for CPU and memory. Project start/stop controls. Git integration for committing patches with generated messages.

But the patches dashboard is why HexOps exists. Everything else is convenience. Patch visibility is the point.

## Proof It Works: This Week

Three days ago, CVE-2026-23864 dropped. Another React and Next.js vulnerability. Denial of service via memory exhaustion. Not as severe as React2Shell, but still needs patching.

This time was different.

I opened HexOps. The patches dashboard already showed which projects were affected. I selected all of them, clicked update, reviewed the results. Committed the changes with generated messages. Done.

The whole process took maybe 30 minutes. Most of that was waiting for builds to verify. No context switching. No hunting through directories. No wondering if I missed one.

That's the difference tooling makes. December was a full day of scrambling. January was a coffee break.

## Why Give It Away?

I could keep HexOps internal. It solves my problem. Job done.

But the React2Shell incident affected a lot more than my 22 projects. Wiz reported that 39% of cloud environments contained vulnerable React or Next.js instances. Shadowserver counted 90,000 exposed servers weeks after the patch was available.

That's not because developers don't care about security. It's because checking 22 projects is annoying and checking 220 is impossible without tooling.

The ecosystem has a visibility problem. Developers build things, dependencies accumulate, vulnerabilities appear, and nobody notices until a CVSS 10.0 makes headlines. Then everyone scrambles.

Better tooling makes that scramble shorter. Or prevents it entirely.

I've benefited enormously from open source software. Next.js, React, Node, pnpm, the entire stack I build on. Releasing HexOps is partly about giving back. Someone else managing a pile of projects shouldn't have to build this from scratch.

It's also pragmatic. Unpatched projects become compromised projects. Compromised projects become attack infrastructure. The fewer vulnerable servers sitting around, the better for everyone. Including me.

Mostly, I want to help other developers keep their projects secured in a timely fashion. The tooling gap is real. If HexOps helps someone patch faster when the next critical CVE drops, that's a win for the whole ecosystem.

And honestly, I built this for myself. It works. I use it daily. Figured others might want it too.

## Get It, Use It, Break It

HexOps is available now on GitHub under the MIT license. Free to use, free to modify, free to ignore entirely if it's not for you.

One important note: HexOps is designed for local development. It runs on your workstation where you write code. It's not meant to be deployed on servers, in CI/CD pipelines, or anywhere facing the internet. It has no authentication, no access controls, and assumes you trust everyone on your local network. Keep it where it belongs: on the machine where you develop.

It runs on localhost, watches directories you configure, and stays out of your way until you need it. No cloud accounts. No telemetry. No subscriptions.

The documentation covers installation, configuration, and the feature set in detail. If you're managing more than a handful of projects and the CVE scramble sounds familiar, give it a look.

If you find bugs, open an issue. If you want features, open a discussion. If you build something better, let me know. I'll probably use it.

The React2Shell incident was a wake up call. Not just about one vulnerability, but about how unprepared most of us are when critical patches drop. Better tooling won't prevent the next CVE-2025-55182. But it can make the response a lot less painful.

That's why HexOps exists. That's why it's free.

---

**Project Links:**
- GitHub: [github.com/Hexaxia-Technologies/hexops](https://github.com/Hexaxia-Technologies/hexops)
- Documentation: Included in the repo

**Related Reading:**
- [React Security Advisory for CVE-2025-55182](https://react.dev/blog/2025/12/03/critical-security-vulnerability-in-react-server-components)
- [Next.js CVE-2025-66478 Advisory](https://nextjs.org/blog/CVE-2025-66478)
- [Akamai on CVE-2026-23864](https://www.akamai.com/blog/security-research/2026/jan/cve-2026-23864-react-nextjs-denial-of-service)

---

**About the Author:** Aaron Lamb is a founder of Hexaxia Technologies, a consultancy specializing in cybersecurity, infrastructure engineering, and AI product development. He's been building and breaking things in this industry for 30 years.
