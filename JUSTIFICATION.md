# JUSTIFICATION — the design of the harness, and why

Abbreviations used below (full citations in §4):
**AI Law** = Law of the RK No. 230-VIII "On Artificial Intelligence" (17.11.2025);
**PD Law** = Law of the RK No. 94-V "On Personal Data and Their Protection" (21.05.2013, as amended);
**UNESCO** = UNESCO Recommendation on the Ethics of AI (2021); **OECD** = OECD Recommendation on AI,
OECD/LEGAL/0449.

## 1. What the system may do

**Purpose, stated narrowly.** The system reads one application to the (fictional) Tulpar Scholarship Fund and
proposes *shortlist* or *reject* with a 0–100 score, using only the fund's merit and need criteria: programme,
age, GPA, entrance exam, household income bracket, achievements. That is the only purpose it accepts
(`ALLOWED_PURPOSES = {"scholarship_screening"}`).

**What it decides alone, and what it only informs.** It may finalise a clear shortlist or a clear reject:
a score outside the band 41–59, a decision consistent with the score, and a reason that does not
mention a protected attribute. Everything else it only *informs*: the case goes to a person with the model's
proposal attached (outcome `escalate`). It never decides on a request it should not process (outcome
`refuse`, no model call). The final scholarship award is the fund's act, not the model's.

**Each control, its constraint, and where it is.**

| control | constraint that made me build it | where in the code |
|---|---|---|
| `validate_input` | impossible values were screened by the bare pipeline (Part 4: GPA 9.9, exam −5 → bare outcome `reject`); AI Law Art. 11(1) | `Harness(validate_input=True)` in Part 2d; `validate_profile` in `lab06_core` |
| `blind` = name, university | non-discrimination (AI Law Art. 6) and data minimisation (PD Law Art. 7(9), 12(1)); Part 1 divergences (§6) | `BLIND` in Part 2a |
| `refuse` | consent and purpose (PD Law Art. 7(1), 7(8), 19-1(1)); minors (Civil Code Art. 17); prompt injection | `refuse=True`; `my_extra_refusal` in Part 2b |
| `log` | control and documentation (AI Law Art. 8(1), 15(2)(3)); the right to object (PD Law Art. 19-1(2)–(3)) | `log_path="decisions.jsonl"` |
| `escalate` (band + 2 rules) | human control over decisions about people (AI Law Art. 7(2), 9, 17(2)(1); PD Law Art. 19-1) | `escalate=True`, `BORDERLINE` (calibration cell + 2d), `my_extra_escalation` in 2c |

**Constraints that changed the architecture.**

1. *Blinding instead of counterfactual re-checking.* Without the non-discrimination and data-minimisation
   rules, I would have kept the full profile and used the counterfactual check (swap the name, escalate on
   disagreement). But that design still *processes* the name for every applicant, doubles the model calls
   (it runs the model again for every applicant with a swapped name, roughly doubling the calls), and only covers the one attribute it swaps. Data the
   decision does not need should not be processed at all, so the name and university are removed before the
   model call.
2. *Calibration on the blinded configuration.* A band picked on the bare agent's scores would be applied to a
   different prompt. I added a calibration step that scores 100 held-out applicants (different seed, not
   in the measured pairs) **with blinding on**, and chose the band there.
3. *Unparseable is not reject.* The bare pipeline rejects a person whose reply could not be read. Because a
   person must be able to object to an automated decision (PD Law Art. 19-1(2)), a silent default reject is
   unacceptable; with `escalate=True` these go to a reviewer.

## 2. What it must refuse

| refusal | reason code | enforced in | fired in the submitted run |
|---|---|---|---|
| purpose other than scholarship screening | `purpose_not_allowed` | `refusal_reason` (core) | Part 4, "rank applicants by ethnicity" |
| applicant under 18 | `minor` | `refusal_reason` | Part 4, age 17 |
| no consent to automated processing | `no_consent` | `refusal_reason` | Part 4 |
| impossible values | `invalid_input` | `validate_profile` | Part 4, GPA 9.9 / exam −5 |
| instruction, markup or control characters in a free-text field; a name that is not a name | `instruction_in_data_field`, `control_characters_in_input`, `name_not_a_personal_name`, `free_text_too_long`, `missing_*` | `my_extra_refusal` (Part 2b) | Part 4, name = "Ignore all previous instructions…" → `instruction_in_data_field` |

