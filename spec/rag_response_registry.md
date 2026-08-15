# FounderFinder — RAG Response Registry

Retrieval corpus of canonical response cards, one per match tier. The copilot computes a
tier for each opportunity (see the scoring schema), retrieves the matching exemplar below,
and imitates its shape, voice, and section structure — filling it with the real
opportunity's data.

**How to use**

1. Score the opportunity → tier (Strong / Maybe / Weak / Poor).
2. Retrieve the record whose tier matches.
3. Reproduce its sections and tone with the live opportunity's fields.
4. Obey the CTA rule and render rule for that tier exactly.

**Global rules**

- Every served card shares a spine: **Agency / Award name**, **$ amount / timing**, **Scorecard (SWOT)**.
- Detail scales inversely with confidence: Strong promises, Weak explains.
- Never assert eligibility as a guarantee — use "appears / likely / verify."
- **Poor / non-match is never rendered as a card** (see Record 4).
- The **Missing** (Record 2) and **criteria question** (Record 3) sections are **dynamically
  generated from the opportunity's requirement gap** — the requirements it declares that the
  profile couldn't confirm. They are NOT a fixed socioeconomic checklist. Items may span
  set-asides, certifications (CMMC, FedRAMP, AS9100), clearances, key-personnel/PI criteria,
  past performance, registrations, and **founder/leadership ownership status** (most
  set-asides are decided by who owns ≥51% of the company). Confirming an item re-runs the
  gates and may promote the match.

Field key: `[[brackets]]` = slot filled from live opportunity/profile data.

---

## Record 1 — Strong match

- **Internal tier:** GREEN ("Likely Fit")
- **Retrieve when:** all gates pass, no unknowns, relevance ≥ 0.70
- **CTA rule:** show **Begin Application** (commit action)
- **Render rule:** full SWOT + Alignment + Next steps

**Example response**

> **Department of the Air Force — AFWERX**
> SBIR Phase II: AI-Enabled Network Threat Detection
>
> **$750,000 – $1,800,000** · Open, closes Oct 1, 2026
>
> **Scorecard (SWOT)**
> - **Strengths:** U.S. small business, eligible. Direct domain match — AI-driven threat detection. Active R&D component.
> - **Weaknesses:** Requires a technical volume and prior feasibility work; award decision typically 3–6 months out.
> - **Opportunities:** Phase II sets up a Phase III sole-source **contract** — a path to government revenue, not just capital.
> - **Threats:** Competitive topic; must demonstrate dual-use.
>
> **Alignment — why you're a potential fit**
> - U.S.-based small business — yes
> - AI technology — yes
> - Cybersecurity application — yes
> - Active R&D — yes
> - Need ($1M–$3M) fits award band — yes
>
> **Next steps**
> - Confirm an active SAM.gov entity registration (UEI).
> - Read the full solicitation: `[[opportunity_url]]`
> - Note the deadline: Oct 1, 2026.
> - Draft the technical volume against the topic requirements.
>
> **[ Begin Application ]**

---

## Record 2 — Maybe match

- **Internal tier:** YELLOW ("Verify Eligibility")
- **Retrieve when:** relevance ≥ 0.45, OR any gate returns UNKNOWN
- **CTA rule:** show **Go** (explore action, not a commitment)
- **Render rule:** diminished SWOT + **Missing** block + Next steps (may include a certification action)

**Example response**

