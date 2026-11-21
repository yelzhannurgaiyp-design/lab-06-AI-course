# REPORT — what the measurement showed, who it affects, and whose responsibility it is

Setup of the run: model `Qwen/Qwen2.5-3B-Instruct`, greedy decoding, Colab T4 GPU. `N_PAIRS = 40` pairs per
attribute, `SEED = 2026`: 200 pairs, 400 profiles, 400 model calls for the bare run (468 s).

## 1. Measurement read-out

**What was compared.** Each pair is two invented applicants with identical merit fields. Only one thing
differs: the language of the name (`name_language`), the gender coding of the name (`gender_coded_name`) or the
tier of the university (`university`). Two null groups give the floors: `null_name` (another name of the same
language and gender coding) for the two name attributes, and `null_university` (another university of the same
tier) for `university`.

**Overall.** The bare agent shortlisted 356 of 400 profiles (89 %) and rejected 44. All 400 replies were
parseable. Rejected scores ran from 20 to 45, shortlisted from 45 to 85 (median 75). The two decisions overlap
only in the 40–49 band (33 rejected, 31 shortlisted).

**Flips: pairs whose two members got different outcomes (bare agent).**

| attribute | pairs | flips | flip rate [95% CI] | pairs with different scores | mean \|diff\| |
|---|---|---|---|---|---|
| name_language | 40 | 2 | 5.0 % [1.4 %, 16.5 %] | 11 | 3.12 |
| gender_coded_name | 40 | 4 | 10.0 % [4.0 %, 23.1 %] | 6 | 1.12 |
| university | 40 | 1 | 2.5 % [0.4 %, 12.9 %] | 10 | 2.50 |
| *null_name (floor)* | 40 | 0 | 0.0 % [0.0 %, 8.8 %] | 7 | 2.00 |
| *null_university (floor)* | 40 | 1 | 2.5 % [0.4 %, 12.9 %] | 9 | 2.12 |

All flips were shortlist ↔ reject; there were no other flips.

**Direction: each pair turned so the named group comes first.**

| comparison | pairs | first favoured | other favoured | McNemar p | score gap, first − other [95% CI] | sign p |
|---|---|---|---|---|---|---|
| kazakh-language name vs others | 28 | 2 | 0 | 0.500 | +0.7 [−1.8, +3.8] | 1.000 |
| russian-language name vs others | 26 | 0 | 1 | 1.000 | +0.2 [−1.7, +2.1] | 1.000 |
| english-language name vs others | 26 | 0 | 1 | 1.000 | −1.0 [−4.2, +1.9] | 1.000 |
| male-coded vs female-coded name | 40 | 0 | 4 | 0.125 | −0.4 [−1.2, +0.6] | 0.219 |
| flagship vs regional university | 40 | 1 | 0 | 1.000 | **+2.5 [+1.1, +4.1]** | **0.002** |

Shortlist rates by group: English-language names 96.2 %, Kazakh 96.4 %, Russian 92.3 %; female-coded 90.0 %,
male-coded 80.0 %; flagship 92.5 %, regional 90.0 %.

**Decisions: no attribute stands clear of its floor.** With 40 pairs, an exact McNemar test at 0.05 needs at
least 6 pairs favouring one side and none the other way (15 %), or 8 with one the other way, or 10 with two.

- `name_language`: 2/40 flips against 0/40 for `null_name`. The intervals overlap. No direction (p ≥ 0.5).
- `gender_coded_name`: 4/40 flips against 0/40. All four favoured the female-coded name (McNemar p = 0.125).
  That is below the 6 the test needs, and the interval [4.0 %, 23.1 %] overlaps the floor's [0 %, 8.8 %]. I
  **cannot** claim a gender effect on decisions. I also cannot exclude one: a real effect of 4–5 pairs in 40
  would look exactly like this.
- `university`: 1/40 flips, the same as its floor `null_university` (1/40). No effect on decisions.

**Scores: one effect stands clear.** The university tier moves the *score*. Flagship minus regional is
+2.5 points [+1.1, +4.1]: the interval excludes zero, and all 10 pairs whose scores differed favoured the
flagship university (sign test p = 0.002). This stays significant if I correct for testing five comparisons
(0.05 / 5 = 0.01). It cannot be explained by the floor: in `null_university`, 9 pairs also had different scores
(mean |diff| 2.12), but there both universities are of the same tier, so the floor has no direction. The
university effect is therefore not the size of the score change (2.50 vs 2.12 mean |diff|) but its **sign**:
it always goes against the regional university. It almost never changes the decision, because most scores lie
far from the boundary (median 75, boundary about 45). The other score gaps are all within ±3 points and their
intervals include zero (half-widths about ±0.9 for gender, ±1.9 to ±3.0 for name language).

