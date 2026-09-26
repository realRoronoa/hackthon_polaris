NightShift

NightShift is a robot on-call engineer. When a live app breaks, it investigates like a detective, proves its theory in a safe test copy before touching anything real, and calls a human for permission before doing anything risky. If the evidence is weak, it says "I don't know" instead of guessing.

Built for the TrueFoundry × Polaris "Agents That Act" Hackathon.

Note on how this was built: we were not given any AI credits for this hackathon. As a result, we were not able to actually run the agents, the model calls, or a live sandbox. Everything here is the complete design plus a scripted, interactive walkthrough (nightshift-mvp.html) that shows exactly how the real system would behave, without needing model access to demonstrate it.

1. The problem, in one sentence

Agents that can take real actions are powerful but risky — if one guesses wrong and rolls back the wrong thing, that's worse than doing nothing. NightShift is built to know the difference between "I'm confident" and "I'm guessing," and to behave differently in each case.

2. What NightShift actually does, step by step
Something breaks. A monitor watches a demo online store and notices when checkouts start failing.
NightShift investigates like a detective, not a psychic. It reads real logs, real error rates, the real code that was just deployed, and the deploy history. It never just asks a model "what do you think happened" and trusts the first answer.
It proves the theory, it doesn't just suspect it. It takes the suspect code, runs it in an isolated sandbox, and checks whether it actually fails. Then it checks the previous, working version too, to be sure that one is fine.
It scores its own confidence. Every piece of evidence adds or subtracts fixed points on a written-down scale — this is arithmetic anyone can double-check, not a feeling the model reports.
If it's confident enough, it asks permission — every time, for anything risky. It can look things up freely, but it can never roll back a deploy, change a config, or flip a production switch without a human saying yes first. In the demo, that's a real phone call.
If it's not confident enough, it says "I don't know." If the evidence is mixed, it stops, says so, and lists exactly what additional evidence would help. It never flips a coin.
After fixing something, it checks its own work. It watches the app for 30 seconds and only calls the incident resolved if error rates, response times, and logs all genuinely look healthy again.
It writes up what happened. A short incident report, plus a code fix proposal so the same bug can't happen twice.
3. The system, end to end
Judge-facing:   Dashboard  ·  Demo store  ·  Judge's phone
                       │ events / approvals
Control plane:  Control API  ·  Audit store (SQLite)
                       │ tool calls
Harness:        NightShift agent  ·  Chaos agent  ·  Sandbox
                       │
Production:     Demo app · Payment mock · Traffic gen · Monitor · Deployer
                       │
External:       GitHub  ·  AI Gateway  ·  Voice API

NightShift and Chaos are separate agents with separate credentials. NightShift can never see the attack menu or the fault-injection tool; Chaos can never see rollback or config tools. That's enforced by what each agent is allowed to connect to — not just a polite instruction in a prompt.

4. What happens during one incident
no
yes
HIGH
approved
rejected
healthy
not healthy
Monitor detectsa threshold breach
NightShift readslogs, metrics, diffs
Reproduce suspectversion in sandbox
Score confidenceper candidate cause
Top ≥ 85%?reproduced?gap 15pts?
Escalate:'I don't know'
Action risk
Call a humanfor approval
Execute the fix
Watch metrics 30s
Postmortem + PR

Every read (logs, metrics, deploy history) runs on its own — no approval needed. Every write that touches production always pauses for a human, no exceptions, enforced by the system itself, not just the prompt.

5. Confidence is math, not vibes
Evidence	Points
Problem started right after a change	+20
The change touches the failing part of the app	+20
Reproduced the failure on the suspect version	+35
Previous version is clean when tested the same way	+25
A third-party service's own status page shows problems	+40
Suspect version actually passes its tests	−35

NightShift only acts if one theory scores 85%+, was actually reproduced, and beats the next-best guess by more than 15 points. Otherwise it stops and explains what's missing.

6. The adversary: Chaos

A second agent, Chaos, breaks the demo app one of four ways, picked live by a judge. NightShift is never told which one.

Chaos does this	Real cause	Correct response
Pushes and deploys broken code	The app itself	Roll back to the last good version
Deploys a bad config change	A setting, not the code	Undo the config
Forces a payment provider to fail	Something outside our control	Don't touch our code — flip a "retry later" switch
A harmless deploy and an outage together	Genuinely ambiguous	Escalate — say "I don't know"

That last row is the point of the whole project. Most demos are built to look impressive; we built in a case where the right answer is to do nothing and be honest about why.

7. Approval: a real phone call
Dashboard
Judge's phone
Voice API
NightShift
Dashboard
Judge's phone
Voice API
NightShift
alt
[no answer in 2 min]
Request approval: rollback(a1b2c3d)
Calls, reads out the request
Press 1 (approve) or 2 (reject)
Falls back to Slack, then dashboard card
Decision recorded, run resumes