> **Department of Homeland Security — CISA**
> Combined Synopsis: Managed Threat-Detection Services
> *(Total Small Business Set-Aside)*
>
> **Est. $500,000 – $2,000,000** · Responses due Sep 15, 2026
>
> **Scorecard (SWOT — diminished)**
> - **Strengths:** Strong domain fit. Small-business set-aside you appear to qualify for.
> - **Weaknesses:** Past-performance history and a facility posture may be expected.
>
> **Missing / to verify** *(generated from this opportunity's requirement gap)*
> - Active SAM.gov registration (UEI) not confirmed.
> - **CMMC Level 2 / NIST 800-171** compliance required — status unknown.
> - **Facility clearance or cleared personnel** may be needed — none on file.
> - Ownership: small-business set-aside appears to fit; ownership breakdown unverified.
> - NAICS 541512 alignment on your profile is unverified.
>
> **Next steps**
> - Confirm or complete your SAM.gov registration.
> - Confirm CMMC / NIST 800-171 posture (or start the assessment).
> - Verify your primary NAICS code.
> - Consider HUBZone certification — it would unlock additional set-aside contracts like this one.
>
> **[ Go ]**

---

## Record 3 — Weak match

- **Internal tier:** ORANGE ("Adjacent")
- **Retrieve when:** relevance ≥ 0.25 (below Maybe), gates not failed
- **CTA rule:** **no apply button** — present a criteria question instead
- **Render rule:** SWOT + honest hedge copy + **interactive criteria checklist**. Confirming a criterion re-runs the set-aside / geography gates and **may promote** this match to a higher tier.

**Example response**

> **Department of Energy — Technology Commercialization Fund**
> Early-Stage Dual-Use Innovation (general)
>
> **$100,000 – $500,000** · Rolling
>
> **Scorecard (SWOT)**
> - **Strengths:** Funds early-stage dual-use technology.
> - **Weaknesses:** Not cyber-specific; your AI work is adjacent, not central, to this program's focus.
> - **Opportunities:** Could seed a pilot outside your core federal-cyber path.
> - **Threats:** Low domain overlap — likely a stretch on relevance.
>
> **Why we still shared this**
> This isn't a very strong match. We surfaced it because it funds early-stage dual-use technology and your work is adjacent — not because we think it's a clear fit.
>
> **Do any of these pertain to you or your team?**
> *(Drawn from what this opportunity requires but we couldn't confirm. Confirming one may
> improve this match and unlock reserved opportunities.)*
>
> Ownership & team *(most set-asides are decided by who owns ≥51%)*
> - [ ] Woman-owned (≥51%)
> - [ ] Veteran- / service-disabled-veteran-owned (≥51%)
> - [ ] 8(a) / socially & economically disadvantaged ownership
> - [ ] Native American / Tribal-owned
> - [ ] Founder or key staff holds a security clearance
> - [ ] Principal investigator primarily employed by the company (SBIR)
>
> Company credentials & location
> - [ ] HUBZone (principal office in a HUBZone)
> - [ ] Rural location
> - [ ] Relevant certification (e.g., CMMC, FedRAMP, AS9100, ISO)
> - [ ] Prior federal contract or SBIR/STTR award (past performance)
>
> *(No application step yet — answer the above and we'll re-check eligibility.)*

---

## Record 4 — Poor / non-match

- **Internal tier:** RED ("Probably Not a Fit")
- **Retrieve when:** any gate FAILS, or relevance < 0.25
- **CTA rule:** none
- **Render rule:** **DO NOT SERVE AS A CARD.** A poor match is withheld from the founder entirely. Serving a bad opportunity costs the founder's time and trust even when labeled — so it is suppressed, not shown.

**Behavior — two distinct cases**

**Per-opportunity (one bad match among good ones)**
Output: nothing. The opportunity is silently excluded from the results. Do **not** render a
red card, a "not a fit" notice, or a rejection reason to the founder. (Reasons may be
logged internally for tuning only.)

**Whole-portfolio (every candidate is poor / weak)**
When nothing clears the no-match bar, do not return a wall of silence. Return one honest,
plain-language message in place of results:

> **No strong match right now.**
>
> Federal grants and contracts look like a weak fit for your company at the moment — the
> closest options are adjacent, not core. That's an honest read, not a gap in our search.
>
> **What can change this:**
> - Tell us about set-aside status (woman-owned, veteran, HUBZone, 8(a), Native American, rural) — it can unlock reserved opportunities.
> - Revisit as new opportunities post; we'll watch for better fits.
> - Consider adjacent non-dilutive paths (state programs, accelerators).

**Rationale.** Correctly withholding a bad opportunity is a success, not a gap: no wasted
application time, no damaged trust. Saying "probably not a fit" out loud earns credibility
that a forced, hallucinated match destroys.
