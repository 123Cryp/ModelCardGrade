# ModelCardGrade

Structural grading of AI model cards on GenLayer, with a registry other contracts can read.

Someone registers the URL of a model card together with the model version the card is claimed to describe. GenLayer validators fetch the card and extract a fixed set of facts, each with an exact quote copied from the page: the model name, the version the card describes, and five sections (intended use, training data, limitations, evaluation results, license). The code, not the model, reads each quote and decides whether the section is present, vague or absent. The grade (A, B, C or D) is then computed by fixed rules in the contract code. No model ever writes a status or a grade, and no model output is accepted without an exact quote from the fetched page. Anyone can challenge a grade with a bond and a second document that cites the original card URL. The final grade is stored in an on-chain registry.

ModelCardGrade grades the structure of a model card, not the quality of the model. A grade A means: the card names the model, states the claimed version, and has a concrete statement for each of the five sections. It does not mean the model is good or safe, or that the statements are true.

## What the model does and what the code does

| Step | Done by |
|---|---|
| Check the URL is https, clean, and under an owner-approved prefix | code |
| Check the claimed version looks like `1.2.0`, `2` or `1.0.0-rc.1` | code |
| Fetch the card | each validator |
| Locate the model name, the version statement and one sentence per section, and copy exact quotes | model |
| Drop every quote that is not an exact substring of the page | code |
| Read the version from the quote and compare it with the claimed version | code |
| Decide present / vague / absent for each section from the words of its quote | code |
| Add up penalty points and map them to A / B / C / D | code |
| Agree on the result | validators |

"Exact substring" means after collapsing whitespace and ignoring letter case.

## Grade rules

Penalty points are added up. Lower is better.

| Section state | Points | Meaning |
|---|---|---|
| PRESENT | 0 | a concrete statement for the section |
| VAGUE | 5 | something is said, but it is generic, a pointer, too short, or lacks the required detail |
| ABSENT | 10 | no usable quote, a quote that does not speak about the section, or a statement that the information is missing |

| Version result | Points | Meaning |
|---|---|---|
| MATCH | 0 | the version quote names exactly one version and it equals the claimed version |
| MIXED | 10 | the quote names the claimed version and at least one other version ("1.2.0 (supersedes 1.1.0)") |
| UNSTATED | 15 | the card has no version quote with a version in it |
| MISMATCH | 25 | the quote names other versions only |

| Penalty | Grade |
|---|---|
| 0 to 4 | A |
| 5 to 14 | B |
| 15 to 29 | C |
| 30 or more | D |

So a single vague section or a single missing section is at best B, an unstated version is at best C, a version mismatch is at best C, and a card with three missing sections is D. `get_rules()` returns the same tables from the contract.