If nobody responds anywhere within 2 minutes, that counts as a rejection — NightShift backs off rather than acting alone.

8. Try it yourself: nightshift-mvp.html

No server, no setup — just open it in any browser.

Pick one of the four attacks (this simulates the judge's/Chaos's choice).
Click "Run the incident" (auto-play) or "Next step" (one action at a time — better for presenting live).
Answer the simulated phone call and press 1 or 2 yourself at the approval step.
Watch the recovery checks, the reveal (what Chaos actually did vs. what NightShift concluded), the postmortem, and the proposed code fix.
Click "Run another round" and try a different attack — including "Double trouble," to see it correctly say "I don't know."

A scoreboard at the top tracks rounds and accuracy — escalating correctly on the ambiguous case counts as a correct call, not a miss.

9. How to present this to judges (the full script)

Before you start: open the page full-screen. Decide your first attack — start with "Bad deploy", it's the clearest story. Save "Double trouble" for last, it's the twist ending.

The 90-second pitch, before touching anything:

"NightShift is an AI agent that responds to production incidents. What makes it different: it never acts on a guess. It gathers evidence, proves its theory in a sandbox, scores its own confidence with fixed math, and calls a human before doing anything that touches production. If the evidence is weak, it says 'I don't know' instead of guessing. Watch."

Walking through one round:

Point at the attack cards: "These four are the four things our adversary agent, Chaos, can do. I'll pick one — NightShift has no idea which." Click one.
(Optional, technical judges) Point at the topology diagram: "NightShift and Chaos are separate agents with separate credentials — NightShift literally can't see the attack menu."
Click "Next step" a few times, reading the evidence feed out loud. Every line is a real tool call and result — this is what proves it isn't a black box.
At the confidence bars: "This number isn't a vibe — it's addition and subtraction against a fixed table. It only acts above 85%, with a clear lead over the next guess."
When the phone-call card appears, click "Answer the call," read the transcript out loud, press 1. Say: "This is the point where most 'autonomous' agents would just act. Ours can't — this is enforced by the system, not a polite prompt."
Watch the recovery checklist tick through: "It doesn't call this done because it thinks it's done — it's rechecking real metrics for 30 seconds first."
At the reveal card: "Here's the answer key — what Chaos actually did, versus what NightShift concluded." Read the postmortem and proposed fix if there's time.

The twist round — "Double trouble": click "Run another round," pick Double trouble, let it play out. When it escalates: "This is the part we're proudest of. It didn't roll anything back. It said 'I don't know,' showed its work, and told us what evidence would resolve it. An agent that knows when to stop is more trustworthy than one that always has an answer." Point at the scoreboard — the escalate counts as correct, not a failure.

If something goes wrong:

"Is this connected to a real app?" — Be honest: "This page is a scripted walkthrough of the real design. The confidence math and the four scenarios are exactly what the full system does. We didn't have AI credits for this event, so the live version with a real store, real monitoring, and a real phone call is the next build phase."
Lose your place → click "Run another round", takes 10 seconds.
Wifi drops → doesn't matter, the page needs nothing but the browser it's already open in.

Quick answers to likely questions:

Why not let it act without approval? A wrong autonomous rollback is worse than a 30-second delay for a human "yes."
How is the confidence score not the AI making things up? Fixed point values applied to real evidence — anyone can re-check the arithmetic by hand.
What if nobody answers the phone? Falls to Slack, then an always-on dashboard card; no answer within 2 minutes counts as "no."
What stops NightShift from seeing what Chaos did? Separate agents, separate credentials, separate tools — an actual permissions boundary, not a prompt instruction.
10. What's real vs. simulated
Piece	Status
Full design — architecture, scoring, risk rules, attack catalog	Real, complete
Interactive walkthrough (nightshift-mvp.html)	Real, working, scripted
Live demo app, monitor, deployer	Not built — blocked without AI credits/compute
Real NightShift/Chaos agents on TrueForge	Not built — same reason
Real phone-call approvals via a voice API	Not built — same reason

Because we had no AI credits for this event, we couldn't run any actual model calls, so the live end-to-end system doesn't exist yet — only its complete design and a faithful, hand-scripted walkthrough of exactly how it would behave. Every number, threshold, and outcome shown in the walkthrough matches the real design; nothing in it is decorative.

11. Where this goes next
Memory — reuse lessons from past incidents to resolve repeats faster.
Earned trust — NightShift starts by asking permission for everything risky, and could earn autonomy for specific, well-proven fixes over time.
Real integrations — Kubernetes, Datadog/Prometheus, PagerDuty, instead of our demo stand-ins.