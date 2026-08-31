# Module 18 — Human-in-the-Loop Systems

### Difficulty
Advanced

### Learning Objectives
- Understand when agents should require human approval before acting.
- Understand approval gates, risk levels, escalation, and review systems.
- Understand how a real approval-queue implementation should handle Approve, Reject, and Edit differently.

### Prerequisites
Modules 1–17.

---

## Lesson 18.1 — Why Human-in-the-Loop (HITL) Matters

### Concept Explanation

Every mitigation in Module 17 — retries, timeouts, guardrails, validation, fallback models — is about making an agent's *own* execution more reliable. Human-in-the-loop design starts from a different, more humble premise: for some actions, no amount of engineering makes fully autonomous execution the right call, because the cost of a rare-but-possible mistake is simply too high to accept, no matter how low you've driven the error rate. This isn't a failure of the agent's design — it's a recognition that "how reliable does this need to be" and "how reliable can I make it" are two separate questions, and for the highest-stakes actions, the honest answer to the first question is "more reliable than any autonomous system can currently guarantee."

The core idea is straightforward: **human-in-the-loop (HITL)** design inserts explicit checkpoints where the agent pauses mid-execution and waits for a human to approve, reject, or edit a proposed action before that action actually takes effect. Notice the precise placement of this pause: it happens *after* the agent has done all its reasoning, tool use, and decision-making — the human isn't replacing the agent's judgment, they're reviewing the agent's *conclusion* before it becomes an irreversible real-world action. This is a crucial distinction from having a human do the entire task manually; the agent still does 100% of the actual work (research, drafting, deciding), and the human's involvement is narrowly scoped to a single yes/no/edit decision at the one moment it matters most.

### A Common Question

**"If an action needs human approval anyway, haven't we just built an elaborate way for the agent to draft something a human reviews — isn't that basically no different from the human doing the task themselves with an AI assistant?"** The difference is in where the effort goes. Without the agent, a human spends time on *both* the hard part (gathering information, drafting, deciding what to say) and the easy part (actually clicking send, filing the paperwork). With a well-designed HITL agent, the agent absorbs all of the hard part — the research, the drafting, the reasoning about what action makes sense — and the human's remaining job shrinks to the fast, low-effort part: glancing at a finished, well-reasoned proposal and making one quick decision. This is a genuine and large efficiency gain, even though a human is still "in the loop" — the loop just got much shorter and much less effortful for them. This is also why the *quality* of what the agent hands the human matters enormously: a vague, unreasoned proposal forces the human to redo the hard part themselves to evaluate it properly, which erases the efficiency gain entirely. This is exactly the problem the review-interface design in Lesson 18.3 is built to avoid.

**"Doesn't this just move the risk from 'agent makes a mistake' to 'human rubber-stamps the agent's mistake without really checking'?"** Yes, and this is a real, well-documented failure mode of HITL systems in practice, not a hypothetical concern — it's serious enough that it appears explicitly in this module's Common Mistakes (approval fatigue). A human who's asked to approve twenty routine, always-fine actions in a row will, predictably, start clicking "approve" on the twenty-first without truly reading it, simply because their brain has learned that this button always leads to a fine outcome. The entire design challenge of a good HITL system is keeping approval requests rare enough, and important enough, that the human reviewing them stays genuinely engaged rather than lapsing into automatic approval — which is precisely why risk-tiering (Lesson 18.2) exists: it's not just a technical filter, it's what protects the human reviewer's attention as a scarce resource.

### Simple Analogy

> A junior employee can draft an email on their own, but a good process has their manager glance at it before it goes out to an important client. The employee (agent) still does the work — the human just gates the risky final action. Notice what the manager is *not* doing here: they're not re-researching the client's account history, re-drafting the email from scratch, or re-deriving the recommendation the junior employee reached. They're doing exactly one thing — reading the finished draft and deciding whether it's good enough to go out — which is only a fast, low-effort task *because* the junior employee already did all the hard work of getting it to a reviewable state. If the junior employee instead handed the manager a half-formed idea with no research behind it, the manager's "quick glance" would turn into doing the job themselves — exactly the failure mode described in the first Common Question above.

### Visual Diagram

```text
AI drafts email
       ↓
Human approves (or edits/rejects)
       ↓
Email is sent
```

**Reading this diagram closely:** the arrow from "Human approves" to "Email is sent" is the only path that leads to the action actually happening — notice there's no arrow directly from "AI drafts email" to "Email is sent." This is the entire mechanical point of HITL: the agent's own conclusion is never sufficient, by itself, to trigger a high-risk action; a human decision is a mandatory, un-skippable link in the chain. The parenthetical "(or edits/rejects)" hints at something this simple diagram doesn't fully draw out — there are actually three distinct outcomes a human reviewer can produce, not one, and each needs genuinely different handling in your code, which is exactly what Lesson 18.3's expanded example below covers.