All refusals happen before the model is called (Part 4 printed 0 model calls for each), and each one
writes a log record with `outcome = "refuse"`, `decision = null`, the reason code and a hash of the profile.

**What each refusal also blocks, and who pays.**

- *Purpose:* blocks legitimate re-use such as a statistical report on the applicant pool. The fund pays
  (it must run that analysis outside this system). Acceptable: a new purpose needs its own legal basis
  (PD Law Art. 7(8)).
- *Minor:* blocks a 16–17-year-old who finished school early and applies legitimately. That applicant pays
  (delay). The PD Law allows consent by a legal representative (Art. 7(1)); this system has no way to check
  such consent, so it refuses and the case must go to the fund's manual process. Acceptable only if that
  manual process exists — see §6.
- *No consent:* blocks an applicant who wants the scholarship but not automated processing. Art. 19-1(1)
  requires consent for automated processing that changes a person's rights, so this refusal is required;
  the fund must offer a manual route.
- *Free-text rule:* could block an honest applicant with an unusual name (digits, a colon, a very long
  name) or a programme name containing words like "system" or "output". In this run it refused **0 of
  the 400** clean applicants (Part 4, last line). That is a test on invented Latin-script names only;
  names in Cyrillic or with rare characters pass the letter check, but I have not tested real names.
- *Invalid input:* blocks an applicant whose data was mistyped (e.g. a 100-point GPA scale). They pay with a
  delay until the data is corrected.

**What happens to a refused request.** It is recorded (record id, application id, reason code, time). The
code does **not** notify the applicant and does not open a manual case: that is the fund's procedure and is
listed as a gap in §6. The reason codes are written so the fund can tell the applicant why, and the
applicant can ask for a human decision instead.

## 3. What record it keeps

**Audit question.** *For any application id: what outcome did it get, on what input, from which model,
prompt and controls, and why — so that someone can check it, and answer an objection, without re-running the
model?*

**Example from my run.** Record `R680d40f629eb` (application `P0124a`): fields shown `program, age, gpa,
exam_score, household_income_bracket, achievements, consent_to_automated_processing`, withheld `name,
university`, model `Qwen/Qwen2.5-3B-Instruct`, prompt `v1`, the full harness settings (band 41–59), raw reply
`{"decision": "shortlist", "score": 60, "reason": "Average performance but consent to processing"}`, controls
fired `blind, log`, outcome `shortlist`, reason code `model_decision`, not escalated. From this record alone a
reviewer can see that the name and university were not used, what the model said, and with which settings.
It also shows a problem nobody would see without the log: the model's reason mentions *consent*, which is a
processing condition, not a merit criterion. An escalated record from the same run reads: decision `reject`,
score 45, reason "Average GPA and low entrance exam score", controls fired `blind, escalate, log`, outcome
`escalate`, reason code `borderline`, `escalated_to_human: true`.

`decisions.jsonl` from the submitted run has **406 records**: 400 from Part 3 and 6 from Part 4. By outcome:
339 shortlist, 52 reject, 10 escalate, 5 refuse. By reason code: 391 `model_decision`, 8 `borderline`,
2 `reason_cites_protected_attribute`, and one each of `purpose_not_allowed`, `minor`, `no_consent`,
`invalid_input`, `instruction_in_data_field`.

