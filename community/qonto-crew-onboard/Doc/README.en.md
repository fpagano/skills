# 🧑‍🚀 qonto-crew-onboard — One-sentence financial onboarding for every new hire

> **Qonto × Anthropic MCP Hackathon submission** · Agent Skill for the Qonto MCP
> "Alex starts Monday as a developer" → the invitation goes out and a policy-compliant capped card request awaits SCA.

---

## 🎯 Why this matters (usefulness)

Every new hire triggers the same manual ritual: invite them on Qonto, create their card with the right caps, prepare their first day — and every departure triggers the reverse, usually forgotten until the frozen-too-late card bites. `qonto-crew-onboard` turns both into one sentence:

1. **A per-role policy** — defined **once** with the user (card type, monthly caps, online-only, team per role), then applied to every arrival. Never invented: a role without a policy triggers questions
2. **One-sentence onboarding** — "Alex starts Monday as a developer": `create_membership` (the invitation goes out), `create_team` if the team is new, `create_card_request` — a capped card **request** that conforms to the policy and waits for SCA approval in the Qonto app
3. **Mirror-image offboarding** — "Sam is leaving Friday": cards **frozen** (`change_card_status`) + a recovery checklist. Nothing is deleted — definitive revocations stay in the app

Would someone use this on a Monday morning? That's literally when new hires show up.

## 📋 Prerequisites

| Prerequisite | Detail | Required |
|---|---|---|
| Qonto **production** account | The skill adapts to any organization — `get_organization` first, nothing hardcoded | ✅ |
| Country | **Universal** — memberships, teams and cards work the same in every Qonto country (FR, DE, ES, IT, AT, NL, BE, PT). No fiscal logic involved | ℹ️ |
| Qonto MCP connected | Official connector (claude.ai / Claude Desktop), OAuth login | ✅ |
| **Admin/Owner** role | Inviting members and requesting cards require it — checked via `get_authenticated_membership`, honestly announced otherwise | ✅ |
| A per-role policy | Defined once with the user. **Never invented by the skill** | ✅ user-decided |
| Plan with available seats | The skill **asks your plan** and counts members before inviting | ℹ️ verified by asking |

## 🗂 The per-role policy (example — yours is defined with you)

| Role | Card | Monthly cap | Online-only | Team |
|---|---|---|---|---|
| Developer | Virtual | €200 | Yes | Tech |
| Account executive | Physical | €1,000 | No | Sales |
| Ops | Virtual | €500 | Yes | Operations |

> ⚠️ This table is **made up** for illustration. The policy is **always** decided by the user, never by the skill — an unknown role triggers questions, not a guess.

## ⚙️ How it works

1. **Account snapshot**: `get_organization` first, `get_authenticated_membership` (requester's role — Admin/Owner needed for the writes, honestly stated otherwise), existing members, teams and cards
2. **Per-role policy**: loaded or defined with the user; plan member limit checked **before** inviting (plan asked, members counted)
3. **One sentence**: "Alex starts Monday as a developer" → name, role, start date extracted; the email is asked for (never guessed); **a full recap is shown before any write**
4. **Execution — every write individually confirmed**: `create_membership` (invitation), `create_team` if new, `create_card_request` — a card *request* pending SCA approval in the app, never presented as an active card
5. **Welcome checklist**: first-day information, card policy, pending items and reminders
6. **Mirror offboarding**: cards frozen (`change_card_status`, reversible), recovery checklist — card, equipment, accesses. Nothing deleted: definitive revocations happen in the Qonto app

**The security model** (not a limitation): solid arrows are risk-free reads; every dashed write requires explicit confirmation in the conversation. And the card is **never** created directly: `create_card_request` produces a *request* that lands in the Requests section of the Qonto app, where an Admin/Owner approves it with their own SCA.

## 🧪 Holds up on messy data

- Authenticated user isn't Admin/Owner? → honest warning, read-only preview of what would be done
- No policy for the role? → questions, never a guess
- Email missing from the sentence? → asked for, never guessed
- Email already invited? → detected in `list_memberships`, invitation skipped with the reason
- Team already exists (different casing)? → matched against `list_teams`, no duplicate created
- Plan seat limit unknown? → the skill asks the plan and counts members before inviting
- Offboarding a member with no cards? → checklist only, and the skill says exactly what the app must handle

## 📤 Output formats

| Output | Format | When |
|---|---|---|
| **Conversation reply** | Markdown tables: onboarding recap (✅ done / 🕐 pending SCA / ⏭ skipped), card-request status, welcome checklist, offboarding checklist | **Always** — the baseline |

## 🎬 Demo video

The ≤ 3-minute demo attached to the PR walks through: the manual ritual → the policy defined once → "Alex starts Monday as a developer" → **the invitation out, the capped card request awaiting SCA, the welcome page ready** → the one-command offboarding freeze. It runs live on a real production account (with fictional hires).

## 💡 Roadmap ideas

- Batch onboarding ("3 interns start on the 1st") — same workflow, grouped recap, one-by-one confirmations
- Card-request tracking with `list_requests` — "where is Alex's card?"

## 🛡 Guardrails

- NEVER writes without explicit confirmation in the current conversation — full recap first, then each write (`create_membership`, `create_team`, `create_card_request`, `change_card_status`) confirmed one by one; a pending request is never presented as done
- **The policy is user-decided, never invented** — unknown role = questions
- Plan member limits: ask the plan, count members, warn before inviting
- **Offboarding never deletes anything**: freeze (reversible) + checklist; definitive revocations happen in the Qonto app
- Admin/Owner checked first (`get_authenticated_membership`), honest degradation otherwise
- Emails never guessed · pagination ≤ 50 everywhere

---

*Submission docs: `SKILL.md` (the skill itself) · step-by-step guide: `docs/PROCEDURE.en.md` · rich docs in `docs/` (HTML, FR/EN).*
