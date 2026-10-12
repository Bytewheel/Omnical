# Omnical — Liturgical movable days & precedence for the holiday sets

**Date:** 2026-10-12
**Working tree:** `~/router-dav` (check `git status` first; the holidays live at the repo root, the
serving code in `rustical/`)
**Practices:** this is a plan; code lives in `~/router-dav`. All changes are in
`scripts/generate-holiday-ics.py` plus the regenerated `holidays/*.ics` it emits. **No Rust changes
and no new dependencies** — the import/refresh engine (`crates/frontend/src/routes/holidays.rs`,
`holidays://{set}` source rows, per-UID refresh via `linked_platforms.rs`) already handles added,
changed and removed events.

---

## 0. The problem, verified today (2026-10-12)

The four Christian sets compute fixed dates and a handful of movable ones, but they are missing
every "Nth Sunday after X" observance, and when a fixed feast and a Sunday collide both are emitted
(or the wrong one wins). Concretely, in the generated files on disk:

- **`rc-1962.ics`** — has **no Sundays after Epiphany (2nd–6th) and no Sundays after Pentecost
  (2nd–24th)** at all. The user's exact case: Sun **2026-10-18** is *St Luke, Evangelist* and
  nothing else — the day's proper observance (the **21st Sunday after Pentecost**, with St Luke
  merely commemorated) is absent. Same for every other Sunday of the Pentecost season and the
  Epiphany season.
- **`rc-current.ics`** — has both Ordinary Time segments but with **systematic off-by-one
  numbering**: the Sunday *before* Christ the King is labeled "Ordinary Sunday 34" (2026-11-15) —
  but Christ the King itself (2026-11-22) **is** the 34th; romcal's live calendar confirms today
  (Mon 2026-10-12) is the *28th* week, i.e. Oct 11 is the 28th Sunday, while our file says 29. The
  early segment is off by one too: 2026-01-18 is "Ordinary Sunday 1" but the Baptism (Jan 11)
  counts as the 1st, so Jan 18 is the 2nd. **Easter Sundays III–VII are missing** (only Easter II /
  Divine Mercy exists). **Collisions are unhandled**: 2026-11-01 emits BOTH "All Saints" and
  "Ordinary Sunday 32" (and the number is wrong anyway — All Saints supersedes the 31st).
- **`anglican.ics`** — no Lent Sundays, no Sundays of Easter, no Sundays after Trinity, no Sundays
  before Lent; only Advent Sundays and a few named days exist.
- **`orthodox.ics`** — the Pascha cycle trunk exists (Lazarus Saturday → Pentecost → All Saints)
  but none of the *named* Sundays of Great Lent, none of the Sundays after Pascha (Thomas,
  Myrrhbearers, Paralytic, Samaritan, Blind, Fathers of Nicaea), and no Sundays after Pentecost.
- A latent bug affecting all four sets: `_first_sunday_after(date)` returns `date` itself when it
  is already a Sunday — so for **2030, where Epiphany (Jan 6) is a Sunday**, Holy Family / Baptism
  of the Lord / the Anglican "1st Sunday after the Epiphany" all land *on Jan 6* instead of Jan 13.
  (2027–2030 are in the generated window; this is a live defect.)

The other religions need nothing of this kind: Judaism, Islam, Hinduism, Buddhism, Sikhism and
China are lunar/solar-relative by construction and have no weekly-cycle precedence; the civil sets
already use `nth_weekday` rules.

## 0.1 The example, verified against two published 1962 Ordines

> "in the 1962 Catholic Liturgy, the Third Sunday after Pentecost would take precedence over St
> Luke, Evangelist, since St Luke Evangelist falls on a Sunday this year"

The principle is right and the 2026 case is precisely (Sunday *after* Pentecost beats St Luke; on
2026-10-18 the Sunday in question is the **21st**):