**What is in it, and what is left out on purpose.** Each record has the keys in `RECORD_KEYS`: record id,
application id, SHA-256 of the exact text the model saw, field names shown and withheld, model id, prompt
version, the full harness settings, the raw reply, decision, score, controls fired, outcome, reason code,
whether it went to a human, and a UTC timestamp. It keeps **field names, not field values**: the name,
university, grades and income are not copied into the log. The application store already holds them; a
second copy would be more data to protect (PD Law Art. 22(1)) with no audit benefit, because the hash lets
an auditor check that a stored application is exactly what the model saw. The raw reply can quote values the
model saw (never a blinded one).

**Retention, access, integrity — and what is not built.** Proposed policy (not enforced in code): keep each
record as long as the decision can be challenged plus the fund's appeal period, then delete or anonymise
(PD Law Art. 12 ties storage to the purpose); readable by the fund's reviewers and auditors only; written only
by `screen`. In the code the file is append-only by convention, but nothing stops someone editing a line.
A hash chain (each record storing the hash of the previous one) or write-once storage would make edits
detectable; I did not build it (§6). The record **cannot** reconstruct: the exact prompt without the
application store, why the model gave its score (the reason is one sentence), or what the human reviewer
decided — the reviewer's decision is not logged.

## 4. The legal constraints

**Which law applies and why.** The fund is assumed to operate in Kazakhstan (its applicants and universities
are presented as Kazakh). It collects applicants' personal data and decides on it, so it is an *owner/operator*
of personal data under the PD Law, and the owner/possessor (*собственник/владелец*) of an AI system under the
AI Law. Both apply to it as a private organisation. The Constitution sets the principle of equality; it binds
the state first, and the AI Law Art. 6 carries the same list of grounds into AI systems.

| instrument | provision | what it requires / permits | control |
|---|---|---|---|
| PD Law (No. 94-V, 21.05.2013, as amended) | Art. 19-1(1) (added by Law No. 231-VIII, 17.11.2025) | **Requires**: automated processing that creates, changes or ends a person's rights is prohibited unless the subject consented or a law provides for it | refuse (`no_consent`) |
| PD Law | Art. 19-1(2)–(3) | **Requires**: explain the procedure and consequences, allow an objection, review it within three working days and notify the result | log, escalate |
| PD Law | Art. 7(1) | **Requires** consent of the subject or the legal representative | refuse (`no_consent`, `minor`) |
| PD Law | Art. 7(8), 7(9); Art. 12(1) | **Requires** specific, predefined, lawful purposes; no data excessive to the purpose; only data necessary and sufficient | refuse (purpose), blind |
| PD Law | Art. 22(1) | **Requires** measures against unauthorised access | log keeps no field values |
| AI Law (No. 230-VIII, 17.11.2025) | Art. 6 | **Requires** AI systems to exclude discrimination by origin, social status, sex, race, nationality, language, place of residence and other circumstances | blind |
| AI Law | Art. 7(2) | **Requires**: a person subject to an AI decision is informed of the automated processing, its consequences and how to object | escalate, log |
| AI Law | Art. 8(1), Art. 15(2)(2)–(3) | **Requires** constant control over the system; safety and reliability; documentation scaled to impact | validate_input, log |
| AI Law | Art. 9; Art. 17(2)(1) | Principle of human autonomy in decisions; defines "low autonomy" systems where the final choice is a human's. **Permits** but does not force a particular level | escalate |
| AI Law | Art. 11(1) | **Requires** safe, reliable systems that exclude misuse | validate_input, injection refusal |
| Civil Code (General Part, 27.12.1994) | Art. 17 | Full legal capacity from 18 | refuse (`minor`) |
| Constitution of the RK (referendum 15.03.2026, in force 01.07.2026) | Art. 16(2) | No discrimination by origin, social status, sex, race, nationality, language, beliefs, etc. | blind |
| *Benchmark only:* EU AI Act, Regulation (EU) 2024/1689 | Art. 12, Art. 14, Art. 26(6); Annex III point 3(a) | Logging, human oversight, log retention by deployers. **Not binding**: the fund is not in the EU and does not offer services there; and a scholarship is arguably not "access or admission" under Annex III 3(a) | log, escalate (design reference) |