**How much could be noise.** Greedy decoding gives the same reply to the same text, so there is no run-to-run
noise. The floors measure something else: how much the output moves when an irrelevant detail of the *same
kind* changes. They are not zero (7 and 9 pairs with different scores, 0 and 1 flips) because a 3B model
reacts to any change of tokens. The floors keep the layout and the merit fixed and change one line. They do
not test other prompt wordings or another order of the fields. Part 3 shows the
same thing from the other side: removing two lines changed 166 of 400 scores (mean |Δ| 14.0), much more than
any attribute moved them.

**What this run can support.** Statements about this model, this prompt and these invented names and
universities only. A different wording, a larger model, or real names in Cyrillic or Kazakh script could
behave differently. A stronger test needs more pairs (the interval width shrinks with √n), several prompt
wordings and several orders.

## 2. Who is affected

The fund has a limited number of places, so each decision has two sides.

- **Applicants from regional universities.** They lose 2.5 points on average against an identical applicant
  from a flagship university. In this run it changed one decision, because almost everybody scored far from
  the boundary. For an applicant near the boundary (scores 40–49, where rejects and shortlists mix) 2.5 points
  can be the difference. If the fund raised its bar or had fewer places, more of them would fall there.
- **Wrongly rejected applicants.** A flipped pair means one of two identical applicants lost a place their
  twin got. In this run that happened to 7 applicants in the three real attributes (2 name-language, 4
  gender, 1 university pairs). For gender, all four losers had the male-coded name, but four is too few to call
  it a pattern. The cost is the largest in the system: a year or more of scholarship money, maybe the decision
  to study at all for a low-income applicant (income bracket 1–2), and no way to know the reason. A rejection
  is rarely appealed, so the error usually stands.
- **Wrongly shortlisted applicants** cost less to themselves, but each one takes a place from someone more
  suitable, who never sees why.
- **Applicants with names or universities the model has never seen** (rare transliterations, small regional
  institutions). The floors show the model moves on any unfamiliar token; these applicants carry that risk.
- **People who never apply** because they expect a regional university or a "wrong" name to count against
  them. They do not appear in any log.
- **Reviewers.** With the harness, 10 of 400 applicants (2.5 %) go to a person. That is a light workload, but it
  also means 97.5 % of decisions are never re-checked.
- **The fund**, which carries the legal and reputational risk of a discriminatory or unexplainable decision.

**Effects below what 40 pairs can see.** A real one-sided effect of 4 pairs in 40 (10 %), like the gender
result, is invisible to this test. Over 1,000 applicants a year that would be about 100 people a year, always
from the same group. The −2.5-point university shift is real but small; at scale it would touch every regional
applicant near the boundary, every year.

**Compounding.** Used year after year, a small tilt repeats on every cohort. If the shortlist feeds a later
stage (an interview, another fund, a list of scholars), the first decision becomes "evidence" for the next one:
a regional-university applicant who is rejected once has a weaker record next time.

## 3. Professional responsibility

**My responsibilities as the developer of the harness.**

1. *Measure before it runs.* I ran the matched-pair test with floors and a detectability check. It holds for
   this model and prompt only and must be re-run whenever the model, the prompt or the blinded fields change.
2. *Do not over-claim.* I must not tell the fund that the harness "removes bias". After blinding, the pairs show
   0 flips only because both twins get identical text; that is true by construction. What I can say: the model
   no longer reads the name or the university; 32 of 400 decisions and 166 of 400 scores changed when those two
   lines were removed; 2.5 % of applicants now go to a person.
3. *Report what was found, including what is uncomfortable.* The university score shift is real (p = 0.002),
   and the gender result is undecided, not "no effect".
4. *Document the limits.* Age is still shown and was not tested. The band was calibrated on 100 invented
   held-out profiles, not on real applicants.
5. *Test the refusals.* All six requests in Part 4 were run: the harness refused all 5 bad ones with 0 model
   calls and wrongly refused 0 of the 400 clean applicants.

What is left: more pairs and prompt wordings, an age attribute, a retention and access policy for the log, a
real appeal procedure, and a way to log the reviewer's own decision.

**The institution's responsibilities.** The fund sets the purpose and the criteria, chooses and trains the
reviewers, answers objections within the legal deadline (three working days, Personal Data Law Art. 19-1(3) —
see JUSTIFICATION.md §4), tells applicants that an automated system is used, and keeps the log safe.

**Where we conflict.** If the fund wants the university shown again "for context", I should show the +2.5
point shift and refuse to sign off on that configuration without a new measurement. If the fund wants a wider
band but has no reviewers, I should show what each band costs (the calibration table) instead of quietly
choosing one. If it insists on something I measured as harmful, I document my objection in writing: a developer
who has measured a risk cannot pretend not to know it.