- **LMS Ordo 2025–2026** (free PDF, lms.org.uk/ordo → `final_ordo_1.42_2025-2026.pdf`), p. 89:
  *"Sun 18 — 21st SUNDAY after PENTECOST II Cl G … Commem of St Luke Evangelist at Low Mass …
  At Lauds Commem of St Luke Evangelist."*
- **FIUV Ordo 2025–2026** (free PDF, same page), 18 Oct: *"DOM. XXI POST PENTECOSTEN, De ea,
  II cl. … comm. S Lucae Evangelistae."*

So the Sunday is kept (II class, green) and the feast is **commemorated**, not transferred — our
set must emit one event for the Sunday, with the commemoration noted.

---

## 1. Authoritative sources (what the implementer compares against)

| Rite | Primary (normative) | Practical verification |
|---|---|---|
| 1962 (EF) | *Rubricarum instructum* **1960 Code of Rubrics** (AAS 52 [1960]) — General Rubrics nn. 112–117 + the appended **Tabula** of occurrence/concurrence; reprinted in every 1962 Missal (e.g. Baronius, pp. *De anno liturgico* appendix) | **LMS Ordo 2025–26** + **FIUV Ordo 2025–26** (both free PDFs, cover through 1 Jan 2027); LMS Ordo 2024–25 for the 2025 Lateran case; **Divinum Officium** (github.com/DivinumOfficium) as a mechanical oracle for 2027–2030 — GPL, use as a **test oracle only, copy no code**; each year's LMS/FIUV Ordo appears the autumn before |
| Roman Rite (OF) | **Universal Norms for the Liturgical Year and the Calendar** (Roman Missal): nn. 4–7 (Sunday), 23 (Easter Sundays), 30 (Lent), 43 (OT), and n. 59 **Table of Liturgical Days** + n. 60 (transfers). Full text: romcal.net/documents/norms.php (also liturgyoffice.org.uk GNLY.pdf) | **romcal.net** renders any year's universal/US calendar on demand; the USCCB publishes annual Ordines; today's page already confirmed "Monday of the Twenty-Eighth Week of Ordinary Time" for 2026-10-12 |
| Anglican | **Common Worship** (CofE 2000): the Calendar + the 8 "Rules to accompany the Calendar" (in every CW volume/lectionary) | The CofE's published Sunday-name calendar (lectionary annuals; the Sunday names are also on every CofE lectionary page per year) |
| Orthodox | The **Typikon** as embodied in the Triodion & Pentecostarion (fixed structure, named Sundays) | **oca.org** or **goarch.org** per-day calendars for 2026–2030 (they print the Sunday names civil-converted, Julian reckoning) |

The generator itself must stay **no-network, stdlib-only, deterministic** (its existing design
constraint) — the sources above are used at *development* time to derive rules and hard-code
self-test anchors, never at runtime.

## 1.1 Verified 2026 anchors (encode these in `self_test()`; today's values are from the two PDFs)

1962: Advent Sunday 2025-11-30 (already correct in the set); Septuagesima Sun **2026-02-01** ✓ (in
set); Holy Family **Jan 11** ✓; 2nd/3rd Sundays after Epiphany **Jan 18 / Jan 25**; Sundays after
Pentecost **Jun 7 (2nd) … Oct 18 (21st)**; Christ the King **Oct 25** ✓; All Saints (Sun) **Nov 1**
with commemoration of the 23rd Sunday after Pentecost; **5th Sunday remaining after the Epiphany
Nov 8; 6th Nov 15**; **24th & Last Sunday after Pentecost Nov 22**; Advent Sunday **Nov 29**.

