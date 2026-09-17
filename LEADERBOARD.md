# Leaderboard: threat model, evidence, and what each fix tier buys

The board is one Firestore collection (`scores`) in project `bugwalk-f3c4a`. The client
reads the top 20 and appends one document per run. Everything below was verified against
the live project, not inferred from the code.

## The one thing no rule can fix

**The score is computed in the browser.** A submission is therefore a claim, not a fact.
No Security Rule can distinguish a real run from the game's own submit function being
called by someone who loaded the page. Any fix below should be read with that in mind:
the goal is to bound what a bad submission can look like, not to prove a good one.

## Measured behaviour of the deployed policy

| probe | result |
|---|---|
| create a run with the public web config | **allowed** |
| `deleteDoc`, `writeBatch().delete()`, REST `DELETE`, REST `:batchWrite` | **denied** (`permission-denied` / HTTP 403) |
| `updateDoc`, `setDoc`, `setDoc{merge}` | **denied** |
| `signInAnonymously()` | `auth/configuration-not-found` -- **Auth is not enabled on the project at all** |
| name 14 chars / 15 chars | accepted / **denied** |
| score 250000 / 250001 | accepted / **denied** |
| score 10,000,000 (earlier same day) | accepted at the time; **later denied** |
| float, string, `null`, or missing `score` | all **denied** |
| `name` as a number or object | **denied** |

Two conclusions worth acting on:

1. **The policy was only in the console.** The score ceiling changed mid-audit from
   `>=10,000,000` to exactly `250,000` with no commit and no diff. Nothing in the repo
   recorded it. Committing `firestore.rules` is what fixes that.
2. **The live API key is not a finding.** A Firebase web `apiKey` is a project
   identifier and is designed to ship in client code. It grants nothing beyond the
   rules; raw REST calls with it fail exactly where the SDK fails. Don't spend time
   trying to hide or rotate it.

## What is reachable, in order of how little it matters

- **Unbounded appends.** Nothing caps the number of rows. A score-ordered top-20 means
  enough high-scoring junk displaces real runs from view entirely.
- **Misleading renders, not code execution.** Every field is written with
  `textContent`, so there is no stored-XSS class here, and the name column
  (`min-width:0` + `text-overflow:ellipsis`) cannot be overflowed. What *is* reachable is
  a bidi override (`U+202E`) making a name paint what looks like a second score column,
  zero-width characters producing a name-less "ghost" row, and stacked combining marks
  rendering a corrupted glyph. This PR strips the first two for display and leaves
  combining marks alone, since they're legitimate in many scripts.
- **Non-deterministic tie-breaking.** The query has no secondary sort and the rank was
  the loop index, so two runs on the same score render as different ranks whose order
  can flip between fetches. This PR ranks client-side instead.

## Fix tiers

**Tier 0 -- what was deployed.** Open create, no auth, client-computed scores. Anyone
with a browser can top the board.

**Tier 1 -- `firestore.rules` (this PR, no console changes needed).** Shape validation
(name string 1-14, score int 0-250000, no unexpected fields), append-only by explicit
policy, everything else closed. Removes the ability to post malformed or out-of-range
rows and makes the append-only behaviour intentional rather than incidental.

**Tier 2 -- `firestore.rules.auth` (this PR, two-step deploy).** Requires Anonymous Auth,
so a submission must carry a token and must claim its own uid. Stops curl, REST, scripts,
and copied payloads dead. Does **not** stop someone driving the page itself -- auto
sign-in makes the identity free. Enable Auth in the console *before* deploying this file
or every submission fails.

**Tier 3 -- server-side verification (not in this PR, and the only real fix).** Submit a
seed plus an input log, re-simulate the run in a Cloud Function, and let the function
write the score with admin credentials. This requires the run to be deterministic and
replayable; the game currently uses unseeded randomness for spawns and timing, so Tier 3
is a genuine refactor, not a rules change. If the board is meant to be competitive, this
is the only version where a score means anything. Until then, the honest framing is
"a fun board that resists casual abuse".

## Migration notes for the new fields

New runs now carry `createdAt: serverTimestamp()` and, when a session exists, `uid`.
Existing runs have neither.

**Do not add a secondary `orderBy` on `createdAt` yet.** Firestore's `orderBy` silently
excludes documents that lack the ordered field, so adding it now would hide every legacy
score from the board. Ties are broken client-side in the meantime. Backfilling requires
owner credentials (console or Admin SDK) because `update` is denied to clients by design
-- which is the intended trade, not an oversight.

## Verifying a deploy

```bash
firebase deploy --only firestore:rules

# a valid append should land
curl -s -X POST "https://firestore.googleapis.com/v1/projects/bugwalk-f3c4a/databases/(default)/documents/scores" \
  -H 'Content-Type: application/json' \
  -d '{"fields":{"name":{"stringValue":"rulecheck"},"score":{"integerValue":"1"}}}'

# and these should each be rejected: score 250001, name of 15 chars, extra fields,
# a float score, and any DELETE or PATCH against an existing document
```

Under `firestore.rules.auth`, that same unauthenticated POST must return 403. That is the
single clearest check that Tier 2 is actually live.
