# Paste-into-GitHub guide

Add file > Create new file, type the path in the heading, paste the code, Commit.

## 1. `README.md`

~~~~~
# Catalogue extractor

Turns messy product information (supplier notes, OCR text from photos, spreadsheet rows) into strict JSON for a website catalogue:
`name`, `price`, `sizes`, `features`, `description` (plus `needs_review` for anything missing).

Standard library only.

```bash
export ANTHROPIC_API_KEY=...
python -m catalog examples/supplier_note.txt        # prints JSON, exit 0
python -m catalog examples/not_a_product.txt        # exit 1 or a "not a single product" answer
python -m unittest discover -s tests -t .           # 25 offline tests
```

How it works (`catalog/`):

- `prompt.py` is the prompt. Rules: JSON only, fixed keys, use only facts in the input, `null` plus `needs_review` instead of guessing.
- `validate.py` checks the reply against the schema and against the input (price, sizes and numbers must appear in the source text).
- `extract.py` runs prompt, parse, validate. On a rejection it re-sends the prompt with the list of problems and the previous reply, up to 3 attempts.

Output shape:

```json
{"name": "AirRun 2 running shoe", "price": {"amount": 349, "currency": "AED"},
 "sizes": ["40","41","42","43","44","45"],
 "features": ["Mesh upper", "Foam sole", "Reflective heel tab"],
 "description": "20-60 words, no hype, no invented numbers.", "needs_review": []}
```

See `FAILURE_CASES.md` for the 22 failure cases, which ones are caught and which are known gaps.
~~~~~

## 2. `.gitignore`

```
__pycache__/
*.pyc
```

## 3. `FAILURE_CASES.md`

~~~~~
# Failure cases

Each row says what goes wrong, what the code does about it, and whether a test covers it.
"Caught" means validation rejects the reply and the retry shows the model the exact problem.
"Known gap" means validation cannot catch it, so a person has to.

| # | Failure | Example | Handling | Test |
| --- | --- | --- | --- | --- |
| 1 | Reply wrapped in a markdown fence | ```` ```json {...} ``` ```` | Caught, rejected (not silently stripped), retried | `test_fenced_json_rejected` |
| 2 | Chatty reply, not JSON | `Sure! Here it is: {...}` | Caught, retried | `test_prose_rejected` |
| 3 | Price as a string | `"amount": "349"` | Caught | `test_price_as_string` |
| 4 | Invented price | input says 349, reply says 299 | Caught: the number must appear in the input | `test_invented_price`, `test_hallucinated_price_then_fixed` |
| 5 | Price missing from input | no price anywhere | Model must return `null` and list `price` in `needs_review`; `null` without that is rejected | `test_null_price_needs_review`, `test_null_price_declared_ok` |
| 6 | Currency not a 3-letter code | `Dhs` | Caught | `test_bad_currency` |
| 7 | Invented size | `47` not in the input | Caught (whole-token match) | `test_invented_size` |
| 8 | Duplicate sizes or features | `["40","40"]` | Caught | `test_duplicate_sizes` |
| 9 | Too few or too many features | `[]`, or 9 items | Caught | `test_no_features`, `test_too_many_features` |
| 10 | Description too short or long | `Nice shoe.` | Caught (20-60 words) | `test_short_description` |
| 11 | Hype words or `!` in description | `the best shoe!` | Caught | `test_hype_and_bang` |
| 12 | Invented number in description | `Tested for 500 hours` | Caught: numbers must come from the input | `test_invented_number_in_description` |
| 13 | Extra or missing keys | `sku` added | Caught | `test_extra_key` |
| 14 | Input is not one product | a meeting email | Model returns the `not a single product` answer; accepted so a person can see it, and `needs_review` carries the reason | `test_declared_not_a_product` |
| 15 | API error or timeout | HTTP 529 | Counts as an attempt; the backend also retries 429/5xx with backoff | `test_api_error_is_an_attempt` |
| 16 | Still invalid after all attempts | model keeps failing | `extract` returns `ok=False` with every reply and its problems; the CLI exits 1 | `test_gives_up_after_max` |
| 17 | **Known gap:** old price vs sale price | `Dhs 349 (was 420)`; reply says 420 | Not caught: both numbers are in the input. The prompt tells the model to use the current price; check price changes by eye | `test_old_price_is_grounded_but_allowed` |
| 18 | **Known gap:** negated size | `(no 46)`; reply lists 46 | Not caught: a text match cannot read "no" | `test_known_gap_negated_size` |
| 19 | **Known gap:** OCR typos | `5OO ml` (letter O) | The model may write `500 ml` in the description; check 12 then rejects it as an invented number. The retry usually drops the number. Fix the typo in the input if it keeps failing | none (needs live model) |
| 20 | **Known gap:** wrong but grounded facts | colour taken from the wrong row of a spreadsheet | Not caught: every word is in the input | none |
| 21 | **Known gap:** non-English input | Arabic notes | Name and features may come out in Arabic; the word-count and hype checks assume English | none |
| 22 | **Known gap:** two currencies in the input | `$40 / Dhs 150` | Model must pick one; no check that it picked the shop's currency | none |