---

## Lesson 18.2 — Approval Gates and Risk Levels

### Concept Explanation

Not all actions carry equal risk, and treating them as if they did produces exactly the two failure modes this module warns against: gate everything, and you've built a system that's constantly interrupting a human for trivial decisions, which both slows the system down and (per the approval-fatigue problem above) trains the human to stop paying attention. Gate nothing, and you've built a fully autonomous system that will, sooner or later, take an action you'd have desperately wanted a chance to stop first. The practical resolution is to classify actions into risk tiers *before* deciding your approval policy, so the policy can be different for genuinely different levels of consequence.

| Risk Level | Example Actions | Approval Needed? |
|---|---|---|
| **Low** | Reading data, searching, drafting (not sending) content | No — fully autonomous |
| **Medium** | Sending internal messages, updating non-critical records | Optional — log and allow undo, or lightweight notification |
| **High** | Sending external communications, spending money, deleting data, modifying production systems | Yes — mandatory human approval before execution |
| **Critical** | Irreversible financial transactions, legal commitments, actions affecting many users at once | Yes — mandatory approval + possibly multi-person sign-off |

**What actually distinguishes these tiers from each other — the underlying dimensions, not just examples:** if you look closely at the table, the tiers aren't ordered by an arbitrary feeling of "how scary does this sound" — they track two concrete, measurable properties of an action: **reversibility** (can this be undone if it turns out to be wrong, and how easily?) and **blast radius** (how much/how many people does this affect if it goes wrong?). Reading data is Low because it's perfectly reversible (reading something has no lasting effect on the world at all) and has zero blast radius. Sending an internal message is Medium because, while not exactly "undoable" once read, its blast radius is small and contained (one internal recipient, low external consequence). Sending an external communication or spending money is High because it's genuinely hard or impossible to fully undo (you can't un-send an email a client already read, you can't costlessly reverse a payment) and it affects someone outside your own organization. Critical actions add a third dimension on top of those two: scale — an action that's individually similar in risk to a "High" action but affects many users or carries legal weight multiplies the consequences of getting it wrong, which is why it warrants an even higher bar (multi-person sign-off) rather than just one reviewer.

### Visual Diagram — Risk-Based Gating

```mermaid
flowchart TD
    A([Agent decides on an action]) --> C{Classify risk level}
    C -- Low --> L[Execute directly]
    C -- Medium --> M[Execute + log]
    C -- High --> H["PAUSE → request<br/>human approval"]
    C -- Critical --> X["PAUSE → request<br/>multi-person approval"]

    style L fill:#dcfce7,stroke:#16a34a
    style M fill:#dcfce7,stroke:#16a34a
    style H fill:#fef3c7,stroke:#d97706
    style X fill:#fee2e2,stroke:#dc2626
```

**How to read this graph:** the color coding tracks risk directly — green means "just let it run," amber means "pause and wait for one person," red means "pause and wait for more than one person." Notice only two of the four branches (Low, Medium) let the agent act immediately; the other two both hit a hard "PAUSE" before anything actually executes. This is the mechanism that makes Module 18.2's risk table concrete: an agent using this flowchart never has to guess whether to ask a human — the risk classification decides it automatically. It's also worth noticing that this classification step happens *before* any execution attempt at all — the risk tier is determined purely from what the agent is *about* to do (which tool, with what input), not from anything that happens during execution. This ordering matters: if you classified risk *after* attempting an action, you'd already have taken the risky action by the time you realized it needed approval, which defeats the entire purpose.

### A Common Question

