---
title: "Jim Wood"
date: 2026-03-09
description: "Resume for Jim Wood — Golang / Backend Developer"
layout: "resume"
showDate: false
showReadingTime: false
showWordCount: false
showAuthor: false
showTableOfContents: false
showPagination: false
draft: false
---

{{< resume-header name="Jim Wood" >}}
| | |
|---|---|
| {{< icon "phone" >}} | [519-998-0694](tel:+15199980694) |
| {{< icon "envelope" >}} | [jimwood1337@gmail.com](mailto:jimwood1337@gmail.com) |
| {{< icon "linkedin" >}} | [linkedin.com/in/woodjp](https://linkedin.com/in/woodjp) |
| {{< icon "github" >}} | [github.com/wood-jp](https://github.com/wood-jp) |
| {{< icon "globe" >}} | [jimsopinion.xyz](https://jimsopinion.xyz) |
{{< /resume-header >}}

---

## Overview

Backend software engineer with over a decade of professional experience designing and implementing complex, interconnected systems across multiple stakeholder groups. Primary language is Go, with hands-on experience in distributed systems, event-driven architectures, blockchain infrastructure, and real-time data pipelines. Proven ability to lead technical initiatives from research through production, often at startups where wearing many hats is the norm.

**Core skills:** Go · Kafka · NATS JetStream · Docker · Docker Bake · GitHub Actions · CI/CD · NATS · REST APIs · PostgreSQL · Terraform · Blockchain (L2 / EVM) · Supply Chain Security

---

## Experience

<div class="keep-together">

### Zircuit — Golang Engineer

*Sep 2022 – Feb 2026 · Fully Remote (Waterloo, ON)*

</div>

- Designed and implemented a proof-orchestration pipeline using NATS JetStream, enabling independent prover services to run in parallel while guaranteeing correctness and failure recovery, minimizing per-block proving time.
- Forked and merged the Optimism and Scroll geth codebases into a unified Layer 2 blockchain foundation, resolving conflicting geth baselines and integrating Rust-built zero-knowledge components callable from Go at runtime.
- Built and maintained the organization's multi-architecture container build system — first with Earthly (including a dockerized Rust/Go hybrid builder with Chainguard base images), then overhauled to Docker Bake with multi-stage builds when Earthly shut down — integrated throughout GitHub Actions CI/CD for consistent, supply-chain-secure builds.
- Designed and implemented a blockchain explorer API service with a generic cursor-based pagination system, allowing future developers to add new endpoints with minimal boilerplate.
- Designed and implemented an automated open-sourcing pipeline (with human gating) to publish Zircuit code while stripping proprietary content and preserving developer privacy; required cross-functional coordination across management, security, devops, and multiple engineering teams.
- Defined and implemented semantic release automation across all services, including PR metadata enforcement and post-merge release workflows via GitHub Actions.

<div class="page-break"></div>

<div class="keep-together">

### Adentro — Principal Developer

*Jul 2016 – Sep 2022 · Waterloo, ON*

</div>

- Led a six-person team to redesign and replace a faulty real-time data pipeline live in production — first resolving conflicting stakeholder requirements through facilitated cross-team meetings, then designing new algorithms and managing a multi-month implementation with zero data loss during cutover.
- Rewrote the email sending service from Python to Go with seamless Mailgun and SendGrid interoperability; the new service delivered over 2 billion emails during tenure.
- Implemented Kafka streaming libraries in Go and migrated all Scala data pipeline services to Go, including co-partitioned key-value stores and automatic partition-assignment detection.
- Designed and implemented full GDPR and CCPA compliance across the platform, spanning the email service, all databases, and the real-time tracking pipeline — covering data retrieval, deletion, and suppression of future tracking on behalf of end users and business customers.
- Resolved critical portal performance by redesigning the data model for business tree representation, reducing page fetch complexity from O(n) to O(1).
- Separated a monolithic router API into per-type services for improved scalability and robustness; enhanced the primary tracking algorithm to correctly handle out-of-order events and transient outages.

<div class="keep-together">

### Sandvine — Software Developer

*Aug 2015 – Jul 2016 · Waterloo, ON*

</div>

- Developed novel network security solutions on the Network Security team to defend against large-scale internet attacks.
- Participated in a DevOps working group that improved tooling and the development environment for all of Sandvine Engineering.

<div class="keep-together">

### Avvasi — Software Developer

*Aug 2014 – Jul 2015 · Waterloo, ON*

</div>

- Designed and implemented the licensing system used across all Avvasi products during the company's transition from physical hardware to a virtualized environment.
- Maintained and enhanced the platform layer bridging physical hardware and company software, including CLI tooling, alarms, system-level tests, and OS-level components.

---

## Education

**Bachelor of Computer Science** — University of Waterloo, 2014
*Minor: Combinatorics and Optimization*