## What the retry loop does not do

- It does not repair JSON for the model. A bad reply costs another call so the model learns what was wrong.
- It stops after 3 attempts by default (`--attempts`). A failed product goes to a person; it is never published half-valid.
- A valid reply can still list fields in `needs_review`. Treat those as "publish after a human fills them in".

## Not tested live

The unit tests use a scripted fake model. The real `AnthropicModel` has not been run here (no API key), and the prompt itself has not been scored on real model replies. Run the four files in `examples/` first and read the results.
~~~~~

## 4. `catalog/__init__.py`

```

```

## 5. `catalog/__main__.py`

```
"""python -m catalog input.txt [--model claude-haiku-5-5]  -> prints JSON, exit 0 ok, 1 failed (needs a human)."""
import argparse
import json
import sys

from .extract import AnthropicModel, extract

ap = argparse.ArgumentParser()
ap.add_argument("input")
ap.add_argument("--model", default="claude-haiku-5-5")
ap.add_argument("--attempts", type=int, default=3)
a = ap.parse_args()
res = extract(open(a.input, encoding="utf-8").read(), AnthropicModel(a.model), a.attempts)
if res.ok:
    print(json.dumps(res.data, indent=2, ensure_ascii=False))
    sys.exit(0)
print(json.dumps({"failed": True, "attempts": res.attempts, "last_problems": res.history[-1]["problems"]}, indent=2), file=sys.stderr)
sys.exit(1)
```

## 6. `catalog/extract.py`

```
"""extract(): prompt -> parse -> validate -> retry with the problems fed back."""
from __future__ import annotations

import json
import os
import time
import urllib.error
import urllib.request
from dataclasses import dataclass, field

from .prompt import build_prompt
from .validate import parse_json, validate


class AnthropicModel:
    """Real backend (needs ANTHROPIC_API_KEY). Not exercised against the live API in this repo's tests."""

    def __init__(self, model_id="claude-haiku-5-5", max_tokens=700):
        self.model_id, self.max_tokens = model_id, max_tokens
        self.key = os.environ.get("ANTHROPIC_API_KEY", "")
        self.base = os.environ.get("ANTHROPIC_BASE_URL", "https://api.anthropic.com").rstrip("/")

    def generate(self, prompt: str) -> str:
        if not self.key:
            raise RuntimeError("ANTHROPIC_API_KEY is not set")
        body = json.dumps({"model": self.model_id, "max_tokens": self.max_tokens, "temperature": 0,
                           "messages": [{"role": "user", "content": prompt}]}).encode()
        req = urllib.request.Request(f"{self.base}/v1/messages", data=body, headers={
            "x-api-key": self.key, "anthropic-version": "2023-06-01", "content-type": "application/json"})
        for attempt in range(4):
            try:
                with urllib.request.urlopen(req, timeout=120) as r:
                    data = json.load(r)
                return "".join(b.get("text", "") for b in data.get("content", []) if b.get("type") == "text")
            except urllib.error.HTTPError as e:
                if e.code in (429, 500, 502, 503, 529) and attempt < 3:
                    time.sleep(2 ** attempt)
                    continue
                raise RuntimeError(f"API error {e.code}") from e


@dataclass
class Result:
    ok: bool
    data: dict | None
    attempts: int
    history: list = field(default_factory=list)  # one {"reply", "problems"} per attempt
    needs_review: list = field(default_factory=list)


def extract(raw: str, model, max_attempts: int = 3) -> Result:
    previous, problems, history = None, None, []
    for attempt in range(1, max_attempts + 1):
        try:
            reply = model.generate(build_prompt(raw, previous, problems))
        except Exception as e:  # network/API failure counts as a failed attempt
            problems, previous = [f"model call failed: {e}"], ""
            history.append({"reply": "", "problems": problems})
            continue
        obj, problem = parse_json(reply)
        problems = [problem] if problem else validate(obj, raw)
        history.append({"reply": reply, "problems": problems})
        if not problems:
            return Result(True, obj, attempt, history, obj["needs_review"])
        previous = reply
    return Result(False, None, max_attempts, history)
```