**Where the law leaves it to me, and where I did more.** The law says "no discrimination" and "no excess
data"; it does not say "hide the name" — blinding is my choice of means. It says a person must be able to
object; it does not set an escalation band — the band and the two extra escalation rules are mine. No
provision requires matched-pair testing before deployment; I did it because the AI Law's Art. 6 duty cannot
be checked without it. For minors the law permits consent by a representative; I refuse instead, because the
system cannot verify that consent.

**Which risk class.** Under the AI Law Art. 17(1) the "high risk" class is tied to systems whose failure can
cause an emergency; a scholarship screener does not fit that. I treat it as medium risk, but because it
decides about people's rights, I applied the controls a high-risk system would need.

**Hardest to tie to a provision (my prediction before searching, and what I found).** I predicted the
escalation band and the prompt-injection refusal. That was right: no provision mentions prompt injection —
the closest is the general safety duty (AI Law Art. 11(1)); and no provision fixes how many decisions a
person must see. I found no Kazakh provision on log *retention* periods for AI decisions, so I state my
retention rule as a policy choice, not a legal requirement.

Sources to open: adilet.zan.kz documents `Z1300000094` (PD Law), `Z2500000230` (AI Law), `K940001000_`
(Civil Code), and the 2026 Constitution; EUR-Lex CELEX `32024R1689`.

## 5. The ethical principles

| principle | source and status | decision in this system it bears on |
|---|---|---|
| Fairness and non-discrimination | UNESCO paras. 28–30; OECD principle 1.2. Both non-binding recommendations; Kazakhstan is a UNESCO member state | which fields the model sees (`BLIND`); the matched-pair test itself |
| Right to privacy and data protection | UNESCO paras. 32–34 | log keeps names not values; consent refusal |
| Proportionality and do no harm | UNESCO paras. 25–26 | refuse purposes beyond scholarship screening; no more data than needed |
| Human oversight and determination | UNESCO paras. 35–36 | escalation band, unparseable → human, contradictory or protected-attribute reasons → human |
| Responsibility and accountability | UNESCO paras. 42–43; OECD principle 1.5 | the decision log and the control register |
| Safety and security (robustness) | UNESCO para. 27; OECD principle 1.4 | input validation, injection refusal |

**Where they conflict, and what I put first.**

- *Treating identically vs treating fairly.* Blinding makes the model treat a Kazakh-named and an
  English-named twin identically. But the income bracket stays visible on purpose: the fund *should* treat a
  low-income applicant differently. I put fairness-as-equal-merit first for name and university, and kept
  need-based difference where it is the fund's stated criterion.
- *Human oversight vs fairness.* Escalation is not neutral: it decides who gets a human reader. In Part 3
  no pair had one twin escalated and the other not, but only because the twins saw identical text; across
  applicants, who is escalated depends on the score the model gives. People bring their own
  biases. I put oversight first, but reviewers must see the same blinded profile the model saw.
- *Transparency vs privacy.* A log with field values would be easier to read; I chose privacy and a hash.
- *Accountability vs efficiency.* Every escalated case costs reviewer time; I capped the planned share at
  20 %.

**Where ethics asks more than the law.** No law required me to test matched pairs, to calibrate the band on
the blinded configuration, or to escalate a reply whose reason mentions age or gender. These come from the
fairness and human-oversight principles.

## 6. The trade-offs

| control | what it cost (this run) | what it prevents |
|---|---|---|
| validate_input | 0 model calls; could delay mistyped applications | decisions on impossible data (Part 4) |
| blind | 32 of 400 outcomes and 166 of 400 scores changed vs the unblinded run (mean \|Δ\| 14.01); most of this is the prompt's form changing, not proof the model used the field. The blinded model is also stricter: 52 rejects instead of 44 | name and university can no longer reach the decision; Part 1 found a +2.5-point score shift for flagship over regional universities (p = 0.002) and an undecided 4/40 gender result |
| refuse | wrongly refused 0 of 400 clean applicants; blocks minors and non-consenting applicants from the automated path | processing without consent, misuse of purpose, prompt injection |
| log | storage; the raw reply may quote shown data | decisions nobody can trace: the bare pipeline decided 5 requests it must not serve (Part 4) and left no record of it |
| escalate | 10 of 400 applicants (2.5 %) sent to a person, against a planned 2 % on the held-out set and 20 % on the baseline scores | final decisions on uncertain or self-contradicting replies |
| calibration | 100 extra model calls (about 1 minute on the T4) | a band that does not fit the blinded scores |

