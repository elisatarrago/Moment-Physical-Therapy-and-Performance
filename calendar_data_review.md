# Moment PT — Schedule Data Review
**File reviewed:** `moment_schedule_snapshot.html`
**Week under review:** Monday July 20 – Friday July 24, 2026 (59 appointments, 6 providers, 24 patients)
**Review date:** July 18, 2026 — no code changes made, findings only.

---

## 1. Critical conflicts (double-bookings / impossible schedules)

### 1.1 Owen Marsh (`marcus`) double-booked — Monday, LIC
- **9:00–10:00** Sofia Marino, Initial Evaluation (`a05`)
- **9:30–10:00** Leo Martin, PT Follow-Up (`a06`)
- Overlap: **9:30–10:00 AM**, same provider, same location, two different patients.

### 1.2 Kendo Nakamura double-booked — Tuesday, Midtown
- **10:00–10:30** Grace Lin, PT Follow-Up (`b02`)
- **10:00–10:30** Robert Ellison, PT Follow-Up (`b03`)
- Identical time slot, same provider, two different patients — exact duplicate double-booking.

### 1.3 Kendo Nakamura — impossible location transition — Friday
- **11:00 AM–12:00 PM** Maria Santos at **Midtown** (`e03`)
- **12:00–12:30 PM** Wesley Tan at **LIC** (`e14`)
- Zero minutes between appointments at two different physical locations — not achievable regardless of travel method. Also notable because Kendo works exclusively at Midtown every other day this week; this is the only LIC appearance.

### 1.4 Hannah Cole booked while on approved PTO — Friday, LIC
- Roster + `PTO_BLOCKS` both mark Hannah **Out (PTO) Thu July 23 – Fri July 24** at LIC.
- Thursday is correctly clear of Hannah appointments (her LIC patients Grace Kim and Omar Haddad were reassigned to Owen Marsh that day — correct coverage).
- **Friday is not clear** — Hannah still shows 3 booked visits:
  - 9:00 AM Grace Kim (`e08`)
  - 10:00 AM Leo Martin (`e09`)
  - 11:00 AM Omar Haddad (`e10`)
- Wesley Tan's Friday LIC slot (`e14`) *was* correctly reassigned to Kendo as PTO coverage — suggesting the coverage plan exists but was only partially applied. These three appointments appear to be the ones that were missed.
- **Compounding UI bug:** in `renderCalendar()`, the PTO band only renders when a specific location is selected (`currentLoc !== "All"`, line ~462). In the default "All locations" view — the view staff land on first — this conflict is completely invisible. You'd only catch it by filtering to LIC.

---

## 2. Data integrity issues

### 2.1 Duplicate booking — Ava Coleman, Monday
- `a01`: 8:00–9:00 AM, Kendo, Midtown, "session 5 of 12"
- `a11`: 2:30–3:00 PM, Kendo, Midtown, "session 6 of 12"
- Same patient, same provider, same location, same day. The patient roster (`PATIENTS`) lists Ava at `used: 8` sessions already completed as of July 15 — meaning her *next* session should be #9, not #5 or #6. These two entries look like a leftover/copy-paste artifact from an earlier point in her plan rather than two real visits. Recommend removing the erroneous entry and correcting the session number on the one that's real.

### 2.2 Plan-of-care linked before evaluation — Hana Yoshida
- Patient roster shows her plan as **"PT Follow-Up (12)"** with `used: 0`, even though her Tuesday appointment (`b08`) is still an **Initial Evaluation** (unbilled, "Not linked" convention used elsewhere for new patients like Sofia Marino and Nathan Powell).
- Her plan of care shouldn't be established in the system until after the eval determines it. This is inconsistent with how the other two new-patient records are handled.
- Separately: her eval (Tue) is with **Diego**, but both her follow-up sessions (Wed `c07`, Thu `d07`) are with **Amelia** — a provider handoff with no annotation. Worth confirming this is an intentional routing decision.

### 2.3 Provider id/name mismatch
- Therapist record: `{id:"marcus", name:"Owen Marsh", ...}` — the internal id doesn't match the displayed name at all. This is almost certainly a leftover from a renamed record. It doesn't break rendering (the id is just a lookup key) but it's a real trap for anyone editing this data later — searching for "Owen" won't find `id:"marcus"`.

---

## 3. Provider continuity / caseload fragmentation

Three Midtown patients split unpredictably between **Priya** and **Kendo** mid-week with no visible reason:

| Patient | Mon | Tue | Wed | Thu | Fri |
|---|---|---|---|---|---|
| Maria Santos | Priya | Priya | Priya | **Kendo** | Kendo |
| Ethan Brooks | Kendo | Kendo | **Priya** | Kendo | Kendo |
| James Ford | Priya | — | **Kendo** | Kendo | — |