**"Who decides the risk tier for a given tool — is this something the agent figures out itself at runtime?"** No, and this is an important design principle: risk classification should be a fixed, developer-defined mapping (like the `RISK_LEVELS` dictionary in Lesson 18.3's code example), not something the LLM decides dynamically on each call. If the agent itself got to decide "is this action risky," you'd be relying on the exact same non-deterministic, occasionally-wrong reasoning process that HITL exists to add a safety net *around* — a compromised or confused agent could simply decide its own dangerous action is "low risk" and skip the gate entirely. By hardcoding the risk tier per tool (or per tool-plus-parameter-range, e.g., "refunds under $50 are Medium, refunds over $50 are High") in ordinary, deterministic application code, the safety gate doesn't depend on the agent's judgment being correct — this is the same principle behind guardrails in Module 17.3: policy enforcement should live in code the agent can't talk its way around.

---

## Lesson 18.3 — Escalation and Review Systems

### Concept Explanation

Two more pieces complete the HITL picture beyond simple risk-tier gating:

- **Escalation**: when an agent is uncertain, stuck, or facing an ambiguous/high-stakes decision — even one that wouldn't normally be gated by its risk tier alone — it should proactively hand off to a human rather than guessing. This is a distinct trigger from risk-tiering: risk-tiering gates based on *what kind of action* is being proposed, while escalation gates based on the agent's own *confidence* about a specific instance of a decision, which risk-tiering alone can't capture (a normally-Low-risk search action might still deserve escalation if the agent has searched five times and found nothing useful, because continuing to act autonomously with no useful information is itself a form of risk).
- **Review systems**: a structured way for humans to see pending approvals, the agent's reasoning/plan behind the proposed action, and approve, reject, or edit with minimal friction. The quality of this interface is not a cosmetic detail — it directly determines whether the human reviewer can actually do a meaningful review in the few seconds they're likely to spend on it (per the approval-fatigue discussion in Lesson 18.1).

### Practical Example (Conceptual)

```python
RISK_LEVELS = {"send_email": "high", "search": "low", "delete_record": "critical"}

def execute_action(action: dict, approval_queue) -> dict:
    risk = RISK_LEVELS.get(action["tool"], "medium")

    if risk in ("high", "critical"):
        approval_queue.submit(action)     # pause and wait for a human decision
        return {"status": "pending_approval", "action": action}

    return run_tool(action)  # low/medium risk: execute immediately (with logging)
```

*Explanation:* `RISK_LEVELS.get(action["tool"], "medium")` implements the fixed, developer-defined mapping discussed above — note the default of `"medium"` for any tool not explicitly listed, which is a deliberate safety choice: an unrecognized or newly-added tool defaults to a cautious middle tier rather than silently defaulting to fully autonomous ("low") execution, so a developer who forgets to explicitly classify a new tool doesn't accidentally create an ungated high-risk action. When the risk is high or critical, `approval_queue.submit(action)` hands the action off and the function returns immediately with a `"pending_approval"` status — notice this function does *not* block and wait synchronously for a human to respond; it returns right away, which connects back to Module 20.4's asynchronous task-queue pattern (a human approval can take minutes or hours, and holding a request open that whole time is exactly the anti-pattern that module warns against).

### What Happens After the Human Decides: Approve vs. Reject vs. Edit

The example above shows how an action gets *into* the approval queue, but a genuinely complete implementation needs to handle all three things a human reviewer can do with it — and each one requires meaningfully different follow-up logic, not just a single generic "resume" path:

```python
def handle_approval_decision(pending_action: dict, decision: dict) -> dict:
    outcome = decision["outcome"]  # "approved", "rejected", or "edited"

    if outcome == "approved":
        # The action runs exactly as the agent proposed it — no changes.
        result = run_tool(pending_action)
        log_decision(pending_action, outcome, reviewer=decision["reviewer_id"])
        return {"status": "executed", "result": result}

    elif outcome == "rejected":
        # The action never runs. Critically, the agent needs to find out WHY
        # it was rejected, not just THAT it was rejected, or it may propose
        # the same (or a similarly flawed) action again on the next attempt.
        log_decision(pending_action, outcome, reviewer=decision["reviewer_id"],
                     reason=decision.get("reason"))
        notify_agent(
            pending_action["task_id"],
            f"Your proposed action was rejected. Reason: "
            f"{decision.get('reason', 'No reason given.')} "
            f"Do not repeat this exact action; reconsider your approach."
        )
        return {"status": "rejected", "reason": decision.get("reason")}

    elif outcome == "edited":
        # The human changed something about the proposed action before
        # approving it (e.g., corrected the email's recipient or amount).
        # The EDITED version executes, not the agent's original proposal --
        # and the edit itself is valuable signal worth recording.
        edited_action = decision["edited_action"]
        result = run_tool(edited_action)
        log_decision(pending_action, outcome, reviewer=decision["reviewer_id"],
                     original=pending_action, edited=edited_action)
        return {"status": "executed_with_edits", "result": result}

    raise ValueError(f"Unknown decision outcome: {outcome}")
```

*Explanation:* the **Approved** branch is the simplest — the agent's proposal executes unchanged, and the decision is logged for audit purposes (this is precisely the kind of record the `approvals` table in Module 24's capstone database design is built to hold). The **Rejected** branch does something the earlier, simpler example glossed over entirely: it doesn't just stop — it calls `notify_agent(...)` to feed the rejection *and its stated reason* back into the agent's own context, so that if the agent is given another attempt at the broader task, it has a chance to actually learn from why its first proposal was turned down, rather than blindly generating the same rejected proposal again (which, absent this feedback, is a realistic outcome given the agent has no other way of knowing what was wrong with it). The **Edited** branch is the subtlest of the three: it explicitly executes the *human's edited version*, not the agent's original proposal — and it deliberately logs both versions side by side (`original=pending_action, edited=edited_action`), because that difference is genuinely useful data: a pattern of humans consistently editing the same field on the same type of action (say, always correcting the tone of drafted emails) is a strong, concrete signal that the agent's prompt or instructions for that action need improvement, in a way a simple approve/reject count alone would never surface.

