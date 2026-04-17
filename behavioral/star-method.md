# STAR Method

## What Is STAR

A structured framework for answering behavioral questions:

| Letter | Stands for | What to cover |
|--------|-----------|--------------|
| **S** | Situation | Context — where, when, what was happening |
| **T** | Task | Your responsibility — what were you supposed to accomplish |
| **A** | Action | What YOU specifically did — concrete steps, decisions |
| **R** | Result | Outcome — quantified if possible, what you learned |

---

## Why Interviewers Use It

Behavioral questions are based on the premise: **past behavior predicts future behavior**. They want to understand:
- How you actually handled real situations (not hypotheticals)
- How you think and make decisions
- What you value and how you work with others

---

## STAR for Senior Engineers

At senior level, the bar is higher. Interviewers expect:

| Junior | Senior |
|--------|--------|
| Solved a bug | Identified the root cause pattern, prevented class of bugs |
| Wrote a feature | Designed the architecture, unblocked the team |
| Followed the process | Improved the process |
| Asked for help | Mentored others and escalated when needed |
| Completed tasks | Owned outcomes, not just tasks |

---

## Crafting Strong Actions

The **Action** is the most important part. Be specific:

**Weak:** "I worked with the team to fix the problem."
**Strong:** "I proposed three options to the team with estimated tradeoffs, ran a proof of concept for option 2 over the weekend, presented the results Monday, and we agreed to proceed. I personally wrote the DB migration scripts and coordinated the deployment window with the infrastructure team."

### Action verbs for senior engineers
- **Designed**, **architected**, **proposed**, **established** — technical leadership
- **Influenced**, **persuaded**, **aligned**, **facilitated** — soft skills
- **Mentored**, **coached**, **enabled**, **unblocked** — people leadership
- **Measured**, **tracked**, **reduced**, **improved** — results orientation
- **Escalated**, **negotiated**, **pushed back** — courage

---

## Results — Make Them Concrete

Bad results:
- "It went well."
- "The team was happy."
- "We shipped it."

Good results:
- "Deploy time reduced from 45 min to 8 min, removing a blocker for 3 teams."
- "Incident MTTR dropped from 2 hours to 15 minutes over the next quarter."
- "Feature shipped 2 weeks ahead of schedule, directly contributed to closing a €200k deal."
- "Three junior engineers I mentored are now independently leading features."

If you don't have numbers:
- "Based on user feedback, the error rate dropped significantly after the fix."
- "The process change was adopted by 4 other teams without prompting."

---

## Common Mistakes

### 1. Using "We" instead of "I"
Interviewers are asking about **you**, not the team. Say "I" even if it was collaborative — just acknowledge the team.

Bad: "We designed the system."
Good: "I drove the initial design, proposed the key architectural decisions, and got alignment from the team."

### 2. Too vague
"I did some research and we figured it out." → No signal for the interviewer.

### 3. Skipping the Result
Many candidates spend all their time on Situation and Action, then run out of time for Result. The Result is where value is demonstrated.

### 4. Only positive stories
Interviewers expect stories about **failure**, **conflict**, and **mistakes**. Refusing to share them signals low self-awareness.

### 5. Over-explaining the Situation
The Situation should be 10–15% of your answer. Most candidates spend 50% there.

---

## Preparation Strategy

### Story Bank — Prepare 10–15 Stories
Cover these themes:
- **Achievement:** something you're proud of
- **Leadership without authority:** drove something without formal power
- **Failure/mistake:** owned it, learned from it
- **Conflict:** disagreed with a colleague or manager, resolved it
- **Technical decision:** trade-off you analyzed and decided on
- **Mentoring:** helped someone grow
- **Ambiguity:** navigated unclear requirements
- **Prioritization:** too much to do, chose wisely
- **Going above and beyond:** did more than required
- **Feedback received:** changed your behavior based on criticism

### Story Template
For each story, write out:
```
SITUATION: [2 sentences max]
TASK: [1 sentence — your specific responsibility]
ACTION: [3–5 concrete bullet points of what YOU did]
RESULT: [Quantified outcome + what you learned]

TAGS: [which questions this story answers]
```

### Reuse Stories Flexibly
One strong story can answer multiple questions:
- "Tell me about a time you led without authority" → also answers leadership, influence, ownership
- "Describe a failure" → also answers resilience, growth, accountability

---

## Answer Length

**Target: 2–3 minutes.**

Practice with a timer. Interviewers can tell you more if they want detail — but they can't give you back time you wasted with a 10-minute story.

---

## Example: Full STAR Answer

**Question:** "Tell me about a time you had to make a technical decision under pressure."

**Answer:**

> **Situation:** We were three days before a major product launch when our load tests revealed that our notification service would fail at 10x expected load — we were expecting 50,000 concurrent users but could only handle 5,000.
>
> **Task:** I was the tech lead for the notification service. I needed to either fix the performance issue or propose a plan that kept the launch on track.
>
> **Action:** I immediately gathered the team for a 30-minute war room. We profiled the service and found the bottleneck: we were making a synchronous DB call for each notification, and the DB was the bottleneck. I outlined two options: (1) add a Redis cache in front of the DB — low risk, 2 days of work; (2) move to async processing via a queue — better long-term, but risky to introduce 3 days before launch. I recommended option 1 for the launch, with option 2 planned for the following sprint. The engineering manager agreed. I personally implemented the caching layer, wrote the load test, and coordinated with QA for regression testing.
>
> **Result:** After the fix, we tested up to 80,000 concurrent users — 60% headroom above the target. The launch went smoothly. Post-launch, I wrote a post-mortem and we added load testing as a mandatory gate in our CI pipeline. That policy has since caught two similar issues in other services before they reached production.