- Priya works Mon–Thu (part-time) and **has zero appointments on Thursday** — a full idle day for a scheduled working provider — while her regular patient Maria Santos's Thursday session was routed to Kendo instead of staying with Priya, who was available.
- Recommend clarifying whether this is deliberate team-based/co-treatment care or an unintentional fragmentation of caseload. Either way, Priya's empty Thursday is a concrete utilization gap worth fixing.

Similarly at LIC, Grace Kim, Leo Martin, and Omar Haddad rotate between Owen Marsh and Hannah Cole through the week (partly explained by Hannah's PTO, see 1.4) without any "covering for X" label — front desk and patients have no way to tell a provider swap is intentional vs. an error.

---

## 4. Care-gap / follow-up risks (no scheduling error, but worth surfacing)

- **Olivia Grant** — 11 of 12 sessions used, `next: null`. One session short of completing her plan with nothing on the books.
- **Nina Patel** — last visit was a **No-Show** (July 16), `next: null`. No rebooking follow-up scheduled.
- **Ethan Brooks** — reaches "session 12 of 12" this Friday (`e02`), completing his package with no re-evaluation/renewal appointment yet visible.

None of these are calendar conflicts, but they're the kind of thing that silently falls through the cracks without a recall/renewal-tracking mechanism.

---

## 5. Workload distribution this week

| Provider | Appointments | Notes |
|---|---|---|
| Amelia Ruiz | **17** | Heaviest load, all at Soho. Wed and Fri both have zero-gap stretches (Wed 11:30 AM–1:30 PM, Fri 11:30 AM–1:30 PM back-to-back with no break). |
| Kendo Nakamura | 15 | Mostly Midtown; the one LIC appointment Friday causes the impossible-transition conflict (1.3). |
| Diego Álvarez | 9 | Mostly Soho, one Midtown eval Thursday (adequate 2-hour gap before returning to Soho — not a conflict). |
| Owen Marsh (`marcus`) | 8 | LIC. |
| Priya Shah | 5 | Part-time (Mon–Thu) but idle all Thursday — see §3. |
| Hannah Cole | 5 (3 invalid) | 3 of these fall inside her own approved PTO — see §1.4. |

Daniel Cho and Rachel Green are both seen **every single day this week** by Amelia — confirm this daily cadence is a deliberate post-op protocol requirement rather than a byproduct of not spreading the caseload; if intentional, no action needed, just flagging since it contributes to Amelia's load being ~2x her nearest peer.

---

## 6. Recommendations to fix (next pass)

1. **Resolve the 4 conflicts in §1** — reassign or reschedule Leo Martin (1.1), Grace Lin or Robert Ellison (1.2), Wesley Tan or Maria Santos (1.3), and the 3 Friday Hannah appointments (1.4) to an actual available covering provider.
2. **Enforce PTO as a hard constraint, not a decorative band.** Right now `PTO_BLOCKS` is purely visual and only shown when filtering by location — it didn't stop Hannah's Friday slots from being booked and doesn't warn staff in the default "All locations" view. Booking against a PTO block should be blocked or at minimum flagged everywhere, not just in a filtered view.
3. **Remove the duplicate Ava Coleman Monday entry** and correct her session numbering to match her actual completed-session count (2.1).
4. **Don't link a plan of care until after the evaluation** — align Hana Yoshida's record with the "Not linked (new patient)" pattern used for Sofia Marino / Nathan Powell (2.2).
5. **Fix the `marcus`/Owen Marsh id mismatch** so the data key matches the displayed name (2.3) — low risk today, but a maintenance trap.
6. **Add a "covering for [provider]" tag** on reassigned appointments so mid-week provider switches (Priya↔Kendo, Marcus↔Hannah) are self-explanatory instead of looking like errors.
7. **Build in travel/buffer time** between locations for any provider who crosses sites same-day, and a standing lunch/break window so no provider (currently Amelia) has a 2-hour fully back-to-back stretch.
8. **Add a recall/renewal list** surfacing patients near package completion (Olivia Grant, Ethan Brooks) or with a recent no-show and no rebook (Nina Patel), so these don't require someone to notice manually.
9. **Rebalance Soho caseload** between Amelia (17 appts) and Diego (9 appts) if clinically feasible, given Amelia is carrying roughly 2x Diego's load this week.

---

*No code was modified as part of this review. Once you confirm which items to act on, the next step would be correcting the `APPTS`/`PATIENTS` data and, optionally, making the PTO check a real scheduling guard rather than a display-only band.*