A card is UNGRADED when no model name is grounded, or when fewer than three of these six signals are grounded: the version, and each of the five sections. A section is a signal only if its quote is on the page and speaks about the section (it contains one of the section's words). A quote that says "not provided" counts as a signal, because it is evidence about the section. A grade attempt that finds too little is "undetermined"; after 3 undetermined attempts (or when the card times out) the card becomes UNGRADED.

### How a section quote is read

Rules apply in this order to the lower-cased quote. The first rule that fires decides.

| Order | Rule | State |
|---|---|---|
| 1 | The quote is empty, or contains none of the section's words (for example "limitation", "risk", "bias" for limitations; "train", "dataset", "data" for training data) | ABSENT, no signal |
| 2 | Negated missing words are removed first: "no details are missing", "nothing is undisclosed", "not unknown". A conjunction ("and", "but", "while", ...) stops the removal | |
| 3 | An explicit absence: `n/a`, `tbd`, `undisclosed`, `unspecified`, `not applicable`, "not provided / available / disclosed / documented / specified / released / evaluated / licensed / publicly known", "no information", "no known limitations", "no license", "without a license"; and, in quotes of at most 8 words, `unknown`, `missing`, `empty`, `none`, `nothing` | ABSENT |
| 4 | A vague marker: `various`, `diverse`, `wide range`, `general purpose`, `etc`, `see the paper`, `coming soon`, `strong performance`, "some limitations", "publicly available data", ... | VAGUE |
| 5 | Fewer words than the section needs (5 words; 2 for evaluation; 1 for license) | VAGUE |
| 6 | Evaluation without a metric-like number (`91%`, `41.2`, `71`) | VAGUE |
| 7 | License without both the word "license" and a known license name (MIT, Apache, BSD, GPL family, MPL, CC-BY family, CC0, OpenRAIL, ...), or with a negation next to the license ("not released under ...") | VAGUE |
| 8 | Otherwise | PRESENT |

The worst reading wins: a quote that holds a good sentence and also "not documented" is ABSENT. One quote cannot serve two sections: if the quotes of two sections are the same, or one contains the other (at least 3 words), only the earlier section keeps it. The version is read only from a quote that contains a version word (version, release, v1, ...). Dates such as `2024.06.13` are ignored unless they are the claimed version.

These are word lists, not understanding. Unusual phrasing can be graded too low, and the challenge path exists to correct it.

## Lifecycle

```
register_card -> REGISTERED
REGISTERED --grade_card--> GRADED (challenge window opens)
REGISTERED --grade_card, undetermined, 3rd time--> UNGRADED
REGISTERED --stale, expire_card--> UNGRADED
GRADED --challenge_grade (2 GEN bond)--> CHALLENGED
GRADED --finalize after the window--> FINAL
CHALLENGED --resolve_challenge, letter changes--> FINAL (challenger gets bond + assurance)
CHALLENGED --resolve_challenge, letter holds or counter evidence does not cite the card--> FINAL (registrant gets bond + assurance)
CHALLENGED --resolve_challenge, undetermined--> GRADED, bond refunded, finalizable now
CHALLENGED --stale, cancel_challenge--> GRADED, bond refunded, finalizable now
```

The registrant may attach an optional assurance of 0 to 100 whole GEN when registering. It is returned at the end of the card's life unless a challenge changes the letter, in which case the challenger receives it together with their bond.

A challenge is only about evidence. The counter-evidence page must cite the full card URL and sit under the prefix the card was registered under; that prefix is stored on the card at registration and never changes. The original card is not extracted again. A section quote on the counter page that speaks about its section replaces the stored quote for that section, and the merged quotes are graded by the same rules. The model name and the version are never replaced. The challenge succeeds only if the letter changes, in either direction.

## Contract

`contracts/modelcardgrade.py` (single file, deploy as is). Constructor: `challenge_window_seconds`, `stale_after_seconds` (each 60 to 604800). Suggested values: `1800, 3600` for testing and `172800, 259200` for production.

| Method | Who | Purpose |
|---|---|---|
| `add_prefix(prefix, label)` / `remove_prefix(prefix)` | owner | manage the https URL prefixes new cards may be registered under; existing cards keep their stored prefix |
| `register_card(url, claimed_version)` payable, 0-100 whole GEN | anyone | register a card for a claimed version; invalid input is refunded to the caller's balance, not reverted |
| `grade_card(card_id)` | anyone | consensus extraction and grading |
| `challenge_grade(card_id, counter_evidence_url)` payable, exactly 2 GEN | anyone but the registrant | dispute the grade |
| `resolve_challenge(card_id)` | anyone | consensus extraction of the counter evidence, merged with the stored result |
| `cancel_challenge(card_id)` | anyone | refund a challenge that stayed unresolved past the stale limit |
| `finalize(card_id)` | anyone | close an unchallenged grade after the window |
| `expire_card(card_id)` | anyone | release a card stuck before its first grade |
| `withdraw()` | anyone | pull payment of the caller's balance |

Views: `get_params`, `get_rules`, `get_stats`, `list_prefixes`, `list_cards(offset, limit)`, `get_card`, `find_cards(url)`, `find_card(url, claimed_version)`, `get_grade(card_id)`, `get_grade_by_url(url, claimed_version)`, `meets_min_grade(card_id, min_grade)`, `meets_min_grade_by_url(url, claimed_version, min_grade)`, `get_balance`, `get_accounting`.

One URL can be registered for several claimed versions (up to 10). Lookups by URL need the exact version after removing a leading `v`: `2.1` does not find `2.1.0`.

Money: the contract's real balance always equals the sum of internal balances, open challenge bonds and open assurances (`get_accounting`).

## Using the registry from another contract

Only FINAL grades are exposed through `get_grade`, `get_grade_by_url` and the `meets_min_grade` views. An empty string or `False` means "no final grade".

```python
registry = gl.get_contract_at(Address("0xMODELCARDGRADE_ADDRESS"))
if not registry.view().meets_min_grade_by_url(CARD_URL, "2.1.0", "B"):
    raise Exception("no final grade B or better for this model version")
```

## Deployment

ModelCardGrade is submitted on its own as a contract; there is no frontend in this repository. The deployed address will be added here after the live tests on GenLayer Studio.

`evidence/` holds sample model cards used for live testing. The sample prefix is `https://raw.githubusercontent.com/123Cryp/ModelCardGrade/main/evidence/`.

| File | Claimed version | Expected grade |
|---|---|---|
| `m1.txt` | `2.1.0` | A (penalty 0) |
| `m1.txt` | `2.0.0` | C (penalty 25, version mismatch) |
| `m2.txt` | `1.0.0` | B (penalty 10, vague training data and license) |
| `m3.txt` | `0.9.0` | C (penalty 25, training data and limitations absent, evaluation vague) |
| `m4.txt` | `3.0.0` | D (penalty 40) |
| `m5.txt` | `1.0.0` | UNGRADED (no model name) |
| `m6.txt` | `1.4.2` | C (penalty 15, version unstated) |
| `m7.txt` | `1.4.2` | B (penalty 10, quote names two versions) |
| `m7.txt` | `1.0.0` | C (penalty 25, version mismatch) |

Counter evidence:

| Counter file | Card | Result |
|---|---|---|
| `m2-update.txt` | `m2.txt` `1.0.0` | upheld, B to A |
| `m2-partial.txt` | `m2.txt` `1.0.0` | rejected, stays B |
| `m1-retraction.txt` | `m1.txt` `2.1.0` | upheld, A to C (penalty 20) |
| `m1-note.txt` | `m1.txt` `2.1.0` | rejected without a model call on the counter page (does not cite the card) |
| `m4-repair.txt` | `m4.txt` `3.0.0` | upheld, D to A |

## Tests

```
python3 -m unittest discover -s tests -p "test_*.py"
```

No pip install is needed. The suite runs against a small offline stand-in for the GenLayer SDK (`tests/genlayer_stub.py`) with a scripted stand-in for the model. After every call it checks that the contract's real GEN balance equals its recorded obligations. It covers the URL rules, version parsing, a table of section quotes with negations, the parser fuzz tests (random word soup, case and whitespace invariance, "appending an absence phrase always gives ABSENT", "blanking a quote never lowers the penalty", timing on adversarial input), every weight and threshold, the validator checks against a tampered leader, all challenge paths, stuck-state exits, the registry views, garbage input to the payable methods, a randomized sequence test and the sample evidence in the table above. The stand-in model cannot show how a real model behaves; that is covered by the live tests on Studio.

See `ARCHITECTURE.md` for the state machine, consensus design, accounting and trust assumptions.

## License

MIT
