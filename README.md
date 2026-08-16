# Game Development Engines 2024

**Comparing Godot, Unity, and Unreal Engine — from a year of building one 3D game.**

Kenneth W. · Senior capstone (CSCI 495), University of Tennessee at Martin · Presented at ACM

---

## What this is

Between September 2023 and November 2024, two of us built **DATASPIRE** — a 3D first-person
action platformer — in the Godot game engine, and used that build to answer a question:
*how effective is Godot for 3D game development, compared to Unity and Unreal Engine?*

This write-up is the honest version of that comparison.

**A note on evidence, because it determines what this document can claim.** We used Godot every
day for a year. We did **not** build anything comparable in Unity or Unreal — we read their
documentation. The original abstract said so plainly, calling itself a *"surface-level
comparison"* limited by *"our relative inexperience in Unity and Unreal Engine compared to
Godot."* That was accurate then and it is accurate now.

So every claim below is tagged:

- 🔨 **Built** — first-hand, from a year of development. Dated in the project log.
- 📄 **Documented** — looked up and cited, not built. Vendor documentation where it was
  reachable; where it wasn't, contemporaneous industry reporting, flagged as secondary in the
  source list.

The tagging is the point. An engine comparison written by people who have only shipped in one
engine is worth reading *if it is clear about which parts are experience and which parts are
reading*. Blurring the two is what makes most engine comparisons useless.

One more convention: **sections 1–3 are the 2024 project**, and **section 5 is deliberately priced
at 2024** rather than today, because the licensing terms of that window are what actually drove
the decision. Section 4 mixes both and says so.

---

## 1. General architecture

All three engines solve the same problem — *how do you compose a game object out of reusable
parts?* — and all three arrived at a similar answer with different vocabulary.

