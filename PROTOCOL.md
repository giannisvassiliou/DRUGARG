# Held-out set 2: protocol (write once, before any run)

**Purpose.** Test drugarg 0.9.4 on queries that played no part in designing it. Held-out set 1 (20 queries, run on 0.9.3) motivated the 0.9.4 fixes, so it is now development evidence. It stays in the paper with its 0.9.3 results.

## 1. Sources (the queries come from these, not from the authors)

- **P** — Phansalkar S. et al. *High-priority drug–drug interactions for use in electronic health records.* JAMIA 2012;19(5):735–43, Table 2 (15 interactions, expert panel).
- **B5** — American Geriatrics Society 2023 Beers Criteria, Table 5 (drug–drug interactions to avoid in older adults).
- **B3** — American Geriatrics Society 2023 Beers Criteria, Table 3 (drug–disease / drug–syndrome interactions).

Neither source mentions drugarg, PrimeKG or DDInter. Both were chosen before any held-out-2 query existed. They are the best-known expert lists for exactly the two kinds of question drugarg answers: an interaction with a co-medication, and a contraindication from a condition.

## 2. Unit and inclusion

A unit is one row of a source table. If a row names several distinct interacting drugs or conditions, each one is a separate unit (e.g. warfarin + amiodarone / ciprofloxacin / macrolides / TMP-SMX / SSRIs = 5 units). **Every eligible unit is used: there is no sampling,** so no seed or choice is involved.

Units are excluded only by these rules, fixed in advance:

| Rule | Excluded units |
|---|---|
| E1. Drug class defined only by an external list the source does not reproduce | P8 (QT + QT, torsades.org list); B5 anticholinergic + anticholinergic; B3 delirium / dementia / LUTS–BPH anticholinergics (Table 7) |
| E2. Criterion is a combination with no specified pair | B5 "three or more CNS-active drugs" |
| E3. Population defined by sex, which drugarg cannot express | B5 alpha-1 blocker + loop diuretic (older women); B3 urinary incontinence in women |
| E4. Condition is a history, not a diagnosis | B3 history of falls or fractures |
| E5. Query identical to a development or held-out-1 query | P3 febuxostat + azathioprine (= dev `gout/azathioprine`) |
| Merge. Two units giving an identical query become one query | P4 SSRIs + MAOI and P7 TCAs + MAOI → `heldout2/p04` |

Result: **47 queries** (36 testable after the pre-freeze resolution check in section 5) (Phansalkar 12, Beers Table 5: 13, Beers Table 3: 22). Two are near-duplicates of development queries; they are flagged `near_duplicate_of_dev` and reported separately.

## 3. Turning a unit into a query (mechanical)

- **Candidate side.** For a drug pair, the candidates are the drugs for the main indication of the *first-listed* class (Phansalkar: drug A; Beers Table 5: object drug). For a drug–disease row, they are the drugs for the main indication of the listed drug class, and the patient has the listed disease.
- **disease_text** = the main indication of that class (e.g. triptans → migraine; theophylline → asthma). This is the one remaining judgement call. Check each against the drug's DrugBank indication text with `drugarg primekg-text` before freezing, and change it only if the text names a different first indication.
- **Co-medication** = the first drug the source names for the interacting class. If the source names only a class, take the alphabetically first approved member (ACE inhibitors → benazepril; benzodiazepines → alprazolam; loop diuretics → bumetanide; macrolides except azithromycin → clarithromycin; SSRIs → citalopram).
- **Condition.** A curated condition key where one exists (`c_hf`, `c_ulcerhx`), plus the PrimeKG disease name (`patient_disease`) so that KG contraindication edges also fire. Otherwise `patient_disease` only (syncope, delirium, dementia, Parkinson disease, chronic kidney disease).
- **No `exclude_text`, no `exclude_drugs`.** Indication noise is reported as found, as in held-out set 1.
- **expect** = the source's recommendation, paraphrased. Nothing is added from our own knowledge.

