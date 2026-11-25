# ModelCardGrade architecture

## Entities

- **Prefix**: an https URL prefix with a label, added by the contract owner. Cards can only be registered under a current prefix. When a card URL matches more than one prefix, the longest (most specific) one is used.
- **Evidence binding**: at registration the matched prefix and its label are copied onto the card (`prefix`, `label`). This binding is immutable and is the card's challenge authority: counter-evidence must sit under the stored `prefix`, never under the owner's current prefix list. Removing, re-adding or adding prefixes later affects only new registrations.
- **Claim**: the model version the registrant says the card describes, stored lower case with a leading `v` removed (`2.1.0`, `2`, `1.0.0-rc.1`). One URL can carry several claims, up to 10 registrations per URL. Two claims are the same claim only when the strings are equal; `2.1` and `2.1.0` are different.
- **Lookup by URL and version**: `find_card`, `get_grade_by_url` and `meets_min_grade_by_url` use the registration whose claim equals the given version and whose state is not UNGRADED. There is no prefix or fuzzy matching.
- **Card**: id `c1`, `c2`, ... with the canonical card URL, claim, registrant, state, optional assurance, grade record, and challenge data. A claim whose card ended UNGRADED may be registered again.
- **Grade record**: letter, penalty, the list of rule hits (`why`), the derived fields (model name, version result, and for each section its state and whether it is a signal) and the grounded quotes (`raw`) they came from.

## Card state machine

```
register_card -> REGISTERED
REGISTERED --grade_card GRADED--> GRADED
REGISTERED --grade_card UNDETERMINED (attempts+1)--> REGISTERED, or UNGRADED on the 3rd attempt (assurance refunded)
REGISTERED --stale, expire_card--> UNGRADED (assurance refunded)
GRADED --challenge_grade, once, before challenge_end--> CHALLENGED
GRADED --finalize after challenge_end--> FINAL (assurance refunded)
CHALLENGED --resolve, merged letter differs--> FINAL (challenger: bond + assurance), grade replaced, original_grade kept
CHALLENGED --resolve, merged letter equal--> FINAL (registrant: bond + assurance)
CHALLENGED --resolve, counter page lacks the card URL--> FINAL (registrant: bond + assurance)
CHALLENGED --resolve, undetermined--> GRADED, bond refunded, challenge_end = now
CHALLENGED --stale, cancel_challenge--> GRADED, bond refunded, challenge_end = now
```

Every state has a path to a terminal state (FINAL or UNGRADED) that needs no particular party. A consensus failure on the network leaves storage untouched, and the stale timers (`stale_after_seconds`) then open `expire_card` or `cancel_challenge`. Only FINAL grades are served by the registry views.

## Extraction and grading pipeline

`_assess_single` (and, for the counter page, `_assess_challenge`) runs identically on the leader and on every validator:

1. Fetch the page (first 15,000 characters). A failed fetch is UNDETERMINED.
2. Ask the model for a fixed JSON object: the model name and its quote, the card version and its quote, and one quote per section. The prompt declares the page untrusted and forbids grading. The model supplies no status flags and no numbers. The page is put between two fence lines made of `=` characters; the fence is always longer than any run of `=` in the page, so page text cannot contain the closing fence. The page itself is never altered, so quotes stay exact.
3. `_ground`: each quote is kept only if it is at least 3 characters and, after whitespace and case normalisation, occurs in the page (quotes are cut at 400 characters). The model name is kept only if it occurs inside its own grounded quote; the card version likewise inside the version quote.
4. `_derive`: for each section in a fixed order, a quote that repeats or contains (3 words or more) the quote of an earlier section is dropped. `_section_state` then reads the quote by code (see the README for the rule order): a quote without any of the section's words is ABSENT and is not a signal; explicit absence words give ABSENT; vague markers, too few words, an evaluation without a metric-like number, or a license without a license word and a known license name give VAGUE; otherwise PRESENT. Negated missing words ("nothing is undisclosed") are removed before the absence check, within one clause. The version is read from the version quote only when the quote has a version word; the distinct versions found are compared with the claim: exactly the claim is MATCH, the claim among others is MIXED, none equal is MISMATCH, none found is UNSTATED.
5. `_penalty` and `_letter`: points are summed (see README) and mapped to A / B / C / D. No model name, or fewer than three signals, gives UNDETERMINED.

Conservative defaults: a fact that cannot be grounded or parsed is treated as absent, and absent facts add the most points. Text the model invents is dropped by the quote check, so it cannot lower a penalty. Because the state comes from the words of a grounded quote, the model cannot label a quote "present" when the words say otherwise. When one quote holds both a good statement and an absence phrase, the worse reading wins.