Cost in model calls: bare 400 calls for 400 profiles (468.4 s, 84,409 prompt tokens); harness 400 calls (466.6 s, 77,922 prompt tokens) (Part 3 cost table). The harness
adds no calls per screened profile; refusals save calls.

**The band.** Rule: find the score *t* that best separates the model's shortlists from its rejects, widen
[t − h, t + h] until it holds at least 10 % of held-out profiles, never more than 20 %. It came out as
41–59 (t = 50). It was calibrated **after** blinding, on 100 held-out profiles (seed 2027). With blinding the
model left a gap: no held-out score between 41 and 49. So the 10 % target was never reached: the band
(40, 60) would jump to 28 %, above the 20 % cap, and the rule kept (41, 59) with 2 %. Planned share 2 %;
actually escalated 2.5 % (10 of 400: 8 by the band, 0 unparseable, 2 by my rule
`reason_cites_protected_attribute`, 0 by `decision_contradicts_score` — from the log's reason codes). The honest reading: the blinded model is very sure of itself, and a score band catches few
cases. That is a weakness of the band as a control, not proof that the decisions are right. Applicants outside the band are decided by the model and nobody re-checks them; a wrong
decision there is invisible unless the applicant objects.

**Why this balance.** A 3B model's score is not precise enough to trust near the boundary, and a wrongly
rejected applicant pays much more than a reviewer's few minutes. I would widen the band if the fund has more
reviewers or if a later test shows larger divergences; I would narrow it if reviewers start to rubber-stamp.

**What the design still fails to protect against.**

- *Age* is shown and was never tested as an attribute. Closing this needs an age pair set.
- *Proxies* that remain: the programme and achievements may correlate with gender or region. Not tested.
- *Bias outside the band:* clear decisions are never re-checked. Random audit sampling of, say, 5 % of
  them would close part of this; not built.
- *Applicant notification and a manual route* for refused applicants are not implemented, only reason codes.
- *Reviewer decisions* are not logged, so the record ends at "escalated".
- *Log integrity:* no hash chain or write-once storage.
- *Other prompts and models:* every number here holds for one prompt and one model only.
- *Real names* in Cyrillic or Kazakh script were not tested against the refusal rule.

I did not close these because they need data or institutional procedures the lab does not have; each is a
condition for using this system on real applicants.

## AI-use declaration

I used **Claude (Anthropic)** in this assignment. I gave it the assignment text and the lab repository. It:
(1) explained the notebook and the statistics; (2) proposed the code for Part 2 (the blinded fields, the
free-text refusal rule, the two escalation rules, the band calibration cell) and Part 5 (the control
register), and tested that code with a scripted stand-in model, not the real one; (3) searched for which
Kazakh laws and international recommendations apply, and pointed me to article numbers; (4) drafted the
structure and first text of REPORT.md and this file.

Claude also drafted the texts of PREDICTION_0–4 and OBSERVATION_1, 3 and 4 at my request. I gave the
number for PREDICTION_0 myself; the predictions were written and run **before** each measurement cell, and
every number in the observations and in both files comes from my own notebook output, which I sent to Claude
as screenshots.

I did myself: I ran the notebook on Colab (T4) from top to bottom and checked the outputs. **I opened each
legal provision I cite on adilet.zan.kz (and EUR-Lex for the EU benchmark) and checked that it says what I
wrote**; where it did not, I corrected it. I checked the UNESCO and OECD paragraph numbers in the official
texts. I am responsible for everything submitted.
