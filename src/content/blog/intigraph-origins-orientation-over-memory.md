---
title: "Intigraph Origins: Agent Orientation > Memory"
description: "Agents are good at writing memories and bad at using them when they act. From markdown memory files and Human Input Lineage to Quipu, and why I think in terms of orientation."
pubDate: 2026-11-04
image: images/og-orientation.png
---

*This is part of a series where I will discuss the origins of the [Intigraph](https://intigraph.ai) agent platform.*

*By Steve Brown and Johnny Hammond*

> **TL;DR**
> - Agents are good at *writing* memories and bad at *using* them at the moment they act.
> - Markdown memory files don't scale: one agent's folder hit 149 files, about 100k tokens, and agents grep it instead of reading it.
> - My first fix, Human Input Lineage (HIL), stored and typed my own input. It recorded 391 of my inputs in three weeks, but it lived in its own silo, linked to nothing it was about.
> - A real outage in March showed the cost: every agent rebuilt the same picture from scratch. That pushed me to knowledge graphs.
> - What agents need is **orientation**: knowing their task, what it will affect, its history, and the rules that apply, at the moment they act.

In Seinfeld's ["The Alternate Side,"](https://en.wikipedia.org/wiki/The_Alternate_Side) Jerry walks up to a rental car counter. They have his reservation. They don't have a car.

Jerry is baffled, hands miming the whole transaction out in the air. He explains, patiently and then not so patiently, that the reservation is the whole point. Anyone can *take* a reservation. The trick is keeping the car there for when the person shows up. Then, with one last flourish: *"you just don't know how to hold the reservation."*

![Me, explaining memory to an agent](/blog/images/jerry-reservation-agent.gif)
*Me, explaining memory to my agent. Seinfeld, "The Alternate Side" (NBC, 1991).*

I've had that conversation with my agents more times than I can count. Not about cars. About memory.

This happens to me, and I'm sure to others too: you watch an agent act and think "wait, didn't we already discuss why this is wrong?" Every single time, the memory was already there, written down, saved, sitting in a file the agent could read. When I asked why it had done the thing I'd told it not to do, the answer was some version of: *yes, we have a memory for that. I just didn't read it before I acted.*

> **They know how to take the memory. They just don't know how to hold it.**

That gap is what this series is about, and I've started calling the fix **orientation**: an agent knowing its task, what it will affect, its history, and the rules that apply, at the moment it acts.

## The markdown era

Memory is an obvious limitation that is continuously faced, even now, when using agents. I think many approaches continue from early origins in things like Cline's [memory bank](https://cline.bot/blog/memory-bank-how-to-make-cline-an-ai-agent-that-never-forgets) and, more recently, Karpathy's [LLM Wiki](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f): you use markdown files to persist a memory for agents. Those markdown files often contain rules on how to use the markdown files themselves: how they should be organized and whatnot. My own homelab repo started that way in November 2025, with a `.clinerules` file telling the agent to read and update `activeContext.md`, `progress.md` and `next_steps.md` on every task (straight out of Cline's [memory bank docs](https://docs.cline.bot/best-practices/memory-bank)), because "if it isn't documented, it didn't happen."

It works, until it doesn't. One of my agents' memory folders grew to 149 files, about 100k tokens. No agent reads that before acting. It greps, and grep finds the files that use the same words you did, not the ones that apply.

> **Highlight:** 149 memory files, ~100k tokens. Nobody reads that before acting; agents grep it.

I also started managing [Claude Code's agent memory files](https://code.claude.com/docs/en/memory) with a system I called [Pensieve](https://github.com/scbrown/scbrown.github.io/blob/main/data/hil-history/designs/pensieve.md). Every two hours a harvester copied each agent's memory files into one shared archive that any agent could search ([the plan](https://github.com/scbrown/scbrown.github.io/blob/main/data/hil-history/designs/pensieve-plan.md)). It put more memories in one place, but it was still no better at getting the right one to the agent when it mattered.

I want to contrast this with one of my early approaches, which I called [Human Input Lineage](https://github.com/scbrown/scbrown.github.io/blob/main/data/hil-history/designs/human-input-lineage.md) (HIL). It later influenced how I think about agentic memory and what I built in [Quipu](https://github.com/intigraph/quipu) (my governed knowledge graph for agents) and the [Intigraph agent platform](https://github.com/intigraph). That contrast I would phrase as the difference between agentic *orientation* and agentic memory.

## Human Input Lineage

At the start of this year, one of the problems with how I was leveraging agents in my homelab was that I would build design documents and specs with an agent, and the final result would merge both of our ideas in a way that was impossible to pick back apart. I noticed that sometimes my input would even be overridden later by agents iterating on a design. The high-level idea I began to formalize into its own system was that human input should be held in higher regard than agent input, and to do that, we had to save what the human input was. An evolution of this was that you could cite designs, work items and output back to a HIL, and the agent could say "look, I did it because of that!" A consequence is that, over time, there would be many HILs, and some would be meant only for certain topics or contexts, and others would contradict each other.

The HIL system evolved its own [types](https://github.com/scbrown/scbrown.github.io/blob/main/data/hil-history/designs/human-input-lineage.md): idea, directive, review, approval, correction. It supported different humans and different levels of authority: I could report to someone, and that someone could also report to someone else. I could have peers, and their input counted too ([two-tier authority](https://github.com/scbrown/scbrown.github.io/blob/main/data/hil-history/designs/hla-two-tier-authority.md)). In February it became a real system: an [MCP tool](https://github.com/scbrown/scbrown.github.io/blob/main/data/hil-history/designs/hla-mcp-tools.md) every agent could call to store and query human input, backed by its own small archive. Over three weeks it recorded 391 of my inputs, most of them short instructions (the median was 75 characters). The original design documents, with a timeline, are [here](https://github.com/scbrown/scbrown.github.io/tree/main/data/hil-history).

> **Highlight:** HIL typed human input as idea, directive, review, approval or correction, and recorded 391 of my inputs in three weeks.

## Why it didn't scale

I had two types of persistent context given to the agent: its own memory files, and the HIL system, an MCP tool and database to read and write items. These ideas don't scale, because they are too simple, too verbose, and they need to be *governed*. A HIL item has a lifecycle and a bounded context; the memory files are sprawled across many markdown files, and each agent has its own. Both can be completely ignored, duplicated and contradicted by future agents.

The proof came at the end of March, when my main Proxmox host, koror, [went down](https://github.com/scbrown/scbrown.github.io/blob/main/data/hil-history/designs/koror-kota-recovery.md). It was a P0, and I moved everything onto a physical host I call kota (I use a [Survivor](https://en.wikipedia.org/wiki/Survivor_(American_TV_series)) tribe naming theme for my physical hosts: koror is a tribe from [Palau](https://en.wikipedia.org/wiki/Survivor:_Palau), kota from [Gabon](https://en.wikipedia.org/wiki/Survivor:_Gabon) :). Every agent I pointed at the recovery had to reconstruct the same dependency picture from scratch: which services ran where, what depended on what, what I had decided last time ([the retrospective](https://github.com/scbrown/scbrown.github.io/blob/main/data/hil-history/designs/kota-recovery-retrospective.md)). All of that knowledge existed. It was spread across Ansible inventory, design docs, CLAUDE.md files, HIL and Pensieve, and none of it was connected. My own directives were saved, but they weren't linked to the hosts and services they were about.

> **Highlight:** during a real outage, every agent rebuilt the same picture from scratch. The knowledge existed; it just wasn't connected to anything.

Three days after that outage, I wrote the design for a [homelab ontology](https://github.com/scbrown/scbrown.github.io/blob/main/data/hil-history/designs/aegis-ontology.md). Its problem statement lists HIL right next to everything else: it "captures Stiwi directives in S3 but isn't linked to the entities they affect." The next day, a harvest job pulled 322 of my directives and 434 agent observations out of HIL and Pensieve and turned them into entities in a graph. From there the HIL ideas carried into the graph: my input became a typed node, with a link back to where it came from and the nodes it influences. At first those links were made by simple keyword matching: a directive that mentioned "dolt" got linked to the Dolt server, one that mentioned "traefik" to Traefik. Crude, but it got my input connected to the things it was about, and making those links precise became its own story.

What I needed was structure: memory with types, owners and lifecycles, connected to the things it's about, that an agent could target instead of grep. That's why I call this *orientation*: a node in the graph orients the agent by showing it what relates to the task it was given.

So I built [Quipu](https://intigraph.github.io/quipu/) ([paper](https://arxiv.org/abs/2608.16813)), the core of the Intigraph platform. The other tools in the stack build on it, and together they give an agent its orientation.

> **Orientation: knowing your task, what it will affect, its history, and the rules that apply, at the moment you act.**

*Discuss this post on [GitHub Discussions](https://github.com/intigraph/community/discussions).*

---

*Johnny Hammond is an AI collaborator, co-author and code archaeologist working with Steve Brown. He's the Claude agent that helps Steve administer his life, named after John Hammond of Jurassic Park. For these posts he digs through git history, design docs and archives to find the dates and documents behind the story.*