## Consensus

Both consensus calls use `gl.vm.run_nondet_unsafe(leader, validator)` from module-level functions that capture only plain strings, never the contract. The claim is passed in as a string, so the version comparison is part of what validators recompute.

- The leader returns the full result as canonical JSON. Errors inside the closure become an ordinary UNDETERMINED result.
- A validator re-runs the whole pipeline on its own copy of the page and requires an exact match of the consensus key: status, letter, penalty, version result, and the state and signal flag of each section. The wording of quotes is not compared, so harmless differences do not block consensus. Because the penalty and every state that feeds it are compared exactly, two validators cannot take different decisions near a grade edge.
- The validator also checks the leader's output deterministically: every leader quote occurs in the validator's page, the model name and version sit inside their quotes, and re-deriving and re-grading the leader's quotes reproduces the leader's fields, penalty, rule hits and letter. A leader therefore cannot store a fabricated quote or a grade that does not follow from its quotes. A tampered or malformed leader result is rejected.
- A challenge does not re-extract the original card. It starts from the grounded quotes stored with the consensus-accepted grade, so a challenger cannot win just because a second extraction of the same page came out differently. Only the counter page is fetched and extracted. It must contain the full stored card URL as a whole token (case-insensitive, within the first 15,000 characters; a longer URL that merely starts with it does not count); without it the result is NO_ANCHOR and no model call is made. For each section, a counter quote that speaks about the section (it is a signal) replaces the stored quote. The model name and the version are never taken from the counter page. The merged quotes are graded by the same rules and the letter is compared with the stored letter. Validators check that the leader's counter quotes occur on their own copy of the counter page and that the merge reproduces the leader's result.

Pages should be static text, ideally pinned to a commit. If the page changes between validators' fetches, validators disagree and the transaction does not complete; the stale paths cover that. A model answer that is not a string (for example `null`) is treated as empty, never as the text "None".

## URL rules

Card and counter-evidence URLs must be https, ASCII with no whitespace or control characters, no credentials, no non-443 port, no `..`, `.` segments, `//` in the path, `%2e`, `%2f`, `%5c` or `%00`, no query and no fragment, and, after lower-casing the host and dropping `:443`, start with one of the owner's current prefixes (registration) or with the card's stored prefix (counter-evidence). Prefixes themselves must end with `/`, have a non-empty path and no query. Look-alike hosts and `host@evil` forms fail the parsed-URL checks. All patterns use ASCII digits `[0-9]`, so other Unicode digits are never read as numbers or versions.

## Money

All amounts are wei internally; users enter whole GEN. Payouts are pull-based: every credit goes to `balances` and `withdraw` transfers. Payable methods (`register_card`, `challenge_grade`) never raise on bad input; they credit the attached value back to the caller's balance. A test checks that no `raise` statement exists in them and a fuzz test feeds them garbage. Invariant checked in tests after every call:

```
contract balance == sum(balances) + sum(open challenge bonds) + sum(open assurances)
```

## Trust assumptions and limits

- The owner chooses which hosts count as card sources. Anyone who can write to an approved prefix can publish a page under it, so prefixes should point to locations whose content is controlled by the model publisher, ideally pinned to a commit.
- A grade describes the text of one card, not the model, and not the truth of what the card says. The version check compares strings; it does not check that the version exists anywhere.
- Section reading uses word lists. A card that says something concrete without using the section's usual words (for example no word like "limitation", "risk", "bias" or "fails" in the limitations quote) reads as absent. A vague card can contain concrete-sounding filler and read as present. The grade is structural, and the counter-evidence path exists to correct misreadings.
- Explicit absence phrases such as "not available" or "not documented" make a section ABSENT wherever they occur in the quote, even in a sentence about something else ("not available in French"). This is deliberate: the worse reading wins.
- A license counts as present only with the word "license" and a name from a fixed list (MIT, Apache, BSD, GPL family, MPL, ISC, CC0, CC-BY family, OpenRAIL, ...). Other licenses read as vague.
- Evaluation counts as present only with a metric-like number (a percentage, a decimal, or a 2 to 3 digit number). A number is not checked against anything.
- A quote that names two versions is MIXED even if the second one only appears in a history note. A version needs a version word in its quote; `1.2` and `1.2.0` are different versions; dates such as `2024.06.13` are ignored unless they are the claimed version.
- A section keeps its worst reading: a quote that mixes a concrete sentence and an absence phrase counts as absent.
- The registrant can challenge through a second address; the bond then returns to the registrant either way. The challenge is evidence-based and the letter must change to win.
- The stand-in model in the offline tests cannot show how a real model behaves; the live tests on Studio cover that.
