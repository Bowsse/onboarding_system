# Multi-Agent Employee Onboarding System

## Overview

Automate the mundane, multi-system, cross-departmental process of onboarding a new employee. Today this is done by a human copy-pasting between Workday, Active Directory, Slack Admin, Google Workspace, a badge system, and a shipping portal — checking 15 checkboxes and re-checking when step 3 fails because step 1 wasn't done.

---

## Architecture Pattern

**Hierarchical/supervisory** — one orchestrator agent delegates to specialist sub-agents treated as tools. Sub-agents have no awareness of each other; all coordination flows through the orchestrator.

**Framework:** LangGraph (Python) — stateful graphs with built-in human-in-the-loop interrupts and parallel branching.

---

## Agent Definitions

Each agent follows the **Contractual Skills framework** (arXiv:2605.22634): explicit goal, inputs, permissions, human approval gates, quality criteria, and verification steps.

### 1. Orchestrator Agent

| Property | Value |
|---|---|
| **Goal** | Onboard employee within SLA, all dependencies met, no gaps |
| **Inputs** | New-hire record (name, role, start date, department, location, manager) |
| **Permissions** | Read access to HR system, write access to onboarding tracker |
| **Human gates** | Escalate if any sub-agent fails or if start date is < 48h away |
| **Logic** | Fan-out parallel to IT + HR + Facilities agents. Wait for all. On completion, trigger buddy agent. On failure, pause and notify onboarding lead. |

### 2. IT Provisioning Agent

| Property | Value |
|---|---|
| **Goal** | All accounts created and verified 24h before start date |
| **Inputs** | Name, email, role, department, required tool list |
| **Tools** | Active Directory API, Google Workspace Admin SDK, Slack SCIM API, GitHub org API, license management system |
| **Quality criteria** | Each account verified with login test; credentials stored in password manager |
| **Human gate** | Manager must confirm tool access level before provisioning (prevents over-provisioning) |

### 3. HR Compliance Agent

| Property | Value |
|---|---|
| **Goal** | Legal and payroll requirements met before day 1 |
| **Tools** | DocuSign API (offer letter), payroll system API (W-4, direct deposit), benefits enrollment portal, LMS (orientation scheduling) |
| **Human gate** | HRBP reviews and signs off on compliance checklist |
| **Output** | Compliance checklist PDF with all signatures |

### 4. Facilities Agent

| Property | Value |
|---|---|
| **Goal** | Physical/logistical setup ready by day 1 |
| **Tools** | Desk booking API, badge system API, shipping API (laptop/monitor) |
| **Human gate** | IT confirms laptop is imaged before shipping |
| **Fallback** | If remote, skip desk/badge, trigger equipment shipment |

### 5. Onboarding Buddy Agent (optional, triggered last)

| Property | Value |
|---|---|
| **Goal** | New hire feels welcomed and has a clear first week |
| **Tools** | Calendar API (schedule 1:1s), Slack API (welcome message, channel invites), Confluence/Notion API (30-60-90 day plan) |
| **Output** | Personalized onboarding doc + calendar pre-filled for week 1 |

---

## State Machine

```
[New hire record received]
         │
         ▼
[Orchestrator validates + plans]
         │
         ├──► IT provisioning (parallel) ──┐
         ├──► HR compliance (parallel) ────┤
         ├──► Facilities setup (parallel) ─┤
         │                                 │
         ◄──────── wait for all ────────────┘
         │
    ┌────┤
    │    ▼ pass
    │  [Buddy agent triggered]
    │         │
    │         ▼
    │  [Onboarding summary generated]
    │         │
    │         ▼
    │  [Human review gate: onboarding lead approves]
    │         │
    │         ▼
    │  [Welcome email sent, tracker marked complete]
    │
    └──── ► fail → [Escalation: pause, notify human, log reason]
```

---

## Key Design Decisions

| Decision | Rationale | Source |
|---|---|---|
| Sub-agents as tools (orchestrator calls them) | Prevents context window overflow in orchestrator | Anthropic (2025), "Building Effective AI Agents" |
| Parallel fan-out for IT/HR/Facilities | Independent sub-tasks; no sequential dependency | Anthropic (2025), "Parallel workflows" |
| Human approval gates at escalation boundaries | Safety-critical actions (provisioning, compliance) need human relay | Okta (2026); WEF (2025) |
| Task contracts per sub-agent | Explicit goal/permission/quality/verification spec prevents drift | arXiv:2605.22634, "Contractual Skills" |
| State persistence in LangGraph | Enables resumption after human-in-the-loop pause | Anthropic (2025), "Memory tools" |
| Orchestrator consolidates, doesn't generate | Separates decision generation from governance | Bandara et al., arXiv:2512.21699 |

---

## References

- Anthropic (2025). *Building Effective AI Agents: Architecture Patterns and Implementation Frameworks.*
- Bandara, E., Gore, R., Foytik, P., Shetty, S., Mukkamala, R., Rahman, A., Liang, X., Bouk, S. H., Hass, A., & Rajapakse, S. (2025a). A practical guide to agentic AI transition in organizations. arXiv:2602.10122.
- Bandara, E., Gore, R., Liang, X., Rajapakse, S., Kularathne, I., Karunarathna, P., Foytik, P., Shetty, S., Mukkamala, R., & Rahman, A. (2025b). Towards responsible and explainable AI agents with consensus-driven reasoning. arXiv:2512.21699.
- Krishnan, N. (2025). AI agents: Evolution, architecture, and real-world applications. arXiv:2503.12687.
- GovernSpec (2026). Contractual skills: A GovernSpec design framework for enterprise AI agents. arXiv:2605.22634.
- Okta (2026). *The Future of AI Security: The Right Architecture for Agents.*
- World Economic Forum (2025). *AI Agents in Action: Foundations for Evaluation and Governance.*

---

## MVP Path (1-2 weeks)

1. Orchestrator + IT agent only (most painful, most repetitive)
2. Hardcoded Slack/email tool integrations
3. Human-in-the-loop gate before account creation
4. Simple SQLite/JSON state file (no DB yet)
5. Metrics: time-to-provision vs. manual baseline
6. Add HR and Facilities incrementally — modular design makes this a config change, not a rewrite