## 7. `catalog/prompt.py`

```
"""The extraction prompt. Retries append the validator's error list so the model can fix exactly what failed."""

SYSTEM = """You turn messy product information into ONE JSON object for a website catalogue.
The input may be supplier notes, text read from photos (OCR, so it can contain typos), or spreadsheet rows.

OUTPUT RULES
1. Reply with the JSON object only. No markdown fences, no commentary.
2. Use exactly these keys and no others:
   {
     "name": string, 3-80 characters, the product name only (no price, size or SKU),
     "price": {"amount": number, "currency": "AED"} or null,
     "sizes": array of strings, [] if the product has no sizes,
     "features": array of 1-8 short strings (max 80 characters each),
     "description": string of 20-60 words, plain sentences, no exclamation marks,
     "needs_review": array of strings naming any field you could not fill from the input
   }
3. Use ONLY facts in the input. Never guess or invent a price, size, material or claim.
   If the price is missing or unclear, set "price" to null and add "price" to "needs_review".
   If no feature is stated, write one feature from the product name and add "features" to "needs_review".
4. Price: the number as written, with a dot for decimals (1,299.50 -> 1299.5). Currency is a 3-letter code
   (Dhs, AED, dirham -> AED; $ -> USD; EUR/euro -> EUR). If several prices appear (old and sale), use the current one
   and add "price" to "needs_review" only if you cannot tell which is current.
5. Sizes: keep the supplier's labels (S, M, 42, 10 oz), each once, in a sensible order.
6. Features are short noun phrases, one idea each, no duplicates.
7. The description is written for a shopper, uses only the stated facts, and never contains hype words
   (best, perfect, amazing, unbeatable, revolutionary).
8. If the input is not about a single product (several products, or nothing usable), return
   {"name": "", "price": null, "sizes": [], "features": [], "description": "", "needs_review": ["not a single product"]}.

INPUT
<<<
{raw}
>>>"""

RETRY = """

YOUR PREVIOUS REPLY WAS REJECTED. Problems found:
{problems}

Previous reply:
{previous}

Reply again with the corrected JSON object only."""


def build_prompt(raw: str, previous: str | None = None, problems: list[str] | None = None) -> str:
    text = SYSTEM.replace("{raw}", raw.strip())
    if previous is not None and problems:
        text += RETRY.replace("{problems}", "\n".join(f"- {p}" for p in problems)).replace("{previous}", previous[:1500])
    return text
```

## 8. `catalog/validate.py`

~~~~~
"""Schema validation + grounding checks. Returns a list of problems; an empty list means valid."""
from __future__ import annotations

import json
import re

KEYS = {"name", "price", "sizes", "features", "description", "needs_review"}
HYPE = ("best", "perfect", "amazing", "unbeatable", "revolutionary")
CURRENCY = re.compile(r"^[A-Z]{3}$")


def parse_json(reply: str):
    """Strict parse. Returns (obj, problem). Markdown fences are rejected, not silently repaired."""
    text = reply.strip()
    if text.startswith("```"):
        return None, "reply is wrapped in a markdown code fence; reply with raw JSON only"
    try:
        obj = json.loads(text)
    except json.JSONDecodeError as e:
        return None, f"reply is not valid JSON ({e.msg} at char {e.pos}); reply with the JSON object only"
    if not isinstance(obj, dict):
        return None, "reply must be a JSON object"
    return obj, None


def _numbers_in(raw: str) -> set[float]:
    cleaned = re.sub(r"(?<=\d),(?=\d{3})", "", raw)  # 1,299 -> 1299
    return {float(x) for x in re.findall(r"\d+(?:\.\d+)?", cleaned)}


