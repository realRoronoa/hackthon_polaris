# NightShift

**NightShift is a robot on-call engineer.**

When something breaks in a live app, most teams get a page at 3am and a human has to figure out what went wrong, decide what to do, and fix it — half asleep, under pressure. NightShift is an AI agent that does that job instead, but it's built to be careful: it doesn't just guess and click "fix it." It gathers real evidence, proves its theory in a safe test copy of the app first, and only then acts — and even then, it calls a human to ask permission before touching anything important.

Built for the TrueFoundry × Polaris "Agents That Act" Hackathon.

---

## The problem, in one sentence

AI agents that can take real actions are powerful, but also scary — if an agent guesses wrong and deletes a database or rolls back the wrong thing, that's worse than doing nothing. NightShift is our attempt at an agent that knows the difference between "I'm confident" and "I'm guessing," and behaves differently in each case.

## What NightShift actually does, step by step

1. **Something breaks.** A small monitoring script is watching a demo online store. When checkouts start failing, it notices.
2. **NightShift investigates — like a detective, not a psychic.** It reads real logs, real error rates, the real code that was just deployed, and the deploy history. It doesn't ask an AI model "what do you think happened" and trust the first answer.
3. **NightShift proves it, it doesn't just suspect it.** Before blaming anything, it takes the suspect version of the code, runs it in an isolated sandbox, and checks: does it actually fail? Then it checks the previous, working version too, to make sure that one is fine. This is the difference between "I think it's this" and "I checked, and it's this."
4. **It scores its own confidence.** Every piece of evidence adds or subtracts points on a fixed, written-down scale (this isn't the AI feeling confident — it's simple arithmetic anyone can double check). Only if the top suspect scores 85% or higher, was actually reproduced, and clearly beats the next-best guess does NightShift consider acting.
5. **If it's confident enough, it asks permission — every single time, for risky actions.** NightShift can look things up freely, but it can never roll back a deploy, change a config file, or flip a production switch without a human saying yes first. In our demo, that "asking permission" step is a real phone call.
6. **If it's not confident enough, it says "I don't know."** This is the part most demos skip. If the evidence is mixed — two equally likely causes, or a test that isn't clearly failing — NightShift stops, says so out loud, and tells you exactly what additional evidence would help. It does not flip a coin.
7. **After fixing something, it checks its own work.** It watches the app for 30 seconds afterward and only calls the incident "resolved" if error rates, response times, and logs all actually look healthy again. If they don't, it says the incident is still open.
8. **It writes up what happened.** A short plain-English incident report, plus a code change proposal (a pull request) so the same bug can't happen again.

## Why there's a second "evil" agent

To prove NightShift isn't just following a script, we built a second agent called **Chaos**. Chaos's whole job is to break the demo app in one of four realistic ways — a bad deploy, a bad config change, an outage in a payment provider NightShift doesn't control, or a deliberately confusing situation with two possible causes at once. A judge picks which one. **NightShift is never told which one was picked.** It has to figure it out from evidence alone, live, in front of everyone.

## The four ways things can break, and the right response to each

| What Chaos does | What actually broke | What NightShift should do |
|---|---|---|
| Pushes broken code and deploys it | The app itself | Roll back to the last good version |
| Changes a config file and deploys it | A setting, not the code | Undo the config change |
| Makes a third-party payment service fail | Something outside our control | Don't touch our own code — turn on a "retry later" safety switch instead |
| Both a harmless deploy *and* an outage happen at once, on purpose | Ambiguous — could be either | Admit it doesn't know, and say what evidence is missing |

That last row is the important one. Most demos are built to show off an agent doing something impressive. We deliberately built in a case where the right answer is for the agent to **do nothing and be honest about why.**

## How confident is "confident enough"?

We didn't want "the AI felt sure" to be good enough to justify a production change. So confidence is computed, not vibes:

- Did the problem start right after a change? **+20 points**
- Does the change actually touch the part of the app that's failing? **+20 points**
- Did we reproduce the failure on the suspect version, in a real test? **+35 points**
- Is the previous version clean when we test it the same way? **+25 points**
- Is a third-party service reporting problems on its own status page? **+40 points** (for the "not our fault" theory)
- Does the suspect version actually pass its tests? **−35 points** (evidence against blaming it)

NightShift only acts if one theory reaches 85%+, was actually tested, and clearly beats the next best guess by more than 15 points. Otherwise, it escalates.

## Why a human still has to say yes

Every action that changes production — rolling back a deploy, undoing a config, flipping a feature flag — pauses the whole process and waits for a real person. In the live demo, that's a real phone call: NightShift explains in plain language what it wants to do and why, and the person on the phone presses 1 to approve or 2 to reject. If nobody answers, it falls back to a chat message, then a button on a dashboard that's always available. If nobody responds within 2 minutes, it treats that as a "no" and backs off rather than acting alone.

This rule is enforced by the system itself, not just by the AI being told to be careful — even if the AI "decided" to skip the approval step, the underlying system wouldn't let the action through without a recorded yes from a human.

## What you can actually try right now

We built a single, self-contained web page — `nightshift-mvp.html` — that walks through this entire process interactively, without needing any of the real servers running. Open it in any browser:

1. Pick one of the four attacks (this simulates what the judge/Chaos would choose).
2. Click **"Run the incident"** to watch it play out automatically, or **"Next step"** to go one action at a time (better if you're presenting live and want to talk through each step).
3. When it reaches the approval step, answer the simulated phone call and press 1 or 2 yourself.
4. Watch the recovery checks, the reveal of what actually happened vs. what NightShift concluded, the generated incident report, and the code fix it proposes.
5. Click **"Run another round"** and try a different attack — including the "double trouble" one, to see NightShift correctly say "I don't know."

There's a running scoreboard at the top tracking how many rounds you've played and how many were handled correctly — including escalating being counted as the *correct* answer when the evidence really is ambiguous.

See `DEMO_GUIDE.md` for a short script if you're presenting this to judges.

## What's real vs. what's simulated in this MVP

Being upfront about this:

- **The reasoning, the confidence scoring, the risk rules, and the four attack scenarios are the real design** — this is exactly what the full system is built to do (see the HLD document for the complete technical design).
- **The MVP page itself is a scripted walkthrough**, not a live system with a real app, real logs, or a real phone line — it's built so anyone can see and understand the whole idea in a browser in two minutes, without spinning up servers.
- **The full build** (real demo store, real monitoring, a real TrueForge-based agent, a real phone call through a voice API) is the next phase, laid out in the HLD's build plan.

## Where things stand

| Piece | Status |
|---|---|
| Overall design (HLD) | Done |
| Interactive walkthrough (this MVP) | Done |
| Real demo app + monitoring + deployer | Next |
| Real NightShift agent on TrueForge | Next |
| Real Chaos agent + attack menu | Next |
| Real phone-call approvals | Next |

## What's next, if we keep building this after the hackathon

- **Memory** — reuse what it learned from past incidents so it's faster the second time the same bug happens.
- **Earned trust** — NightShift starts out asking permission for everything risky, and could earn the right to act on its own for specific, well-proven kinds of fixes over time.
- **Hooking into real tools** — Kubernetes, Datadog, PagerDuty, instead of our demo stand-ins.

---

*For the full technical design — architecture diagrams, data model, tool list, and the exact build schedule — see the HLD document.*

---

**AI credit not given**