| | **Godot** | **Unity** | **Unreal Engine** |
|---|---|---|---|
| Base object | Node | GameObject | `UObject` → Actor |
| Behaviour attaches via | Child nodes + scripts | Components | Components |
| Container / level | Scene | Scene | Level (inside a World) |
| Reusable prefabricated unit | Scene (instanced) | Prefab | Blueprint Class |
| Primary language | GDScript (also C#, C++) | C# (`MonoBehaviour`) | C++ and Blueprints |

### Godot — nodes and scenes 🔨

Godot's docs call nodes **"the fundamental building blocks of your game,"** and **"together,
nodes form a tree."** A node does one job — draw a mesh, hold a collision shape, drive a camera,
navigate a mesh. You build behaviour by composing nodes into a tree and attaching scripts.

A group of nodes saved together becomes a **Scene**, and scenes nest: **"scenes work like new
node types in the editor, where you can add them as a child of an existing node."** A scene
always has one root node, and you can instance it as many times as you like.

In DATASPIRE this meant the Virus enemy was a scene whose root node carried the script defining
its behaviour, with child nodes supplying its mesh, collision, and navigation agent — and that
whole scene could be dropped into a level as a single unit.

The mental model we used while presenting it: *if the Internet is a series of tubes, Godot is a
stack of Legos.* Everything is the same kind of piece, and the pieces nest.

**What this is genuinely good at:** there is exactly one concept to learn. A level, a menu, a
HUD, an enemy, and a weapon are all the same kind of thing — a tree of nodes. Nothing is a
special case. For a two-person team with no engine experience, that uniformity was the single
biggest reason we stayed productive.

**What it costs:** deep node trees get unwieldy, and because a script lives on a node rather than
in a separate behaviour layer, "where does this logic belong" is answered by tree position. We
ended up with three global singletons (`PlayerSettings`, `GlobalScript`, `ZetaScene`) precisely
because some state doesn't belong to any node in particular.

### Unity — GameObjects and Components 📄

Unity separates the container from the behaviour more sharply than Godot does. The manual is
explicit that a GameObject is only a container: **"a GameObject can't do anything on its own; you
need to give it properties before it can become a character, an environment, or a special
effect."** GameObjects **"act as containers for Components, which implement the functionality."**

So where Godot specialises by *adding a child node*, Unity specialises by *adding a component to
the same object*. Every GameObject carries a mandatory Transform component that cannot be
removed. Custom behaviour comes from C# scripts deriving from `MonoBehaviour`, attached as
components.

### Unreal Engine — UObjects, Actors, and Components 📄

Unreal has the deepest hierarchy of the three. `UObject` is the shared base class for most
classes in the engine, providing garbage collection, reflection metadata, and serialisation.

**Actor** subclasses `UObject` and is the thing you place in a level. Epic's own documentation
draws the cross-engine comparison for us: Actor is described as **"a close equivalent to Unity's
GameObject."** Actors contain one or more Components, can be spawned at runtime, and support
network replication.

Unreal further splits components in a way neither other engine does: an **Actor Component** has
no transform and is for abstract behaviour (inventory, attributes), while a **Scene Component**
adds a transform for anything that needs a position in the world. Actors live in **Levels**;
Levels live in a **World**.

Uniquely, Unreal offers two authoring surfaces over the same class system — C++ and **Blueprints**,
a node-based visual scripting system — and you can create a C++ class and then derive a Blueprint
class from it.

### The honest summary

These are three spellings of the same idea. If you understand node/scene, you can read
GameObject/Component and Actor/Component without much friction — Epic says as much itself.

Where they genuinely differ is **how much structure the engine imposes before you write any
code.** Godot imposes almost none: one concept, compose freely. Unreal imposes the most: a
gameplay framework of specific classes with defined roles, plus a second authoring language.
Unity sits between them.

For a student team, Godot's low structure was an advantage — we could build something without
first learning a framework. For a large team on a deadline, the same property is a liability:
Unreal's imposed structure *is* the shared vocabulary that lets many people work on one codebase.
We were not in a position to test that second claim, and we don't.

---

## 2. The stair-stepping gap: Godot vs Unity

**Stair-stepping** is letting a player walk up a step without jumping. It sounds trivial. It is
the single clearest engine-feature gap we hit all year.

**Unity** 📄 ships it. The built-in character controller exposes a `stepOffset` property — you set
a maximum step height and the controller handles the rest.

**Godot** 🔨 does not. There is no native stair-stepping, and there was no standardised solution
to copy. The community proposal for it —
[godot-proposals#2751](https://github.com/godotengine/godot-proposals/issues/2751), *"Add
automatic smooth stairs step-up and step-down for CharacterBody using `move_and_slide()`"* — was
opened on **2021-05-20** and, checked again on **2026-08-16**, is **still open**. That is over
five years without a settled engine-level implementation.

Worth noting what the proposal actually asks for: an optional **`step_height`** parameter on
`move_and_slide()`. It is a request for Unity's `stepOffset`, more or less exactly. The gap is
well understood by Godot's own community; it just hasn't been closed.

So we wrote our own, and then rewrote it. The project log records the rework:

> **2024-02-26** — Movement system rework #1 complete. Notably, the stair-stepping functionality
> was updated.

**Why this one matters more than it looks.** Every custom stair-stepping implementation in Godot
is subtly different, because there is no reference implementation. That means it interacts badly
with everything else touching the character controller — slopes, crouching, sliding, and in our
case dynamic gravity, since we let the player reorient which way is down. A feature Unity users
configure with one number was, for us, a recurring source of movement bugs across two reworks.

**What this does not mean.** It does not mean Unity's character controller is better overall — we
never built a character controller in Unity, so we can't say. It means one specific, verifiable,
commonly-needed feature is built into one engine and absent from the other. That is a narrow
claim, and it is the kind of claim this project can actually support.

---

## 3. Level loading: Godot vs Unreal

Levels have to be loaded before the player can enter them. Both engines can do it. They get there
very differently.

**Godot** 🔨 has no built-in level manager. You write it, and it's about three steps:

1. Load the new scene (the map)
2. Destroy the current map, if one exists
3. Instantiate the loaded scene and add it as a child of the scene tree

That is genuinely easy — it's a handful of lines, and it's obvious what it does, because there is
no machinery between you and the scene tree. It also took us more than one attempt: our first
level-loading script worked only for the first load, and produced a bug where the player spawned
under the map.

**Unreal Engine** 📄 ships this as a subsystem. Its **Level Streaming** system handles loading
and unloading, exposed both in the editor and in code. A `LevelManager` class drives it, and the
`GameInstance` class holds a reference to the manager and exposes public methods like
`LoadLevel()` and `ReloadCurrentLevel()`. Level Streaming also supports dynamically loading and
unloading maps to save memory.

### A correction to our own conference slide

Our ACM deck concluded this comparison with *"different architecture than Godot, but similarly
easy."* **On review, our own evidence doesn't support that.** The same slide listed three things
Unreal gives you that Godot doesn't:

- reload-current-level is provided up front; in Godot you write it yourself
- dynamic streaming for memory savings has no Godot equivalent in our implementation
- you don't have to manage unloading at all

Three advantages, then a verdict of parity. The accurate version is: **Godot's approach is simpler
to understand and Unreal's is more capable.** For DATASPIRE — a handful of hand-built levels
loaded one at a time — simpler was the right trade and we'd make it again. For a large streaming
world, it would not be, and our three steps would have to grow into something much closer to what
Unreal already ships.

Correcting this is the reason this section exists. The original conclusion was the summary we
*wanted*; the bullet points above it were what we actually found.

---

## 4. System requirements

📄 Vendor documentation, retrieved **2026-08-16**. None of this is first-hand measurement.

**Versions.** DATASPIRE shipped on **Godot 4.3** (we upgraded to it on 2024-08-19), so Godot is
pinned to 4.3 here. **Unity 6** released 2024-10-17 and is the version of that era. For Unreal,
Epic's version-specific pages for the 2024 releases would not load, so the figures below are from
the **currently published** specification — treat the Unreal column as approximately, not exactly,
the 2024 position.

| | **Godot 4.3** | **Unity 6** | **Unreal Engine** (current published) |
|---|---|---|---|
| **OS (min)** | Win 7 / macOS 10.13 (Compat) or 10.15 (Forward+) / Linux (2016+) | Win 10 21H1 (19043)+ x64; Win 11 21H2 (22000)+ Arm64 | Win 10 22H2 64-bit |
| **OS (rec)** | Win 10 / macOS 10.15 / Linux (2020+) | *not published* | Win 11 |
| **CPU (min)** | x86_32 + SSE2, x86_64, or ARMv8 | x64 with SSE2, or Arm64 | *not published* |
| **CPU (rec)** | x86_64 + SSE4.2, 4+ cores, or ARMv8 | *not published* | Quad-core Intel/AMD, 2.5 GHz+ |
| **RAM (min)** | **4 GB** | — | *not published* |
| **RAM (rec)** | **8 GB** | **8 GB** | **32 GB** |
| **GPU (min)** | Integrated, Vulkan 1.0 or OpenGL 3.3 | DX10/11/12 or Vulkan capable | *not published* |
| **GPU (rec)** | Dedicated, Vulkan 1.2 or OpenGL 4.6 | *not published* | DX12-compatible, **8 GB+ VRAM** |
| **Storage (min)** | **200 MB** (editor, project, cache) | *not published* | *not published* |
| **Storage (rec)** | **1.5 GB** (incl. all export templates) | High-IOPS drive advised for builds | *not published* |

### What the table actually shows

**The spread in recommended RAM is 4×.** Godot recommends 8 GB; Unreal recommends 32 GB and 8 GB
of dedicated VRAM on top. That is not a small difference — it is the difference between a
mid-range student laptop and a purpose-built workstation. For a two-person capstone with no
hardware budget, this is arguably a bigger practical constraint than any feature gap in section 2.

**The absences are as informative as the figures.** Epic publishes a recommended configuration
but **no minimum CPU, RAM, GPU, or storage requirement at all** — only a minimum OS and DirectX
runtime. Unity publishes RAM and GPU but **no storage figure**, offering general advice about
drive IOPS instead. Godot is the only one of the three that publishes a complete minimum *and*
recommended matrix, including exact disk figures.

**This independently corroborates something we claimed from experience.** Our conference talk
listed *"good documentation"* as a Godot strength and *"lack of online tutorials"* as a weakness —
two separate claims that are easy to collapse into one. The specs table supports the first one
from outside our own experience: the engine with by far the smallest organisation behind it
publishes the most precise hardware requirements of the three.

### Correcting our own install-size numbers

Our ACM deck listed install sizes as **Godot 4: ~190 MB · Unity 5: ~9.5 GB · Unreal Engine 5:
~10–15 GB**. Three problems with that slide, which we're fixing rather than repeating:

1. **The figures were uncited** — they were our own rough observations, not vendor figures.
2. **The versions were mismatched.** It compared Godot **4** against Unity **5** against Unreal
   **5**, while the previous slide in the same deck correctly noted that Unity 6 exists.
3. **The Godot figure was misleadingly favourable.** ~190 MB is roughly right for the bare editor,
   and Godot's docs now give 200 MB — but that excludes export templates. You cannot ship a game
   without them, and with them the official recommended figure is **1.5 GB**. The honest
   comparison is 1.5 GB, not 190 MB.

Godot is still dramatically the smallest of the three by any measure. Neither Unity nor Epic
publishes an official installed-size figure, so the multi-gigabyte numbers for those two remain
**unverified estimates** and are presented here as such.

---

## 5. Licensing and the real cost of choosing an engine

📄 **These are 2024 figures**, deliberately. This project ran from September 2023 to November 2024,
and the licensing terms of that window are the ones that actually shaped the decision. Pricing
them at today's rates would describe an environment we never worked in. Terms have moved since;
anything here should be read as *the position as of 2024*, not as current pricing.

Vendor pages for Unity and Epic block automated retrieval, so most of this section comes from
contemporaneous industry reporting rather than vendor documentation. Sources are listed at the end.

### Where the three engines stood in 2024

| | **Godot** | **Unity** | **Unreal Engine** |
|---|---|---|---|
| Source | Open (MIT) | Proprietary | Source-available |
| Cost to start | Free | Free under threshold | Free |
| Free tier | Unlimited | Personal: revenue/funding **< $200,000/yr** | Free for games; students, educators, hobbyists |
| Paid seat | — | **Pro $2,200/seat/yr** (raised from $2,040) · Enterprise **+25%** | **$1,850/seat/yr** — non-game commercial over $1M/yr only |
| Royalty | **None** | None | **5%** of lifetime gross revenue **above $1M** per product |
| Obligation | Attribution only | Subscription at threshold | Royalty reporting |

**Godot** — MIT-licensed: free and open source, no royalties. You may release your project under
any license, you are the sole copyright owner of what you make, and the only obligation is
reproducing Godot's copyright notice; a link in your credits is considered acceptable.

**Unity** — seat-based. Personal free below **$200,000** annual revenue or funding, a threshold
**doubled from $100,000** as part of the September 2024 changes. Pro rose from **$2,040 to $2,200
per seat per year** (about **+7.8%**) and Enterprise rose **25%**. Unity's mandatory splash screen
became optional from Unity 6.

**Unreal** — free for game development, with a **5% royalty on lifetime gross revenue above $1
million** per product; the first $1M is exempt and Epic Games Store sales are excluded. Separately,
in **late April 2024** alongside UE 5.4, Epic introduced a **$1,850 per seat per year**
subscription for **non-game** commercial users earning over **$1M annually** — film, television,
product configurators, and similar. Game developers were unaffected, as were users staying on 5.3
or earlier.

### The event that actually decided it 🔨

We did not choose Godot on any of the numbers above. We chose it because of a licensing event.

We had **almost decided on Unity** in September 2023. On **2023-09-12**, Unity announced the
**Runtime Fee** — a charge levied *per installation*:

- **Personal:** up to **$0.20 per install**, once a game passed **$200,000 revenue and 200,000
  installs**
- **Pro / Enterprise:** **$0.125–$0.15 per install**, once a game passed **$1M annual revenue and
  1M lifetime installs**

Unity did not adequately define what counted as an "install," and the developer community reacted
badly. We switched to Godot, and DATASPIRE was built in Godot as a direct result.

**Unity cancelled the Runtime Fee in September 2024** — roughly a year later, and before we
presented — reverting to seat-based subscriptions and raising subscription prices instead.

**That is the finding in this section, and it isn't about pricing.** By the time we stood up at
ACM, the policy that drove our engine choice no longer existed.

### Which engine costs more? It depends, and the answer flips

The most useful thing we found is that **there is no single answer** to "which engine is cheaper."
Modelled across five developer scenarios in a September 2023 industry analysis:

| Scenario | Unreal | Unity |
|---|---|---|
| Small team, $1.5M **PC** revenue | $25,000 | **$24,480** |
| Small team, $1.5M **mobile** revenue | **$25,000** | $61,980 |
| Medium team, $15M revenue | $700,000 | **$291,600** |
| Large studio, $500M revenue | ~$24,950,000 | **~$2,961,996** |

Unity is cheaper in three of four — and dramatically so at scale, because a 5% royalty on $500M
is an enormous number while seats are bounded by headcount. But **on mobile at small scale the
ranking inverts** and Unity costs roughly 2.5× Unreal, which is why that analysis concluded small
mobile developers were hit hardest by the fee structure despite being the demographic most
associated with Unity.

⚠️ These are **one analyst's modelled scenarios, not vendor figures**, published during the
Runtime Fee period and reflecting fee-era Unity pricing that was later cancelled. Treat the
*pattern* as the finding and the specific dollar amounts as illustrative.

The pattern is worth stating on its own: **the two engines don't just differ in price, they
differ in the shape of the cost.** Unity's cost is up-front and per-seat, paid out of the
development budget before you know whether the game sells. Unreal's is contingent and per-project,
paid only on revenue you have already earned — Epic only makes money when you make money. An
up-front cost is a fixed risk to a project with uncertain commercial prospects; a royalty is not.
Which is better depends entirely on how confident you are, and on a student capstone with no
revenue at all, both were $0 and neither mattered.

### What generalises past game engines

- **Licensing risk is not the same as licensing cost.** Every price above is knowable in advance.
  What actually moved us was the *possibility of an unfavourable future change* by a vendor who
  could make one unilaterally. Godot's MIT license is valuable less because it costs $0 than
  because there is no party who can change the terms later.
- **Trust, once spent, outlasts the policy.** The fee was withdrawn; the migration it caused was
  not reversed. Adoption data from the same period is consistent with that: entries in the GMTK
  Game Jam made with Godot went from **1,278 of 6,835 (19%) in 2023 to 2,838 of 7,711 (37%) in
  2024**, while Unity's share fell from **59% to 43%**.

  Treat that as one signal, not proof. A single game jam's entrants are hobbyists and students,
  not the commercial market, and the jam is not a random sample of anything. It is the most
  specific adoption figure we gathered, with both numerator and denominator and both years stated
  — which is exactly why its limits are worth stating too.

---

## What this write-up does not claim

- **It is not a three-engine comparison of equal depth.** It is a Godot experience report with two
  focused comparisons — one against Unity, one against Unreal — plus a documented comparison of
  architecture, specs, and licensing. Only the Godot material is first-hand.
- **We measured nothing about Unity or Unreal.** No build times, no line counts, no performance
  benchmarks. We never shipped anything in either engine.
- **The best technical work we did never became a comparison.** Godot's navigation mesh generation
  had no out-of-the-box support for vertical surfaces, so walls and ceilings had to be authored in
  Blender, imported, and stitched to standard navmeshes with navigation links — and Godot's
  pathfinding, not built with vertical navigation in mind, then behaved unpredictably on them. We
  solved it (project log, **2024-08-07** to **2024-08-08**) but never wrote up how Unity or Unreal
  handle the same problem. It is the largest gap between what we learned and what we presented.
- **Our verdict was audience-specific and stays that way.** We recommended Godot *to developers
  getting into game development*. We had no basis for a recommendation to a studio, and still
  don't.

---

## Verifying these figures

**Section 5 is dated to 2024 on purpose** and is not current pricing. Anyone using it to make a
decision today should re-check every figure.

Vendor pages for Unity and Epic block automated retrieval. Godot's documentation and the Godot
proposal were fetched directly and are quoted exactly; Unity's and Epic's own documentation pages
that did load are cited below; the remaining licensing figures come from **contemporaneous
industry reporting**, which is secondary and is labelled as such in the source list.

One source below (the Unreal forum thread) spans 2017–2023 and contains **superseded** figures —
notably an older Unreal royalty model with a $3,000-per-quarter threshold, replaced by the $1M
lifetime threshold. It is cited for the structural argument about up-front versus contingent cost,
not for any dollar amount.

## Sources

### Primary — vendor documentation (retrieved 2026-08-16)

- [Godot — System requirements (4.3)](https://docs.godotengine.org/en/4.3/about/system_requirements.html)
- [Godot — Nodes and Scenes](https://docs.godotengine.org/en/stable/getting_started/step_by_step/nodes_and_scenes.html)
- [Godot — License](https://godotengine.org/license/)
- [Godot proposal #2751 — stair-stepping](https://github.com/godotengine/godot-proposals/issues/2751) — opened 2021-05-20, still open
- [Unity — System requirements for Unity 6](https://docs.unity3d.com/6000.0/Documentation/Manual/system-requirements.html)
- [Unity — GameObjects](https://docs.unity3d.com/6000.0/Documentation/Manual/GameObjects.html)
- [Unreal Engine — Hardware and software specifications](https://dev.epicgames.com/documentation/unreal-engine/hardware-and-software-specifications-for-unreal-engine)
- [Unreal Engine — Gameplay framework](https://dev.epicgames.com/documentation/en-us/unreal-engine/gameplay-framework-in-unreal-engine)

### Secondary — industry reporting on 2024 licensing

- [Unity cancels Runtime Fee: what this means for developers](https://rocketbrush.com/blog/unity-cancels-runtime-fee-what-this-means-for-developers) — Runtime Fee structure, cancellation, and replacement pricing
- [Unreal Engine 2024 subscription pricing announced](https://gamefromscratch.com/unreal-engine-2024-subscription-pricing-announced/) — the $1,850/seat non-game subscription, UE 5.4, April 2024
- [Unreal vs Unity: which costs more?](https://gamefromscratch.com/unreal-vs-unity-which-costs-more/) — modelled cost scenarios, 2023-09-23
- [Unreal Engine vs Unity3D cost](https://forums.unrealengine.com/t/unreal-engine-vs-unity3d-cost/103044) — structural argument only; contains superseded figures

## Repository contents

| Path | What it is |
|---|---|
| `abstract/` | Conference abstract |
| `project-proposal/` | Original project proposal (LaTeX + PDF) |
| `progress-report/` | Dated development log, Sep 2023 – Aug 2024 (LaTeX + PDF) |
| `presentations/acm/` | ACM conference deck, versions 1–5 |
| `presentations/class/` | Course update decks |
| `presentations/individual/` | Individual presentation on Godot |
| `media/` | Demo capture — user interface |

The class update decks are reproduced as delivered, including sections that were never filled in.

The remaining gameplay demo recordings — game map, mechanics, and entity behaviour — are not
included. Most exceeded GitHub's 100 MB per-file limit and were not hosted elsewhere.