## 4. Scoring (fixed now)

For each drug the source names that appears among the candidates:

- **hit**: rejected; or, if the source says "avoid or monitor" / "avoid when possible" / "use with caution", accepted only with a mitigation that answers an effective objection;
- **deferred**: `review` (reported separately, never counted as a hit);
- **miss**: accepted or "pick one" with no objection, or with an objection answered when the source says plain "avoid".

A query is **met** if every named drug present is a hit, **partly met** if at least one is, **not met** if none is, and **not testable** if no named drug is among the candidates. The headline number is drug-level recall (hits / named drugs present), with the query-level counts beside it.

What this does **not** measure: whether drugarg wrongly rejects safe drugs. The sources only list what to avoid, so precision needs the clinician labels (Job 2).

## 5. Freeze checklist

1. Finish the development runs (final5, four settings) and the held-out-1 runs on 0.9.4. Only crash or bug fixes are allowed until step 4.
2. `drugarg check-queries heldout2.json --primekg primekg.sqlite`. Fix only names that do not resolve (a co-medication spelling, a `patient_disease` that matches no node, a disease_text with 0 nodes), using `primekg-find` / `primekg-text`. Log every change in the table below. This step shows counts, not verdicts.
3. Check each disease_text against `primekg-text` (rule in §3) and log any change.
4. **Freeze.** Record the SHA-256 of the drugarg zip, `heldout2.json` and this file (`certutil -hashfile FILE SHA256` on Windows). Send the three files and their hashes to a co-author or a dated archive (OSF, Zenodo, or e-mail) so the time is on record.
5. Run once, all four settings, with `--ddinter-fetch` on the first:
   `drugarg batch heldout2.json --primekg primekg.sqlite --ddinter ddinter.sqlite --ddinter-fetch --management-mitigates-major --untyped-ddi ddinter-silent -o heldout2_main`
   (then `_conservative` without `--management-mitigates-major`, `_strict`, `_lenient`).
6. Score by §4. No code, rule or query change after step 4. Anything learned goes into a later version, reported separately.

### Resolution rules (fixed before applying them)

- **R1.** If PrimeKG names the disease differently, use the shortest node name that contains the source's key word *and names the same condition*.
- **R1b.** If no node names the same condition, use the node that names its direct parent, when one exists (venous thrombosis → "thrombotic disease"). Never use a sibling (e.g. coronary thrombosis).
- **R2.** If there is no such node, keep the query unchanged and mark it `not_testable`. It is still listed and run, but excluded from recall, and reported as a gap in the KG.

### Pre-freeze change log

| query | field | before | after | reason (rule) |
|---|---|---|---|---|
| p02, p12 | disease_text | HIV infection | HIV infectious disease | R1 |
| b3-hf1 | disease_text | intermittent claudication | intermittent vascular claudication | R1 |
| b5-12a–e | disease_text | venous thrombosis | thrombotic disease | R1b (nodes with "thrombosis" are coronary, hepatic-vein, carotid and intracranial, all siblings) |
| b3-hf2 | — | — | not_testable | R2: no pseudobulbar node |
| b3-sy1–sy4 | — | — | not_testable | R2: no syncope / fainting node |
| b3-pd1 | — | — | not_testable | R2: no nausea / vomiting node |
| p13 | — | — | not_testable | R2: "spasticity" matches only X-linked parkinsonism-spasticity and spasticity-intellectual disability-epilepsy syndromes, not spasticity itself |
| b3-dm2–dm4 | patient_disease | dementia → dementia + Lewy body dementia | dementia (disease) only | drugarg bug fixed before freeze: exact-name match now accepts "<term> (disease)" |
| p06, b5-02, b5-03, b3-de7 | — | — | not_testable | R2: the only "pain" node is myofascial pain syndrome (check-queries passed it, but primekg-find shows it is not the same condition) |