def validate(obj: dict, raw: str = "") -> list[str]:
    p: list[str] = []
    extra, missing = set(obj) - KEYS, KEYS - set(obj)
    if extra:
        p.append(f"unexpected keys: {sorted(extra)}")
    if missing:
        p.append(f"missing keys: {sorted(missing)}")
    review = obj.get("needs_review")
    if not (isinstance(review, list) and all(isinstance(x, str) for x in review)):
        p.append("needs_review must be an array of strings")
        review = []
    if review == ["not a single product"]:
        return p  # the declared "nothing usable" answer; shape is checked above
    name = obj.get("name")
    if not isinstance(name, str) or not 3 <= len(name.strip()) <= 80:
        p.append("name must be a string of 3-80 characters")
    price = obj.get("price")
    if price is None:
        if "price" not in review:
            p.append('price is null but "price" is not listed in needs_review')
    elif not isinstance(price, dict) or set(price) != {"amount", "currency"}:
        p.append('price must be null or {"amount": number, "currency": "AED"}')
    else:
        amt, cur = price["amount"], price["currency"]
        if isinstance(amt, bool) or not isinstance(amt, (int, float)) or amt <= 0:
            p.append("price.amount must be a number greater than 0 (not a string)")
        elif raw and float(amt) not in _numbers_in(raw):
            p.append(f"price {amt} does not appear in the input; do not invent prices")
        if not isinstance(cur, str) or not CURRENCY.match(cur):
            p.append("price.currency must be a 3-letter upper-case code such as AED")
    sizes = obj.get("sizes")
    if not (isinstance(sizes, list) and all(isinstance(s, str) and s.strip() for s in sizes)):
        p.append("sizes must be an array of non-empty strings ([] if none)")
    else:
        if len(set(sizes)) != len(sizes):
            p.append("sizes contains duplicates")
        for s in sizes:
            if raw and not re.search(rf"(?<![\w.]){re.escape(s)}(?![\w])", raw, re.I):
                p.append(f"size {s!r} does not appear in the input")
    feats = obj.get("features")
    if not (isinstance(feats, list) and 1 <= len(feats) <= 8 and all(isinstance(f, str) and 0 < len(f) <= 80 for f in feats)):
        p.append("features must be 1-8 strings of at most 80 characters")
    elif len({f.lower() for f in feats}) != len(feats):
        p.append("features contains duplicates")
    desc = obj.get("description")
    if not isinstance(desc, str):
        p.append("description must be a string")
    else:
        words = len(desc.split())
        if not 20 <= words <= 60:
            p.append(f"description has {words} words; it must have 20-60")
        if "!" in desc:
            p.append("description must not contain exclamation marks")
        bad = [h for h in HYPE if re.search(rf"\b{h}\b", desc, re.I)]
        if bad:
            p.append(f"description uses hype words: {bad}")
        if raw:
            new = _numbers_in(desc) - _numbers_in(raw)
            if new:
                p.append(f"description contains numbers not in the input: {sorted(new)}")
    return p
~~~~~

## 9. `examples/not_a_product.txt`

```
Hi team, meeting moved to Thursday 3pm. Please bring the Q4 numbers.
```

## 10. `examples/ocr_photo.txt`

```
STEEL FLASK 5OO ml   vacuum insulated
keeps drinks cold 24h / hot 12h
BPA free lid   AED 59.00
```

## 11. `examples/spreadsheet_row.txt`

```
sku,product,price,size_options,notes
TS-221,Cotton crew tee,45,"S;M;L;XL","100% cotton, pre-shrunk, unisex fit"
```

## 12. `examples/supplier_note.txt`

```
fwd: new trainers from Al Noor sports
"AirRun 2 lightweight running shoe" - Dhs 349 (was 420, sale price now)
sizes 40 41 42 43 44 45  (no 46)
mesh upper, foam sole, reflective heel tab. wash cold. colour: black/white
```

## 13. `tests/__init__.py`

```

```

## 14. `tests/test_extract.py`

~~~~~
import json
import unittest
from pathlib import Path

from catalog.extract import extract
from catalog.validate import parse_json, validate

EX = Path(__file__).parent.parent / "examples"
NOTE = (EX / "supplier_note.txt").read_text()
GOOD = {"name": "AirRun 2 running shoe", "price": {"amount": 349, "currency": "AED"},
        "sizes": ["40", "41", "42", "43", "44", "45"],
        "features": ["Mesh upper", "Foam sole", "Reflective heel tab"],
        "description": "A lightweight running shoe with a breathable mesh upper and a foam sole. "
                       "The reflective heel tab helps you stay visible, and it comes in black and white.",
        "needs_review": []}


def j(**kw):
    d = dict(GOOD); d.update(kw); return json.dumps(d)


class Scripted:
    def __init__(self, *replies): self.replies, self.prompts = list(replies), []
    def generate(self, prompt):
        self.prompts.append(prompt); return self.replies.pop(0)


