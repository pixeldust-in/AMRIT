# AMRIT — Architecture Understanding & Audit

*A plain-language tour of how the platform is built, what's working, what's fragile, and what it could become.*

---

## How to read this document

This is a **map and a health-check** of the AMRIT platform, written so that *anyone* — engineer, product lead, funder, or new joiner — can understand it without prior context. Technical terms are explained the first time they appear, and there's a full **[glossary](#glossary-jargon-decoded)** at the end.

It is deliberately **not** a to-do list. The goal here is *understanding*: what the system is made of, what matters in each part, what's happening today, what challenges that creates, and what the future could look like. Concrete plans and sequencing come later, as a separate exercise.

Every factual claim about the code is based on a direct inspection of the public source repositories (build files, configuration, and actual code) carried out for this audit.

Each major part of the system is described in the same five-beat rhythm, so it's easy to navigate:

> **What it is** · **Why it matters** · **What's happening now** · **The challenge** · **The vision**

---

## Table of contents

1. [The 30-second version](#1-the-30-second-version)
2. [What AMRIT is, and the problem it solves](#2-what-amrit-is-and-the-problem-it-solves)
3. [The big picture: how the parts fit together](#3-the-big-picture-how-the-parts-fit-together)
4. [The parts, broken down](#4-the-parts-broken-down)
   - [4.1 The web applications (what staff click on)](#41-the-web-applications-what-staff-click-on)
   - [4.2 The backend services (the engine room)](#42-the-backend-services-the-engine-room)
   - [4.3 The mobile apps (the field force)](#43-the-mobile-apps-the-field-force)
   - [4.4 The data layer (the single source of truth)](#44-the-data-layer-the-single-source-of-truth)
   - [4.5 Interoperability (talking to the outside world)](#45-interoperability-talking-to-the-outside-world)
   - [4.6 Deployment & operations (how it actually runs)](#46-deployment--operations-how-it-actually-runs)
5. [The cross-cutting concerns (the things that touch everything)](#5-the-cross-cutting-concerns-the-things-that-touch-everything)
6. [The scorecard: where AMRIT stands](#6-the-scorecard-where-amrit-stands)
7. [The five risks that matter most](#7-the-five-risks-that-matter-most)
8. [The bright spot: a north star already exists](#8-the-bright-spot-a-north-star-already-exists)
9. [Three lenses: robustness, scalability, open-sourceability](#9-three-lenses-robustness-scalability-open-sourceability)
10. [The vision: what AMRIT could become](#10-the-vision-what-amrit-could-become)
11. [Glossary (jargon decoded)](#glossary-jargon-decoded)

---

## 1. The 30-second version

AMRIT is an **open-source digital health platform** that brings primary healthcare to millions of people in rural and underserved India — from a health worker knocking on a village door, to a clinic, to a specialist doctor consulting over video, to a national call-centre helpline. Everything revolves around **one shared patient record**.

Under the hood, it's built as **~30 separate software projects** that work together. The good news: the core technology is **more modern than you'd expect** for a platform this old, and at least one part of it is **genuinely excellent engineering**. The hard news: the platform behaves like *a system that assumes nothing will ever go wrong* — it has very few safety nets when the network is slow, a server dies, or data needs to scale across whole states.

The single most encouraging finding in this entire audit: **the team already knows how to build robust software** — one of their mobile apps proves it. The challenge isn't capability. It's **consistency**.

---

## 2. What AMRIT is, and the problem it solves

**AMRIT** stands for *Accessible Medical Records via Integrated Technologies*. It's built by **PSMRI** (Piramal Swasthya Management and Research Institute), a not-for-profit, and it carries a **verified "Digital Public Good" badge** — an international recognition that the software is built for public benefit and is free for any government or NGO to adopt.

To understand *why* it's shaped the way it is, picture the problem:

> A pregnant woman lives in a remote village. The nearest specialist doctor is hours away. Her only regular point of contact with the health system is an **ASHA** — a community health worker who visits on foot, historically carrying a paper register. There is no shared record, no easy way to reach a doctor, no reliable list of which medicines the local clinic has in stock, and no system to follow up.

AMRIT digitises and connects that entire chain. The defining idea is **continuity**: a woman registered by an ASHA at her door, later seen at a local clinic, referred to a specialist by video, prescribed medicine from a tracked inventory, and followed up by a phone helpline — is **one continuous record**, not six disconnected ones. That unification is the whole point.

**The scale is real and growing.** The platform is live in **Assam and Chhattisgarh** and is being scaled up to entire states. That ambition — going from "works in pilots" to "serves whole states reliably" — is the lens through which every finding in this document should be read.

---

## 3. The big picture: how the parts fit together

AMRIT is what's known as a **microservices platform**.

> 💡 **Jargon decoded — "microservices":** Instead of building one giant program that does everything, you build many *small, independent programs* (services), each responsible for one job — one for patient identity, one for inventory, one for telemedicine, and so on. They talk to each other to get work done. The upside is that teams can work on parts independently. The downside is that you now have a *distributed system* — many moving pieces that must coordinate over a network — which is much harder to keep reliable.

There are three technology "families" in AMRIT, each suited to its job:

| Family | Technology | Plain meaning | Where it runs |
|---|---|---|---|
| **Web apps** | Angular | The screens clinic and admin staff click on in a browser | Desktops in clinics & offices |
| **Backend services** | Java + Spring Boot | The "engine room" — logic, rules, and data access | On servers |
| **Mobile apps** | Kotlin (Android) | The apps health workers carry into the field | On phones/tablets |

> 💡 **Jargon decoded — "frontend" vs "backend":** The *frontend* is everything a person sees and touches (the screens). The *backend* is everything behind the scenes (the logic and the data) that the frontend talks to. An **API** is simply the "menu" of requests the backend understands — e.g. "give me patient #123's history."

Here's the whole system on one page. Notice how everything funnels toward a **shared backbone**, and how that backbone connects out to India's national health ecosystem:

```
        THE FIELD              POINTS OF CARE            REMOTE / CALL CENTRES
   ┌──────────────────┐   ┌─────────────────────┐    ┌────────────────────────┐
   │  ASHA worker      │   │  Health & Wellness  │    │  Helpline 104 (general)│
   │  (FLW mobile app) │   │  Centre — HWC       │    │  Helpline 1097 (HIV)   │
   │                   │   │  (web + mobile)     │    │  ECD (early childhood) │
   │                   │   │  Mobile Medical Unit│    │  Telemedicine (video)  │
   └────────┬──────────┘   └──────────┬──────────┘    └───────────┬────────────┘
            │                         │                           │
            └─────────────────────────┼───────────────────────────┘
                                       ▼
        ┌────────────────────────────────────────────────────────────┐
        │                  THE SHARED BACKBONE                         │
        │   Identity · Beneficiary-ID · Common services · Admin        │
        │   Appointment Scheduler · Medicine Inventory                 │
        │                                                              │
        │   ★ Everything orbits ONE patient ("beneficiary") record ★   │
        └───────────────────────────────┬──────────────────────────────┘
                                         ▼
                              ┌─────────────────────┐
                              │      FHIR-API       │  ──►  India's national
                              │ (the universal      │       health ecosystem
                              │  translator)        │       (ABDM)
                              └─────────────────────┘
```

> 💡 **Jargon decoded — "beneficiary":** AMRIT's word for a patient — the person receiving care. A **Beneficiary ID** is the unique number that ties all of one person's records together across every service.

In total there are roughly **30 repositories** (a *repository*, or "repo," is one project's folder of code): about **10 web apps**, **15 backend services**, **2 mobile apps**, and a handful for the database, deployment tooling, documentation, and website.

---

## 4. The parts, broken down

### 4.1 The web applications (what staff click on)

**What it is.** Ten browser-based applications — one for each kind of work: running a Health & Wellness Centre, managing a Mobile Medical Unit, telemedicine, admin configuration, appointment scheduling, medicine inventory, early-childhood-development call centres, and shared common components. All are built with **Angular**, a popular framework for building web apps.

**Why it matters.** This is the face of AMRIT for thousands of clinic and office staff. If the screens are slow, clunky, or in the wrong language, adoption suffers — and *adoption is everything* in public health.

**What's happening now.** Every one of the eight inspected web apps runs on **Angular version 16**, consistently. Consistency is genuinely good — it means one upgrade strategy works everywhere. They share a common component library and are translated into multiple Indian languages by hand.

**The challenge.** Angular 16 reached **"end of life" in November 2024** — meaning it no longer receives security patches or fixes from its makers. The current version is **Angular 20**, so AMRIT is about **four major versions and roughly three years behind**.

> 💡 **Jargon decoded — "end of life" (EOL):** When the people who maintain a piece of software officially stop supporting a version. Running EOL software is like driving a car the manufacturer no longer makes parts for — it works until something breaks, and then you're on your own, including for *security* holes.

Testing of these apps is light (basic automated checks only, no full "click through the app like a user would" testing), and translations are produced manually — which is slow and doesn't cover every regional language users actually need.

**The vision.** A modern, fast, accessible web experience on a *supported* framework, with **AI-assisted translation** so the apps speak every user's language — including the harder regional ones — without an army of human translators.

---

### 4.2 The backend services (the engine room)

**What it is.** About **15 independent backend services**, each built with **Java** and **Spring Boot** (the most widely-used framework for Java backends). They split into:

- **The shared backbone** — *Identity* (who the patient is), *Beneficiary-ID* (the unique number generator), *Common* (shared plumbing like SMS, email, documents), *Admin* (configuration), *Scheduler* (appointments), and *Inventory* (medicines).
- **The care-line services** — *HWC* (clinic), *TM* (telemedicine), *MMU* (mobile medical units), plus the helplines.
- **The translator** — *FHIR-API* (covered separately in [section 4.5](#45-interoperability-talking-to-the-outside-world)).

**Why it matters.** This is where all the rules, calculations, and data live. Every screen and every phone ultimately talks to these services. Their reliability *is* the platform's reliability.

**What's happening now — the good.** The backend is on a **modern, current foundation**: Spring Boot 3 on Java 17, applied *uniformly* across every service. For a platform with this much history, that's a pleasant surprise and a real credit to the team — someone did a serious modernisation here. The services are actively maintained.

**The challenge — three structural gaps:**

1. **There's no "front door."** In a well-built microservices system, all traffic enters through a single **API gateway** that handles security, routing, and rate-limiting in one place. AMRIT has none — each of the 15 services is its own separate door, and they find each other using addresses written into configuration files.

   > 💡 **Jargon decoded — "API gateway":** Think of a building with one staffed reception desk that checks everyone's ID and directs them, versus a building with 15 unguarded side-doors. AMRIT is currently the 15-side-doors building.

2. **The services assume the network never fails.** When one service calls another (or an external system like a video provider), it does so with **no time limit and no fallback**. If the thing it's calling hangs, the calling service waits *forever*, tying up resources. Enough of these and the whole platform can topple — a "cascading failure."

   > 💡 **Jargon decoded — "timeout" & "circuit breaker":** A *timeout* is simply giving up after, say, 5 seconds instead of waiting forever. A *circuit breaker* is an automatic switch that stops calling a service that's clearly broken, so the problem doesn't spread. AMRIT has neither.

3. **The clinical core has no automated tests, and some code is overgrown.** The largest, most important clinical service (HWC) and several others have **zero automated tests** — meaning every change is made without an automatic safety check that nothing broke. One central file is **~4,850 lines long** (healthy files are usually under a few hundred) and has been *copy-pasted* between two services, so fixes in one don't reach the other.

   > 💡 **Jargon decoded — "automated tests":** Small programs that automatically check the real program still works correctly after a change — like a spell-checker for behaviour. Without them, you only find out something broke when a user (or a patient) hits the problem.

There are also smaller issues: errors are sometimes reported as "success" to the caller (which hides problems from monitoring), internal error details can leak out, and a few demo passwords and secrets are left in the code.

**The vision.** The same modern Java foundation, but with the *missing connective tissue*: a single secure gateway, sensible timeouts and circuit breakers everywhere, shared (not copy-pasted) common code, and a real safety net of automated tests around the clinical logic — so the platform stays standing even when individual parts wobble.

---

### 4.3 The mobile apps (the field force)

**What it is.** Two Android apps built in **Kotlin**:
- **FLW-Mobile** — for **ASHA** community health workers serving pregnant women, mothers, and newborns at the doorstep.
- **HWC-Mobile** — for clinic staff (doctors, nurses, lab technicians, pharmacists).

These apps must work **offline** — in villages with no signal — and **sync** their data back to the server later.

> 💡 **Jargon decoded — "offline-first" & "sync":** The app stores everything on the phone first, so it works with no internet. Later, when a connection appears, it *synchronises* — sends the phone's data up to the server and pulls fresh data down. This is essential for rural India, and it's also one of the hardest things to get right in software.

**Why it matters.** This is AMRIT at the very edge — the actual point where care meets a person. It's also where the most sensitive data (a patient's health details) sits on a device that could be lost or stolen.

**What's happening now — a tale of two apps.** This is the most striking finding in the audit. The two apps are worlds apart in quality:

| | **FLW-Mobile (ASHA app)** | **HWC-Mobile (clinic app)** |
|---|---|---|
| Local data encrypted? | ✅ Yes (strong, per-device key) | ❌ **No — stored in plain form** |
| Login secrets stored safely? | ✅ Encrypted | ❌ Plain text |
| Hidden passwords in code? | ✅ Hidden in a secure module | ❌ Hard-coded `"Piramal12Piramal"` |
| Reliable retry if sync fails? | ✅ Robust, survives phone restrictions | ⚠️ Weak |
| Automated tests | ✅ ~200 tests | ❌ Almost none |
| **Captures patient consent?** | ✅ **Yes** | ❌ **No** |

**The challenge.** HWC-Mobile is the soft underbelly: patient health data is stored **unencrypted** on the device, a schema change can **silently wipe** all local data, and login credentials sit in plain text. On a lost or compromised field device, that's a real privacy exposure. Separately, *both* apps handle data-sync conflicts simplistically ("last write wins"), which can occasionally overwrite a worker's unsaved field edits.

**The vision.** Both apps held to the high bar FLW already sets: encrypted everywhere, bulletproof sync that never loses a health worker's hard-won field data, and consistent quality — so the experience and the safety don't depend on *which* app you happen to be using.

---

### 4.4 The data layer (the single source of truth)

**What it is.** A central **MySQL** database (a standard, reliable type of database) holding all the records, with its structure managed by a tool called **Flyway**.

> 💡 **Jargon decoded — "database migrations" & "Flyway":** As software grows, the *shape* of its data changes (new fields, new tables). A *migration* is a versioned, repeatable instruction for making that change safely. **Flyway** is the tool that applies them in order, so every environment — a developer's laptop, the test server, the live system — ends up with an identical, correct structure. Using it is a sign of discipline; many projects don't.

**Why it matters.** Everything else is just a way of reading from and writing to this. The integrity of this data *is* the integrity of patients' medical histories.

**What's happening now.** This is one of the **healthier** parts of AMRIT. There are **102 well-organised, versioned migrations** across four logical areas, and the destructive operations are properly guarded. It reflects genuine care.

**The challenge.** Two things to watch: changes are **forward-only** (there's no built-in "undo" if a migration goes wrong — recovery would mean restoring from a backup), and — more importantly — there is **no automated backup system** (see [section 4.6](#46-deployment--operations-how-it-actually-runs)). For a single shared database holding the health records of entire states, that's the highest-stakes gap in the whole platform.

**The vision.** The same disciplined approach, plus automated, regularly-tested backups and a clear recovery plan — so that the one irreplaceable asset (the patient data) is genuinely safe.

---

### 4.5 Interoperability (talking to the outside world)

**What it is.** A dedicated service, **FHIR-API**, that lets AMRIT exchange health records with *other* systems using a global standard called **FHIR**.

> 💡 **Jargon decoded — "FHIR" & "interoperability":** *Interoperability* means different health systems being able to understand each other's data. **FHIR** (pronounced "fire") is the worldwide common language for health data — like a universal plug shape. A system that "speaks FHIR" can connect to hospitals, labs, and national programs without custom one-off integrations for each.

This is the gateway to **ABDM** — India's national digital health mission, which is building a country-wide network of health records.

> 💡 **Jargon decoded — "ABDM":** *Ayushman Bharat Digital Mission* — India's government program to give every citizen a digital health ID and connect health providers nationally. Being ABDM-compatible means AMRIT can plug into the national ecosystem rather than being an island.

**Why it matters.** This is AMRIT's **strategic bridge** to the rest of the country's health infrastructure — and, because it's standards-based and clean, it's the most natural place to plug in modern capabilities (including AI) in the future.

**What's happening now.** AMRIT *has* a dedicated FHIR service using the standard, well-regarded toolkit (HAPI FHIR) and the modern FHIR "R4" version. Having this at all puts AMRIT ahead of many systems.

**The challenge.** The FHIR toolkit version is a couple of releases behind the latest, and the wildcard-open access setting on these patient-data endpoints (noted in [section 5](#5-the-cross-cutting-concerns-the-things-that-touch-everything)) needs tightening.

**The vision.** A FHIR-native core that makes AMRIT a first-class citizen of India's national health network — and a clean foundation on which standards-based AI and analytics can be built safely.

---

### 4.6 Deployment & operations (how it actually runs)

**What it is.** The machinery for taking the code and *running it* on real servers — packaging it, starting it, keeping it alive, and watching it.

**Why it matters.** Brilliant code that can't be deployed reliably, scaled, backed up, or monitored will still fail people in the real world. This is where "works on my laptop" meets "serves a whole state."

**What's happening now.** Deployment uses **Docker Compose** — a tool for running a set of services together — driven by setup scripts. The database structure is well-managed (Flyway, above), and services are set to restart automatically if they crash.

> 💡 **Jargon decoded — "Docker / containers" & "Docker Compose":** A *container* is a sealed box holding a program plus everything it needs to run, so it behaves identically everywhere. *Docker Compose* is a way to run a handful of containers together on **one machine**. It's great for development and small setups — but it runs everything on a single computer.

**The challenge — this is the platform's weakest area.** Several gaps compound:

- **Everything runs on a single server.** All ~14 backend services *and* all the databases live on one machine. If that machine fails, *everything* stops.
- **No automatic deployment.** Getting new code live is a **manual, hands-on process**, not a smooth automated pipeline. This is slow and error-prone.

  > 💡 **Jargon decoded — "CI/CD":** *Continuous Integration / Continuous Deployment* — an automated assembly line that tests new code and rolls it out to servers safely, at the press of a button (or automatically). AMRIT has the "test/build" half but not the "deploy" half.

- **No automated backups.** Backups rely on someone *manually* running a command. For patient data, this is the single scariest gap.
- **No real "is it healthy?" checks**, no limits on how much memory each service can grab (one misbehaving service can starve the others), and **a few real passwords left in the deployment files.**
- **No way to scale out.** There's one copy of each service. To serve whole states reliably, you need many copies, load-balanced and self-healing — which requires orchestration tooling that isn't in place.

  > 💡 **Jargon decoded — "orchestration / Kubernetes":** When you outgrow one machine, you need a "conductor" that runs many copies of your services across many machines, restarts failed ones, and balances the load automatically. **Kubernetes** is the industry-standard conductor. AMRIT doesn't use one yet.

**The vision.** A modern, automated, cloud-native operation: code that ships itself safely through an automated pipeline, multiple self-healing copies of each service spread across machines, managed databases with automatic backups and tested recovery, and the ability to grow smoothly as more states come online.

---

## 5. The cross-cutting concerns (the things that touch everything)

Some qualities aren't a single "part" — they run through the whole system. Three matter most.

### Security & access control

**What's happening & the challenge.** Logging in works, but **what you're allowed to do once logged in is barely controlled.** In four of the five backend services examined, *any* logged-in user can call *any* function — there's authentication ("who are you?") but almost no **authorization** ("what are you allowed to do?").

> 💡 **Jargon decoded — "authentication" vs "authorization":** *Authentication* is proving who you are (logging in). *Authorization* is what you're permitted to do afterwards (a nurse shouldn't have an administrator's powers). AMRIT mostly does the first, rarely the second.

On the positive side, the system is well-protected against one classic attack ("SQL injection"), and the login mechanism itself is reasonable. But the patient-data FHIR endpoints are set to accept requests from *anywhere* (a wildcard-open setting), demo passwords ship inside the code, and security is hand-built per-service rather than handled consistently in one place.

### Testing & quality discipline

**What's happening & the challenge.** Quality is **uneven**. Some parts (the Common backbone service, the FLW mobile app) have hundreds of genuine automated tests. The clinical core and several other services have **none**. The automated style-checks that exist are set to "advisory" — they warn but never actually block bad code from getting in. So a 4,850-line file and copy-pasted code pass through unimpeded.

### Observability (can we see what's happening?)

**What's happening & the challenge.** Today, operating AMRIT means **reading logs** — and even that is set up for a deployment style different from the one actually used. There are **no live dashboards, no metrics, no alerts, and no request tracing.** In a 15-service system, that's like flying a plane with the cockpit windows painted over: when something goes wrong, there's little visibility into *where* or *why*.

> 💡 **Jargon decoded — "observability":** The ability to see inside a running system — graphs of how fast things are, automatic alerts when something breaks, and "tracing" that follows a single request as it hops between services. Essential once you have many moving parts.

---

## 6. The scorecard: where AMRIT stands

A summary judgement across every area. Scores are out of 10, where ~8+ is "modern and solid," ~5 is "works but dated/risky," and ~3 is "a real liability."

```
Backend code foundation     ███████░░░  7/10   modern & consistent (Spring Boot 3 / Java 17)
Interoperability (FHIR)      ███████░░░  7/10   dedicated FHIR service, ABDM-ready
Data layer (database)        ███████░░░  7/10   disciplined migrations; backups missing
Mobile — FLW (ASHA app)      ████████░░  8/10   genuinely excellent engineering
Mobile — HWC (clinic app)    ████░░░░░░  4/10   unencrypted, weak, untested
Web apps (Angular)           ███░░░░░░░  3/10   on an end-of-life version (Angular 16)
Platform architecture        ███░░░░░░░  3/10   no gateway, no timeouts, no safety nets
Security & access control    ████░░░░░░  4/10   login yes, permissions no
Automated testing            ███░░░░░░░  3/10   clinical core has none; no quality gate
Resilience (failure-proofing)███░░░░░░░  3/10   assumes nothing ever fails
Deployment & operations      ███░░░░░░░  3/10   single server, manual, no backups
Observability (monitoring)   ███░░░░░░░  3/10   logs only; no dashboards/alerts
────────────────────────────────────────────────────────────────
OVERALL PLATFORM MATURITY    ████░░░░░░  ~4.5/10
```

> **The one-line summary:** *A modern engine, fitted into a vehicle with no transmission and no airbags — but with one model car in the garage that proves the team can build the whole thing properly.*

---

## 7. The five risks that matter most

Out of everything, these are the issues that could genuinely *hurt people or the mission* at scale — not cosmetic concerns:

1. **🔴 The whole platform can topple from one slow component.** With no timeouts or circuit breakers, a single stuck dependency can freeze the system. Rare at one-state scale; far more likely across five.

2. **🔴 Everything runs on one server, deployed by hand, with no automatic backups.** A single hardware failure could mean catastrophic, unrecoverable loss of patient data. This is the highest-stakes risk in the audit.

3. **🔴 Logged-in users can do almost anything.** With little authorization, the system can't enforce that staff only access what their role permits — a serious concern for sensitive health data.

4. **🔴 The clinical core has no automated tests.** Every change to the most important medical logic is made without a safety net, and it's duplicated across services so fixes can silently diverge.

5. **🔴 The clinic mobile app exposes patient data.** Unencrypted on-device storage, plain-text credentials, and a hard-coded password mean a lost or stolen device is a privacy incident waiting to happen.

---

## 8. The bright spot: a north star already exists

It would be easy to read sections 4–7 as bleak. They aren't — because of one decisive fact:

**The FLW (ASHA) mobile app is genuinely excellent, and it was built by this same team, in this same ecosystem.**

It encrypts patient data with a secure per-device key. It hides its secrets properly. It has a robust sync system that retries intelligently and survives the aggressive battery-saving features of cheap Android phones. It has ~200 automated tests. **And it already captures patient consent the right way** — a clear, mandatory, translated agreement before any health data is collected.

This changes the entire framing of AMRIT's situation:

> The problem is **not** that the team *can't* build robust, modern, secure software.
> The problem is that this quality bar is applied **inconsistently** across the ~30 projects.

That is a *far* more hopeful — and more solvable — problem. The path forward isn't "rewrite everything from scratch." It's **"raise everything to the bar FLW already set, then go beyond it."** The reference implementation of "good" already lives inside the codebase.

---

## 9. Three lenses: robustness, scalability, open-sourceability

Looking at the whole platform through the three qualities that matter most for AMRIT's future:

### 🛡️ Robustness — *will it stay up and keep data safe?*
**Today: weak.** The platform skips the fundamentals of reliability — timeouts, automated tests on the core, backups, real permissions, monitoring. **But** FLW proves the team can do this well. Robustness is the most *urgent* lens, and the most *achievable*, because the know-how is already in-house.

### 📈 Scalability — *can it grow to serve whole states?*
**Today: blocked.** As built, the platform physically cannot grow gracefully beyond a single server. Scaling to entire states needs orchestration, multiple self-healing service copies, managed databases, and proper monitoring — none of which exist yet. This is the biggest *architectural* gap, and it's directly in tension with the stated ambition of state-wide rollout.

### 🌍 Open-sourceability — *can others freely reuse and build on it?*
**Today: strong in principle, limited in practice.** The whole platform is already open-source and a verified Digital Public Good — a real asset and a core part of PSMRI's identity. But *true* reusability (other organisations adopting pieces of it) is held back by copy-pasted code, oversized files, and hard-coded configuration. The more modular, documented, and self-contained each piece becomes, the more AMRIT fulfils its Digital-Public-Good promise — building blocks the whole ecosystem can adopt, not just AMRIT.

---

## 10. The vision: what AMRIT could become

Pulling the threads together — *without yet prescribing how* — here is the shape of a modernised AMRIT:

- **A platform that expects failure and shrugs it off** — timeouts, circuit breakers, multiple service copies, automatic recovery, and tested backups, so that no single slow service or dead server takes down care delivery.
- **A platform that scales with the mission** — cloud-native, orchestrated, able to grow smoothly from two states to twenty without re-architecting.
- **A platform you can see into** — live dashboards, alerts, and tracing, so problems are caught before users feel them.
- **A consistently safe and modern experience** — every web app on a supported framework, every mobile app encrypted and tested to FLW's standard, permissions properly enforced everywhere.
- **A FHIR-native bridge to the nation** — fully plugged into ABDM, with standards-based data clean enough to power analytics and AI.
- **An AI-enhanced platform** — once the foundation is solid, AI becomes a force-multiplier: assisting low-literacy users in their own language, translating the apps automatically, supporting clinical decisions, triaging helpline calls, and predicting medicine stock-outs before they happen.
- **A true Digital Public Good** — modular, documented, reusable building blocks that *any* health organisation in the ecosystem can adopt, extending AMRIT's impact far beyond PSMRI's own deployments.

The most important takeaway is the most encouraging one: **none of this requires starting over.** The foundation is modern, the data discipline is real, the interoperability bridge exists, and — in FLW — there's already living proof that this team builds excellent software. The work ahead is to make the *whole* platform as good as its best part, and then to build the future on top of it.

---

## Glossary (jargon decoded)

| Term | Plain-language meaning |
|---|---|
| **ABDM** | India's national digital health mission — a country-wide network of digital health IDs and records. Being compatible lets AMRIT connect to it. |
| **API** | The "menu" of requests a backend service understands (e.g. "fetch patient #123"). How programs talk to each other. |
| **API gateway** | A single guarded "front door" for all incoming traffic, handling security and routing in one place. AMRIT lacks one. |
| **ASHA** | A community health worker in India — often the first and only health contact in a village. |
| **Authentication / Authorization** | *Authentication* = proving who you are (login). *Authorization* = what you're allowed to do afterwards. |
| **Backend / Frontend** | *Backend* = the behind-the-scenes logic and data. *Frontend* = the screens people see and touch. |
| **Beneficiary** | AMRIT's word for a patient. The **Beneficiary ID** is the unique number linking all of one person's records. |
| **CI/CD** | An automated assembly line that tests and ships code safely. AMRIT has the testing half, not the shipping half. |
| **Circuit breaker** | An automatic switch that stops calling a broken service so the failure doesn't spread. |
| **Container / Docker** | A sealed box holding a program and everything it needs, so it runs identically everywhere. |
| **Docker Compose** | A tool for running several containers together on **one** machine. Good for small setups. |
| **End of life (EOL)** | When software's makers stop supporting a version — no more fixes, including security ones. |
| **FHIR** | The worldwide common language for health data — a "universal plug" for health systems to exchange records. |
| **Flyway / migrations** | A disciplined, versioned way of evolving a database's structure safely. AMRIT uses it well. |
| **Interoperability** | Different health systems being able to understand each other's data. |
| **JWT** | A digital "wristband" proving a user is logged in, passed along with each request. |
| **Kubernetes / orchestration** | The industry-standard "conductor" that runs and balances many service copies across many machines. AMRIT doesn't use one yet. |
| **Microservices** | Building a system as many small independent programs instead of one giant one. Flexible, but harder to keep reliable. |
| **Observability** | The ability to see inside a running system — dashboards, alerts, and request tracing. |
| **Offline-first / sync** | The app works with no internet by storing data on the phone, then *synchronises* with the server later. |
| **Repository ("repo")** | One project's folder of code. AMRIT spans ~30 of them. |
| **Spring Boot** | The most widely-used framework for building Java backend services. AMRIT's engine room. |
| **Timeout** | Giving up on a request after a set time instead of waiting forever. AMRIT's services don't set these. |

---

*This document is a point-in-time understanding and audit, intended as a shared reference for everyone working on AMRIT's future. It describes the system as it stands; the plan for evolving it is a separate, later exercise.*
