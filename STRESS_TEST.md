# Stress test: what broke and what was fixed

Run it yourself: `python3 stress/stress.py baseline` (the code as first shipped, frozen in `stress/baseline/`) and `python3 stress/stress.py fixed`.

**Result: 12 of 13 cases broke the first version. After the fixes, 13 of 13 pass, and 22 unit tests pass.**

All cases are offline: fake search, fake pages, and a scripted model that plays a fooled or sloppy model. They test the code's defences, **not how a real language model behaves**, and none was run against the live API.

| # | Input | What broke before | Fix | After |
|---|-------|-------------------|-----|-------|
| C1 | Blank business name | Searched for "" and wrote a brief | `clean_name` rejects blank names | PASS |
| C2 | 10,000-character name | Sent a 10,000-character search query | Max 100 characters, 8 words | PASS |
| C3 | "What is the capital of France?" | Treated it as a business and wrote a brief | Rejects questions | PASS |
| C4 | Page says "ignore previous instructions" and plants a claim | The planted claim was printed as **verified**, with a citation, because its quote really was on the page | Pages that contain instruction-like text are quarantined and noted | PASS |
| C5 | Page says "does **not** sell wholesale"; model claims it does | Verified: the quote was on the page, the meaning was reversed | Quote is rejected if it sits in a negated phrase in its own sentence | PASS |
| C6 | Search returns pages about a different company | Wrote a brief about "Zorblax" from pages that never mention it | Pages must mention every word of the name, else dropped | PASS |
| C7 | Model returns 40 claims per section | 1,865-word "one-page" brief | Max 3 verified claims per section; the rest go to the flag list; flags capped at 8 | PASS |
| C8 | 3 MB pages, 60 seed URLs | Held up (control case) | none needed | PASS |
| C9 | Model wraps its JSON in chatter that has braces | **Crashed** with a traceback after the retry | `extract_json` scans for the first valid object | PASS |
| C10 | Model returns claims or ideas as plain strings | **Crashed** (`AttributeError`) | Malformed items are skipped and flagged | PASS |
| C11 | Seed URLs: `file:///etc/passwd`, `localhost`, `169.254.169.254`, `192.168.x.x`, `ftp:` | **Tried to fetch all five.** `file:` would have put a local file's text into a brief | Only public http(s) URLs; private, loopback and link-local IPs blocked | PASS |
| C12 | Same URL given three times | Listed as three sources | Duplicates removed | PASS |
| C13 | Page is out of date (a 2014 staff count) | No sign in the brief that the figure is old | Claims near a year 3 or more years old get "(source dated 2014)" | PASS |

## The most important finding
The first version's check ("the quote appears on the page") proves a quote exists, not that the claim is true. C4 and C5 both passed that check while being wrong. The fixes narrow that gap but do not close it.

## Still not solved
- **Real prompt injection:** quarantine only catches common phrases. A cleverly worded page can still reach the model. Pages are untrusted input; read the brief before using it.
- **Meaning:** negation is checked in one sentence only. Sarcasm, "used to sell", and claims that quote real words but change their meaning can still pass.
- **DNS tricks:** a public-looking domain that resolves to a private address is not caught.
- **Wrong-company pages** that happen to contain the name (a different "Acme") still pass.
- **Real model behaviour** under these inputs is untested.