class Failures(unittest.TestCase):
    def problems(self, **kw): return validate(json.loads(j(**kw)), NOTE)

    def test_good_passes(self): self.assertEqual(validate(GOOD, NOTE), [])
    def test_fenced_json_rejected(self): self.assertIn("fence", parse_json("```json\n{}\n```")[1])
    def test_prose_rejected(self): self.assertIn("not valid JSON", parse_json("Sure! Here it is: {}")[1])
    def test_price_as_string(self): self.assertTrue(self.problems(price={"amount": "349", "currency": "AED"}))
    def test_invented_price(self): self.assertTrue(any("do not invent" in x for x in self.problems(price={"amount": 299, "currency": "AED"})))
    def test_old_price_is_grounded_but_allowed(self):  # 420 is in the note; grounding cannot tell old from new (documented)
        self.assertEqual(self.problems(price={"amount": 420, "currency": "AED"}), [])
    def test_null_price_needs_review(self): self.assertTrue(self.problems(price=None))
    def test_null_price_declared_ok(self): self.assertEqual(self.problems(price=None, needs_review=["price"]), [])
    def test_bad_currency(self): self.assertTrue(self.problems(price={"amount": 349, "currency": "Dhs"}))
    def test_invented_size(self): self.assertTrue(self.problems(sizes=["40", "47"]))
    def test_known_gap_negated_size(self):  # note says "(no 46)"; a text check cannot read negation, so this passes. Documented.
        self.assertEqual(self.problems(sizes=["40", "46"]), [])
    def test_duplicate_sizes(self): self.assertTrue(self.problems(sizes=["40", "40"]))
    def test_no_features(self): self.assertTrue(self.problems(features=[]))
    def test_too_many_features(self): self.assertTrue(self.problems(features=[f"f{i}" for i in range(9)]))
    def test_short_description(self): self.assertTrue(self.problems(description="Nice shoe."))
    def test_hype_and_bang(self): self.assertEqual(len(self.problems(description="This is the best shoe ever made for running and walking and everything else you could want to do in a day! " * 1)), 2)
    def test_invented_number_in_description(self):
        d = GOOD["description"] + " Tested for 500 hours."
        self.assertTrue(any("numbers not in the input" in x for x in self.problems(description=d)))
    def test_extra_key(self): self.assertTrue(any("unexpected keys" in x for x in validate({**GOOD, "sku": "x"}, NOTE)))
    def test_declared_not_a_product(self):
        o = {"name": "", "price": None, "sizes": [], "features": [], "description": "", "needs_review": ["not a single product"]}
        self.assertEqual(validate(o, "meeting moved"), [])
    def test_examples_ground(self):
        flask = {**GOOD, "name": "Steel flask", "price": {"amount": 59.0, "currency": "AED"}, "sizes": [],
                 "description": "A vacuum insulated steel flask with a BPA free lid. It keeps drinks cold for 24 hours and hot for 12 hours, so it suits a long day out or a daily commute."}
        self.assertEqual(validate(flask, (EX / "ocr_photo.txt").read_text()), [])


class Retry(unittest.TestCase):
    def test_first_try(self):
        m = Scripted(j()); r = extract(NOTE, m)
        self.assertTrue(r.ok); self.assertEqual(r.attempts, 1)

    def test_retry_feeds_problems_back(self):
        m = Scripted("```json\n{}\n```", j()); r = extract(NOTE, m)
        self.assertTrue(r.ok); self.assertEqual(r.attempts, 2)
        self.assertIn("REJECTED", m.prompts[1]); self.assertIn("markdown code fence", m.prompts[1])
        self.assertNotIn("REJECTED", m.prompts[0])

    def test_gives_up_after_max(self):
        m = Scripted("nope", "nope", "nope"); r = extract(NOTE, m)
        self.assertFalse(r.ok); self.assertEqual(r.attempts, 3); self.assertEqual(len(r.history), 3)

    def test_api_error_is_an_attempt(self):
        class Boom:
            n = 0
            def generate(self, p):
                Boom.n += 1
                if Boom.n == 1: raise RuntimeError("API error 529")
                return j()
        r = extract(NOTE, Boom()); self.assertTrue(r.ok); self.assertEqual(r.attempts, 2)

    def test_hallucinated_price_then_fixed(self):
        m = Scripted(j(price={"amount": 299, "currency": "AED"}), j()); r = extract(NOTE, m)
        self.assertTrue(r.ok); self.assertIn("do not invent", m.prompts[1])


if __name__ == "__main__":
    unittest.main()
~~~~~