2025 (for the feast-on-Sunday rule, from LMS 2024–25): Nov 9 = Dedication of the Lateran
(II class, feast **of the Lord**) kept over the would-be 22nd Sunday after Pentecost — contrast
with St Luke 2026 (saint's feast, II class) which does **not** beat a II class Sunday.

OF: Oct 11 = 28th Sunday; **Nov 1 = All Saints (31st superseded)**; Nov 8/15 = 32nd/33rd; Nov 22 =
34th = Christ the King; Easter III–VII = **Apr 19 / Apr 26 / May 3 / May 10 / May 17**; Jun 7 =
9th (6th–8th absorbed by Lent/Easter; Trinity supersedes the 8th).

---

## 2. The rules to encode

### 2.1 `rc-1962` — Sundays after Pentecost (reconstructed & verified against LMS+FIUV 2026, LMS 2025)

1. Trinity Sunday is the **1st** Sunday after Pentecost (emitted as "Trinity Sunday", I class, as
   today). The next Sundays are numbered **2nd … 24th**.
2. Number the season's Sundays **forward**: every Sunday slot from Trinity+1 through the last
   Sunday before Advent consumes the next number — **including slots occupied by a higher-ranked
   day** (Christ the King, All Saints, a II-class feast of the Lord). The superseded number is
   *consumed* (its Mass is what the week's ferias use; the Ordo shows "Commem of the Nth Sunday").
   2026: … 21st (Oct 18) → CTK = 22nd's slot (its ferias say "Mass of 22nd Sunday" per the Ordo) →
   All Saints = 23rd's slot ("Commem of 23rd Sunday after Pentecost" ✓ printed).
3. Forward numbering stops at the **23rd**. The **last Sunday before Advent is always the 24th**
   ("24th & Last Sunday after Pentecost").
4. The Sundays between the 23rd and the 24th are the **impeded Sundays after Epiphany resumed**
   (those that did not fit between Holy Family and Septuagesima), assigned **from the end**: the
   Sunday immediately before the 24th takes the *highest* impeded number, the next one down takes
   the next, and leftover impeded numbers are permanently dropped. 2026: kept in January were the
   2nd and 3rd (impeded: 4th, 5th, 6th); the two free slots Nov 8/15 → **5th and 6th** (LMS:
   "5th/6th SUNDAY remaining after the EPIPHANY"; FIUV: "DOM. V/VI QUI SUPERFUIT POST EPIPHANIAM").
   The 4th is simply not resumed in 2026.
5. Sundays after Epiphany, January segment: Holy Family = 1st (feast, II class), then **2nd … 6th**
   as fit before Septuagesima; 2030 has four (2nd–5th, Jan 20 … Feb 10 — with Epiphany itself on
   Sunday Jan 6, Holy Family is Jan 13); 2027 has only the 2nd (Jan 17).

Per-year outcomes of this algorithm for 2026–2030 (the implementer re-derives and then verifies
each against the annual Ordo/Divinum Officium before committing):

- **2026**: 2nd–21st = Jun 7–Oct 18; CTK Oct 25 (=22nd); All Saints Nov 1 (=23rd); resumed 5th/6th
  = Nov 8/15; 24th = Nov 22.
- **2027** (Easter Mar 28): 2nd–23rd = May 30–Oct 24; CTK Oct 31; resumed slots Nov 7/14 → by rule
  4 the **5th and 6th** (impeded: 3rd–6th); 24th = Nov 21.
- **2028** (Easter Apr 16): 2nd–20th = Jun 18–Oct 22; CTK Oct 29 (=21st); 22nd = Nov 5, 23rd =
  Nov 12; resumed **6th** (the only impeded one) = Nov 19; 24th = Nov 26.
- **2029** (Easter Apr 1): 2nd–22nd = Jun 3–Oct 21; CTK Oct 28 (=23rd); resumed **4th/5th/6th** =
  Nov 4/11/18; 24th = Nov 25.
- **2030** (Easter Apr 21): 2nd–19th = Jun 23–Oct 20; CTK Oct 27 (=20th); 21st = Nov 3, 22nd =
  Nov 10, 23rd = Nov 17, **24th = Nov 24**; the one impeded Epiphany Sunday (6th) is *not*
  resumed (no slot — kept in January were the 2nd–5th).

### 2.2 `rc-1962` — occurrence (what a curated set needs; disposition cited to the 1960 Code / Tabula)

Rank ladder for the days our set emits: **I class Sunday** (Advent, Lent I–IV, Passion/Palm,
Easter, Low Sunday, Pentecost) > **I class feast** (Christmas, Epiphany, Annunciation, St Joseph,
Precious Blood, St Joseph the Worker, SS Peter & Paul, Assumption, All Saints, All Souls,
Immaculate Conception, St Michael, Christ the King) > **II class Sunday** (Septuagesima/
Sexagesima/Quinquagesima — confirmed II cl by the Ordo — Sundays after Epiphany, Sundays after
Pentecost, resumed Sundays) > **II class feast** > **III class feast** / ember days / octaves.

Dispositions (each with a verified or to-verify vector in §3.3):

| Collision | Outcome | Vector |
|---|---|---|
| I class Sunday vs I class feast | Sunday kept; feast **transferred** to the next day free of I/II class | St Joseph 2028-03-19 (Lent IV Sunday) → Mon 2028-03-20; Immaculate Conception 2030-12-08 (Advent II) → Mon 2030-12-09 (the 2013 precedent) |
| I/II class Sunday vs **II class saint's feast** | Sunday kept; feast **commemorated** only | **St Luke 2026-10-18 (verified)**; SS Simon & Jude 2029-10-28; Visitation 2028-07-02; St John Baptist 2029-06-24 |
| II class Sunday vs **II class feast of the Lord** (Holy Name, Transfiguration, Exaltation of the Cross, Dedication of the Lateran, Sacred Heart…) | **Feast kept**; Sunday commemorated | Lateran 2025-11-09 (verified, LMS 2024–25); Transfiguration 2028-08-06 |
| I class feast vs II class Sunday | Feast kept; Sunday commemorated | All Saints 2026-11-01 (verified); Assumption 2027-08-15; Precious Blood 2029-07-01; St Michael 2030-09-29 |
| Any Sunday vs III class feast | Sunday kept; feast omitted (no event) — verify whether the 1960 Code allows a commemoration; either way we emit only the Sunday | St Anthony of Padua 2027-06-13; St Patrick 2030-03-17 |
| Privileged Paschal octave-day / Easter Sunday vs II class feast | Octave day kept; feast transferred to the next day free of I/II class (paschal weekdays after the octave are IV class and do not block — verified "FERIA IV Cl" in the LMS Ordo) | St Mark 2027-04-25 (3rd Sunday of Easter) → Mon 2027-04-26 (verify) |
| Annunciation impeded by Holy Week | Transferred past the octave | 2029-03-25 (Palm Sunday) → Mon 2029-04-09, the first free day after Low Sunday (2024 precedent: Annunciation on Monday of Holy Week → Monday after Low Sunday) |

Two items to re-verify at implementation from the Code/Tabula text directly (flagged): the
"feast of the Lord beats a II class Sunday" clause (it is explicit in the *current* rite — GNLY
Table II.5 and the Notitiae 10 (1974) 40 reply — and the 2025 Lateran outcome shows 1962 agrees),
and the 2030 corner where **Holy Family (Jan 13) coincides with the fixed Baptism of Our Lord
(Jan 13)** — expected: Holy Family kept, Baptism commemorated.

### 2.3 `rc-1962` — sanctorale corrections (all verified against the FIUV Ordo's universal 1962 calendar)

The current table has one outright anachronism and several gaps among I/II class feasts:

| Date | Currently | Correct (1962) | Rank |
|---|---|---|---|
| Jan 13 | — | **Baptism of Our Lord** | II |
| Feb 2 | — | **Purification of the BVM (Candlemas)** | II |
| Feb 24 | — | **St Matthias, Apostle** (1962 date, not May 14) | II |
| Mar 19 | — | **St Joseph, Spouse of the BVM** (verified I class, FIUV Ordo: "S. Ioseph … Conf. et Ecclesiae universae Patroni, I cl.") | I |
| **May 1** | SS Philip & James ❌ (pre-1955 date) | **St Joseph the Worker** | I |
| May 11 | — | **SS Philip and James, Apostles** | II |
| Jul 1 | — | **Most Precious Blood** | I |
| Aug 6 | — | **Transfiguration of Our Lord** | II |
| Aug 24 | — | **St Bartholomew, Apostle** | II |
| Sep 14 | — | **Exaltation of the Holy Cross** | II |
| (1,28), (3,23) | junk "skip" rows | delete | — |

Keep the existing III class entries as they are; the 1962 universal calendar keeps III class
feasts optional — our curated list is fine.

### 2.4 `rc-current` — numbering & precedence (from the Universal Norms, verified)

- **Numbering**: the Baptism of the Lord is the **1st** Sunday of Ordinary Time (GNLY n. 38, 6);
  the early segment numbers 2nd … upward until Ash Wednesday; the late segment numbers **backward
  from the 34th = Christ the King** (n. 6.4), skipping numbers whose Sunday is superseded by a
  solemnity (2026: the 8th is taken by Trinity's Sunday, the 31st by All Saints). Total is ≤ 34.
- **Easter Sundays III–VII** (n. 23): add them (2026: Apr 19 … May 17).
- **Precedence** (Table n. 59 + nn. 5, 60):
  - Sundays of Advent/Lent/Easter (I.2) outrank **all** solemnities/feasts; an impeded solemnity
    goes to the **preceding Saturday** (n. 5) — 2028: St Joseph (Mar 19 = Lent IV Sunday) → **Sat
    Mar 18**; 2030: Immaculate Conception (Dec 8 = Advent II) → **Sat Dec 7** *per the universal
    norm* (note in the description that some conferences, e.g. the US, keep it Monday Dec 9 for
    the obligation).
  - Ordinary/Christmas Sundays (II.6) yield to **General solemnities** (I.3) — All Saints 2026,
    Assumption 2027, St John Baptist 2029 — and to **feasts of the Lord in the General Calendar**
    (II.5: Presentation, Transfiguration, Holy Cross, Lateran Dedication — explicitly confirmed by
    the Notitiae reply printed with the Table), but **outrank saints' feasts** (II.7), which are
    omitted that year (n. 60: "other celebrations are omitted") — Sts Simon & Jude 2029-10-28,
    Visitation 2028-07-02, St Luke 2026-10-18 (feast, noted as omitted in the Sunday's
    description).
  - Annunciation impeded by Holy Week/Easter octave (2029-03-25 = Palm Sunday): transfer — the
    US/typical disposition is the Monday after the Second Sunday of Easter (**2029-04-09**);
    verify against the USCCB/romcal 2029 calendar at implementation.
- Fix `_first_sunday_after` to be strictly "after" (Epiphany-Sunday 2030 → Baptism Jan 13, Holy
  Family Jan 13 — 1962 — and the Anglican 1st-after-Epiphany all currently land on Jan 6).

### 2.5 `anglican` — align to Common Worship

The set's existing shape is already CW, not 1662 BCP (Christ the King "Sunday next before Advent",
All Saints' Sunday, Visitation May 31), so CW is the scheme to follow. Add, with CW's names:
Sundays of Epiphany (Baptism of Christ = 1st; 2nd–4th as fit); up to three
**Sundays before Lent** ("Third/Second Sunday before Lent", "Sunday next before Lent"); **Lent
Sundays 1–5** + Palm Sunday; **Sundays of Easter 2–7**; **Sundays after Trinity 1–25** (numbering:
Trinity = 1st, the last is "Sunday next before Advent"); fix All Saints' Sunday (CW: observed on
the Sunday following Nov 1 when Nov 1 is a weekday — the current code moves it *forward* when Nov 1
is itself a Sunday, which is wrong: Nov 1 Sunday = All Saints' Day itself).
Precedence (CW "Rules"): Principal Feasts outrank Sundays; Festivals falling on a Sunday transfer
to the Monday following (with stated exceptions) — encode the short rules list and verify each
year's names against the CofE calendar. In-window collisions to handle: Visitation 2030-05-31
(Fri, ok), St Mary 2027-08-15 (Sun → CW keeps the BVM feast on the Sunday? — verify rule), St John
Baptist 2027-06-24 (Thu), Michaelmas 2030-09-29 (Sun).

### 2.6 `orthodox` — named Sundays and numbering (Julian reckoning, civil-converted as today)

Add: the **five named Sundays of Great Lent** (1st Triumph of Orthodoxy; 2nd St Gregory Palamas;
3rd Veneration of the Cross; 4th St John Climacus; 5th St Mary of Egypt) at `pascha − 42 … − 7`
days; the **Sundays after Pascha**: Antipascha/St Thomas (+7), Myrrhbearers (+14), Paralytic
(+21), Samaritan (+28), Blind (+35), **Fathers of Nicaea (+42 = the 7th Sunday)**, and the
existing Pentecost (+49)/All Saints (+56) stay; then **Sundays after Pentecost numbered 2nd …
as far as they run** before the Nativity-cycle Sundays (the Holy Forefathers and Holy Fathers
Sundays) interleave in December — verify the tail per year against
oca.org/goarch.org pages.
Precedence, kept light (stated in descriptions, not a full Typikon engine): a fixed **Great Feast
falling on one of these Sundays is kept and the Sunday noted** in the description (e.g. a Great
Feast displaces the numbered Sunday). The 12 Great Feasts never mutually collide with Pascha-cycle
dominicals in ways that need more than a note for a day-level calendar.

---

## 3. Design

### 3.1 A per-day resolver instead of independent `add()` calls

Both RC sets grow a small occurrence engine inside the generator (pure stdlib):

1. Each set builder constructs a list of `LitDay(date, rank, kind, summary, slug, color, note)`
   where `rank` is the rite's ladder (§2.2 / §2.4), `kind` distinguishes {sunday, feast, feria,
   ember, octave_day} and carries `feast_of_the_lord: bool` for 1962.
2. A `resolve(year)` pass groups by date and applies the rite's table: the winner becomes the
   day's `Event`; losers become a trailing clause in the winner's DESCRIPTION
   ("Commemorated: St Luke, Evangelist." / "The 23rd Sunday after Pentecost is commemorated." /
   "Sts Simon and Jude are omitted this year (the Sunday takes precedence)." / "Transferred from
   18 Oct (21st Sunday after Pentecost)."). Transferred feasts are emitted **on the transferred
   date** (that is what an Ordo prints) via the same resolver run over the moved entries.
3. `write_set` stays as-is (UID dedupe, date sort, folding) — one event per day now holds by
   construction.

### 3.2 UID / refresh discipline

- Keep the scheme `hol-{set}-{slug}-{date}@omnical`.
- New slugs: 1962 Sundays `epiphany-{n}` (incl. the resumed ones — the slug is the number, the
  SUMMARY says "Nth Sunday after Epiphany" or "Nth Sunday after Epiphany (resumed after the
  23rd after Pentecost)" per the Ordo phrasing), `pentecost-{n}` (2–24); OF Sundays keep `ot-{n}`
  with the **corrected** number (so 2026-11-15 becomes `ot-33-20261115`; the old `ot-34-20261115`
  disappears) and `ot-early-*` retires into `ot-{n}`. Renumbered UIDs churn once on refresh —
  acceptable (the linked-platforms refresh diffs by UID: adds + removes), and unavoidable for
  correctness; keep every *unchanged* event's UID byte-identical to limit the diff.
- Transferred feasts: the UID's date is the transferred date (the event exists where it is kept).
- The deployment story is already complete: `deploy.sh:155` stages `holidays/` to the router; a
  user's `holidays://{set}` source row refreshes by UID (§17.8.3 engine). No code changes needed.

### 3.3 Self-test additions (extend the existing `checks` list)

Verified-today vectors (from the Ordines) go in as hard anchors, e.g.:

- `("21st after Pentecost 2026", sundays_1962(2026)[datetime.date(2026,10,18)], 21)`
- 2026-11-08 = "5th after Epiphany (resumed)", 2026-11-15 = 6th, 2026-11-22 = 24th & last
- 2026-11-01 winner = All Saints (I), loser (23rd Sunday) commemorated
- OF: 2026-10-11 = 28th; 2026-11-22 = Christ the King = 34th; Easter V 2026 = May 3
- The feast-of-the-Lord pair as a regression test of the resolver: a 1962 II-cl feast **of the
  Lord** on a II-cl Sunday is kept (the 2025-11-09 Lateran model, LMS 2024–25), while a II-cl
  **saint's** feast on the same kind of Sunday is only commemorated (2026-10-18, verified).

And the per-year Sunday-sequence assertions for 2027–2030 from §2.1 — **each must be confirmed
against that year's published Ordo (or Divinum Officium's rendered calendar) before the anchor is
committed**; if a source contradicts the algorithm, fix the algorithm and re-derive all five
years (do not special-case).

## 3.4 Phase breakdown (each phase regenerates, self-tests, and commits separately)

1. **rc-1962 Sundays** — the numbering algorithm (§2.1) + Sundays after Epiphany + anchors for
   all five years. Highest-value, zero-precedence-risk (pure additions plus the already-correct
   temporal).
2. **rc-1962 occurrence + sanctorale corrections** — the resolver (§2.2), the §2.3 table fixes,
   and the collision vectors. This changes St Luke 2026 et al. into "Sunday + commemoration".
3. **rc-current fixes** — numbering off-by-ones, Easter III–VII, collision handling, the two
   Saturday transfers (2028 St Joseph, 2030 IC), the 2029 Annunciation transfer, and the
   `_first_sunday_after` fix (also unblocks 2030 for the other sets).
4. **anglican** — CW Sundays + rules.
5. **orthodox** — named Sundays + numbering + notes.
6. **Regeneration & deploy** — `python3 scripts/generate-holiday-ics.py`, review the full ICS
   diff, commit, deploy, refresh a live calendar that carries `holidays://rc-1962` and verify the
   Sundays appear and the St Luke day reads correctly.

## 3.5 Gates

```sh
cd ~/router-dav
python3 scripts/generate-holiday-ics.py     # self-test must pass (incl. new anchors)
git diff --stat holidays/                   # review the churn; unchanged years should be stable
cd rustical && cargo test --workspace      # must stay green (frontend tests use synthetic ICS)
```

Manual: stage on the router via `deploy.sh`, import or refresh `rc-1962` into a scratch calendar,
and check: Sun 2026-10-18 shows "21st Sunday after Pentecost" with "Commemorated: St Luke" in the
description; Nov 2026 shows the resumed 5th/6th Sundays and the 24th & Last; `rc-current` shows
Oct 11 2026 as the 28th Sunday and no second event on All Saints.

## 4. Risks & non-goals

- **Numbering-rule residual risk**: the "resumed Sundays assigned from the end" reading (§2.1.4) is
  empirically pinned by two independent Ordines for 2026, but the implementer must confirm it
  against the 1960 Code text / each year's Ordo for 2027–2030 — the anchors are the safety net.
- **UID churn** on the renumbered `ot-*` events: one visible add/remove wave per refreshed
  calendar; intended.
- **National calendars are out of scope** (Epiphany/Ascension/Corpus Christi transferred to Sunday
  in many countries — the descriptions already say so; IC 2030 notes the US Monday variant).
- **Non-goals**: full sanctorale for either rite, local/diocesan feasts, vigils as separate events
  (already noted in descriptions), concurrence (Vespers) logic beyond what the descriptions state,
  any change to the rustical import/refresh engine, and any network access at generation time.