```text
Human Review UI (conceptual):
┌─────────────────────────────────────────────────────────┐
│ Pending Approval #482                                    │
│ Action: send_email                                       │
│ To: client@example.com                                   │
│ Subject: "Project Update"                                │
│ Reasoning: "User requested a status update be sent after │
│ the weekly report was generated."                        │
│                                                            │
│ [ Approve ]   [ Edit ]   [ Reject ]                       │
└─────────────────────────────────────────────────────────┘
```

*Explanation:* showing the agent's concise reasoning alongside the proposed action lets the human reviewer make an informed, fast decision instead of blindly trusting or re-doing the whole analysis themselves. Notice the interface shows exactly the fields a reviewer needs to evaluate this *specific* action (recipient, subject, and the stated reasoning for why this email is being sent at all) without dumping the agent's entire multi-step research trace on the screen — this is a deliberate design choice, not an omission: too much context defeats the purpose of a fast review just as surely as too little context does, echoing the same "signal vs. noise" trade-off Module 17.1 describes for context overload, applied here to a human reader instead of the model itself.

### Key Takeaways
- HITL isn't "give up on automation" — it's targeted risk management, gating only the actions where a mistake is costly or hard to reverse, based on the concrete, measurable dimensions of reversibility and blast radius rather than a vague sense of "this feels risky."
- Risk-based tiering avoids the two failure extremes: gating everything (kills the value of automation and induces approval fatigue) or gating nothing (unsafe for high-stakes actions) — and the tier a given action belongs to should be a fixed, developer-controlled classification, never something the agent decides about itself at runtime.
- Escalation should be a first-class agent behavior, not an afterthought — a well-designed agent knows when to say "I'm not confident enough to proceed without a human," and this trigger is distinct from (and complementary to) risk-tier gating.
- Approve, Reject, and Edit are three genuinely different outcomes requiring three genuinely different code paths — a Rejected action should feed its reason back to the agent rather than silently vanishing, and an Edited action's specific difference from the original proposal is valuable signal for improving the agent, not just a one-off correction.

### Common Mistakes
- Applying the same approval gate to every action regardless of risk, causing approval fatigue — humans start rubber-stamping without really reviewing, defeating the safety purpose; this is precisely why risk-tiering exists, and skipping it in favor of "just gate everything to be safe" produces a *less* safe system in practice, not a more cautious one.
- Not showing the human reviewer enough context (just "approve this email?" with no visible content/reasoning) — makes the review meaningless, since the reviewer either has to dig for the missing context themselves (defeating the speed advantage of HITL) or approve blind (defeating the safety advantage).
- Building HITL as a manual code review afterthought rather than a designed part of the agent's control flow (Module 16's state management makes "pause and resume" for approval tractable) — an agent's execution needs to genuinely suspend and later resume from exactly where it left off once a decision comes back, which is only possible if the underlying state was checkpointed properly in the first place.
- Treating "Rejected" as equivalent to simply not executing an action, without feeding the rejection reason back to the agent — this silently discards the single most useful piece of information the human just provided, and risks the agent proposing an equally-flawed action again with no memory of having been corrected.
- Discarding the difference between a human's edit and the agent's original proposal instead of logging both — this loses a systematic, low-effort source of feedback about exactly where the agent's judgment consistently falls short, which is far cheaper to collect passively (every edit already happening in normal operation) than to gather through dedicated evaluation efforts (Module 19).

### Exercise
Classify the following actions into the four risk tiers: reading a customer's order history; issuing a $500 refund; searching product documentation; permanently deleting a user account.

### Challenge
Design an escalation policy for an autonomous research agent: under what specific conditions should it stop and ask a human for guidance rather than continuing on its own (e.g., conflicting sources, budget/tool-call limits approaching, ambiguous instructions)? For each condition, specify what information the escalation message to the human should include so they can make a fast, informed decision.

### Knowledge Check
1. Why is gating every single action a bad HITL design, not a safe one?
2. What information should a human approval review interface show, at minimum?
3. Give one condition under which an agent should proactively escalate rather than wait to be asked.
4. Why does a Rejected decision need to feed a reason back to the agent, rather than simply not executing the action?
