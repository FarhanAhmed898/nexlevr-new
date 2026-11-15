# Paste into GitHub
Create each file below (GitHub: Add file > Create new file, type the path in the name box).

## `README.md`

````
# Business research agent

Give it a business name. It searches public pages, fetches them, and writes a one-page
markdown brief: what they sell, who their customers are, online presence, three growth
ideas. Anything it could not verify is listed under **Could not verify**.

```
export ANTHROPIC_API_KEY=...
python3 -m research_agent "Business name" --out brief.md
python3 -m research_agent "Business name" --url https://their-site.com --url https://another-page
```
Standard library only, Python 3.9+. `RESEARCH_MODEL` overrides the model.

## How it works (and why it flags things)
1. **Gather** (`sources.py`): DuckDuckGo HTML search, plus any `--url` you give, then fetch up to 6 pages and strip them to text. Every failed search or fetch is recorded and printed in the brief.
2. **Extract** (`brief.py`): the model gets the numbered pages and must return JSON claims, each with a source id and an **exact quote** from that page.
3. **Verify in code, not by the model**: a claim passes only if its quote is found on the cited page (whitespace and curly quotes normalised) and any number in the claim also appears on the page. Failures move to *Could not verify* with the reason.
4. **Ideas** are labelled as suggestions, not facts, and tagged with the source they build on.
5. If no page can be read, it writes no brief and says why. Bad JSON from the model gets one retry.

## What was tested, honestly
- 12 offline tests pass (`python3 -m unittest discover -s tests -t .`): text extraction, search parsing, quote verification (good, fabricated, wrong source, invented number), retry, flagging, failed fetches, end to end.
- The model in tests is a scripted mock, not a language model. **It was never run against the live API or live search**, because the build sandbox had no API key and no web access. `examples/sample_brief.md` is a fixture run on a fictional company and only shows the format.
- Expect to adjust on first real run: DuckDuckGo may block or change its HTML (use `--url` to bypass); JavaScript-only sites yield little text and are skipped.

## Limits
- Verification proves a quote exists on a page, not that the page is right or current.
- A quote can be real but not support the claim; a person should skim the sources.
- Only 6 pages of 12,000 characters each are read. Paywalled and login pages are not.

````

## `examples/sample_brief.md`

````
<!-- Fictional company, scripted mock model, fixture page. Shows the format only. -->
# Example Bakery (fictional): research brief

## What they sell
- Sells sourdough bread, croissants and custom birthday cakes [S1]

## Who their customers are
- Customers are in Sharjah and Dubai [S1]

## Online presence
- Takes orders online and is on Instagram [S1]

## Three growth ideas (our suggestions, not facts)
1. Corporate breakfast boxes. Dubai offices already fall inside the delivery area [builds on S1]
2. Weekly bread subscription. online ordering exists, so repeat orders are the next step [builds on S1]
3. Instagram cake gallery. custom cakes sell on visuals [builds on S1]

## Could not verify
- (online_presence) Has 12 branches: quote not found on the cited page

## Sources
- [S1] https://example.test/a

````

## `research_agent/__init__.py`

````

````

## `research_agent/__main__.py`

````
import argparse
import sys

from .brief import ask_model, build, render
from .llm import AnthropicClient
from .sources import gather


def run(name, client, seed_urls=(), search_fn=None, fetch_fn=None, max_pages=6):
    kw = {k: v for k, v in (("search_fn", search_fn), ("fetch_fn", fetch_fn)) if v}
    pages, notes = gather(name, seed_urls, max_pages=max_pages, **kw)
    if not pages:
        raise SystemExit("No readable pages found, so no brief was written.\n" + "\n".join(notes))
    data = ask_model(client, name, pages)
    ok, bad, ideas = build(name, pages, data)
    return render(name, pages, ok, bad, ideas, notes)


def main(argv=None):
    ap = argparse.ArgumentParser(prog="research_agent")
    ap.add_argument("business")
    ap.add_argument("--url", action="append", default=[], help="seed page (repeatable)")
    ap.add_argument("--out", help="write markdown here")
    a = ap.parse_args(argv)
    text = run(a.business, AnthropicClient(), a.url)
    if a.out:
        open(a.out, "w").write(text + "\n")
    print(text)


if __name__ == "__main__":
    sys.exit(main())

````

## `research_agent/brief.py`

````
"""Ask the model for claims with quotes, verify them in code, render the brief."""
import json
import re

SYSTEM = """You write a company brief using ONLY the numbered source pages given.
Return JSON only, in this shape:
{"sells":[C],"customers":[C],"online_presence":[C],
 "ideas":[{"idea":"...","why":"...","basis":"S1"}]}
where C = {"text":"one factual sentence","source":"S1","quote":"exact words copied from that page"}.
Rules: every claim needs a source id and a quote copied exactly from that page.
If the pages do not say it, leave it out. Do not guess. At most 3 claims per section
and exactly 3 ideas. Each idea's "basis" is the source id of the fact it builds on."""


def _norm(s):
    return re.sub(r"\s+", " ", re.sub(r"[‘’]", "'", re.sub(r"[“”]", '"', s))).strip().lower()


def verify_claim(claim, pages):
    """Return (ok, reason). A claim passes only if its quote is on its page."""
    page = next((p for p in pages if p.id == claim.get("source")), None)
    if page is None:
        return False, "cites a source that was not fetched"
    quote = _norm(str(claim.get("quote", "")))
    if len(quote) < 15:
        return False, "no usable quote"
    if quote not in _norm(page.text):
        return False, "quote not found on the cited page"
    claim_nums = set(re.findall(r"\d[\d,.]*", str(claim.get("text", ""))))
    if not all(n in quote or n in _norm(page.text) for n in claim_nums):
        return False, "a number in the claim is not on the page"
    return True, ""


def ask_model(client, name, pages):
    ctx = "\n\n".join(f"[{p.id}] {p.url}\n{p.text}" for p in pages)
    user = f"Company: {name}\n\nSources:\n{ctx}"
    last = ""
    for attempt in range(2):  # one retry on bad JSON
        raw = client.complete(SYSTEM, user if attempt == 0 else user + "\n\nReturn valid JSON only.")
        m = re.search(r"\{.*\}", raw, re.S)
        try:
            return json.loads(m.group(0))
        except (AttributeError, ValueError):
            last = raw[:200]
    raise ValueError(f"model did not return valid JSON: {last!r}")


def build(name, pages, data):
    """Split everything the model said into verified and unverified."""
    ok, bad, ideas = {"sells": [], "customers": [], "online_presence": []}, [], []
    for sec in ok:
        for c in data.get(sec, []) or []:
            good, why = verify_claim(c, pages)
            (ok[sec].append(c) if good else bad.append((sec, c.get("text", "?"), why)))
    for i in (data.get("ideas") or [])[:3]:
        good, why = verify_claim({"source": i.get("basis"), "quote": i.get("quote", ""), "text": ""}, pages) \
            if i.get("quote") else (any(p.id == i.get("basis") for p in pages), "basis source was not fetched")
        ideas.append((i, good, why))
    return ok, bad, ideas


def render(name, pages, ok, bad, ideas, notes):
    L = [f"# {name}: research brief", ""]
    titles = {"sells": "What they sell", "customers": "Who their customers are",
              "online_presence": "Online presence"}
    for sec, title in titles.items():
        L.append(f"## {title}")
        L += [f"- {c['text']} [{c['source']}]" for c in ok[sec]] or ["- Nothing verified from the sources found."]
        L.append("")
    L.append("## Three growth ideas (our suggestions, not facts)")
    for n, (i, good, _) in enumerate(ideas, 1):
        tag = f"[builds on {i.get('basis')}]" if good else "[basis not verified]"
        L.append(f"{n}. {i.get('idea', '?')}. {i.get('why', '')} {tag}")
    if not ideas:
        L.append("- No ideas could be tied to a fetched source.")
    L += ["", "## Could not verify"]
    flags = [f"- ({s}) {t}: {w}" for s, t, w in bad]
    flags += [f"- (idea) {i.get('idea', '?')}: {w}" for i, g, w in ideas if not g]
    flags += [f"- Run note: {n}" for n in notes]
    L += flags or ["- Nothing flagged. Every claim's quote was found on its source page."]
    L += ["", "## Sources"] + [f"- [{p.id}] {p.url}" for p in pages]
    return "\n".join(L)

````

## `research_agent/llm.py`

````
"""Model clients. AnthropicClient needs ANTHROPIC_API_KEY; MockClient is for tests."""
import json
import os
import urllib.request

MODEL = os.environ.get("RESEARCH_MODEL", "claude-sonnet-5-5")


class AnthropicClient:
    def __init__(self, api_key=None, model=MODEL):
        self.key = api_key or os.environ.get("ANTHROPIC_API_KEY")
        if not self.key:
            raise SystemExit("Set ANTHROPIC_API_KEY (or run with --mock for a demo).")
        self.model = model

    def complete(self, system: str, user: str) -> str:
        body = json.dumps({"model": self.model, "max_tokens": 2000, "system": system,
                           "messages": [{"role": "user", "content": user}]}).encode()
        req = urllib.request.Request(
            "https://api.anthropic.com/v1/messages", data=body,
            headers={"x-api-key": self.key, "anthropic-version": "2023-06-01",
                     "content-type": "application/json"})
        with urllib.request.urlopen(req, timeout=120) as r:
            data = json.load(r)
        return "".join(b.get("text", "") for b in data["content"] if b["type"] == "text")


class MockClient:
    """Returns scripted replies in order. Not a language model."""

    def __init__(self, replies):
        self.replies, self.calls = list(replies), []

    def complete(self, system, user):
        self.calls.append(user)
        return self.replies.pop(0)

````

## `research_agent/sources.py`

````
"""Find and fetch public pages. Standard library only."""
import re
import urllib.parse
import urllib.request
from dataclasses import dataclass
from html.parser import HTMLParser

UA = "Mozilla/5.0 (research-agent; student project)"


@dataclass
class Page:
    id: str
    url: str
    text: str


class _Text(HTMLParser):
    SKIP = {"script", "style", "noscript", "svg", "head"}

    def __init__(self):
        super().__init__()
        self.parts, self.depth = [], 0

    def handle_starttag(self, tag, attrs):
        if tag in self.SKIP:
            self.depth += 1

    def handle_endtag(self, tag):
        if tag in self.SKIP and self.depth:
            self.depth -= 1

    def handle_data(self, data):
        if not self.depth:
            self.parts.append(data)


def html_to_text(html: str) -> str:
    p = _Text()
    p.feed(html)
    return re.sub(r"\s+", " ", " ".join(p.parts)).strip()


def fetch(url: str, timeout: int = 15, max_bytes: int = 600_000) -> str:
    """Return page text, or raise on any failure."""
    req = urllib.request.Request(url, headers={"User-Agent": UA})
    with urllib.request.urlopen(req, timeout=timeout) as r:
        raw = r.read(max_bytes)
        charset = r.headers.get_content_charset() or "utf-8"
    return html_to_text(raw.decode(charset, errors="replace"))


def parse_ddg(html: str) -> list:
    urls = []
    for href in re.findall(r'class="result__a"[^>]*href="([^"]+)"', html):
        q = urllib.parse.urlparse(href).query
        target = urllib.parse.parse_qs(q).get("uddg", [href])[0]
        if target.startswith("http") and target not in urls:
            urls.append(target)
    return urls


def ddg_search(query: str, timeout: int = 15) -> list:
    url = "https://html.duckduckgo.com/html/?q=" + urllib.parse.quote(query)
    req = urllib.request.Request(url, headers={"User-Agent": UA})
    with urllib.request.urlopen(req, timeout=timeout) as r:
        return parse_ddg(r.read().decode("utf-8", errors="replace"))


def gather(name, seed_urls, search_fn=ddg_search, fetch_fn=fetch, max_pages=6):
    """Collect up to max_pages readable pages. Returns (pages, notes).

    notes lists every search or fetch that failed, so the brief can say
    what the agent could not reach."""
    notes, urls = [], list(seed_urls)
    for q in (name, f"{name} about", f"{name} customers reviews"):
        if len(urls) >= max_pages * 2:
            break
        try:
            for u in search_fn(q):
                if u not in urls:
                    urls.append(u)
        except Exception as e:  # network, blocking, parsing
            notes.append(f"search failed for '{q}': {e}")
    pages = []
    for u in urls:
        if len(pages) >= max_pages:
            break
        try:
            text = fetch_fn(u)
        except Exception as e:
            notes.append(f"could not fetch {u}: {e}")
            continue
        if len(text) < 200:
            notes.append(f"{u}: too little text (likely needs JavaScript or a login)")
            continue
        pages.append(Page(f"S{len(pages) + 1}", u, text[:12000]))
    return pages, notes

````

## `tests/__init__.py`

````

````

## `tests/test_agent.py`

````
import unittest
from research_agent.sources import Page, html_to_text, parse_ddg, gather
from research_agent.brief import verify_claim, ask_model, build, render
from research_agent.llm import MockClient
from research_agent.__main__ import run

TEXT = ("Example Bakery is a family bakery in Sharjah. We sell sourdough bread, croissants and "
        "custom birthday cakes. Order online for delivery across Sharjah and Dubai. Follow us on Instagram. " * 3)
PAGES = [Page("S1", "https://example.test/a", TEXT)]


def claim(text, quote, src="S1"):
    return {"text": text, "source": src, "quote": quote}


class T(unittest.TestCase):
    def test_html_text_skips_scripts(self):
        self.assertEqual(html_to_text("<p>Hi</p><script>x=1</script><b>there</b>"), "Hi there")

    def test_parse_ddg(self):
        h = '<a class="result__a" href="//duckduckgo.com/l/?uddg=https%3A%2F%2Fa.test%2Fx&rut=1">A</a>'
        self.assertEqual(parse_ddg(h), ["https://a.test/x"])

    def test_good_quote_passes_even_with_odd_spacing(self):
        ok, _ = verify_claim(claim("Sells sourdough", "we sell  sourdough bread,\n croissants"), PAGES)
        self.assertTrue(ok)

    def test_fabricated_quote_fails(self):
        ok, why = verify_claim(claim("Has 40 stores", "operates forty stores across the UAE"), PAGES)
        self.assertFalse(ok); self.assertIn("not found", why)

    def test_unknown_source_fails(self):
        ok, why = verify_claim(claim("x", "family bakery in Sharjah", "S9"), PAGES)
        self.assertFalse(ok); self.assertIn("not fetched", why)

    def test_invented_number_fails(self):
        ok, why = verify_claim(claim("Founded in 1985", "family bakery in Sharjah"), PAGES)
        self.assertFalse(ok); self.assertIn("number", why)

    def test_short_quote_fails(self):
        self.assertFalse(verify_claim(claim("x", "bakery"), PAGES)[0])

    def test_bad_json_retries_once_then_raises(self):
        with self.assertRaises(ValueError):
            ask_model(MockClient(["nope", "still nope"]), "X", PAGES)
        self.assertEqual(ask_model(MockClient(["nope", '{"sells":[]}']), "X", PAGES), {"sells": []})

    def test_unverified_goes_to_flag_section(self):
        data = {"sells": [claim("Sells bread", "We sell sourdough bread"), claim("Has 40 stores", "forty stores everywhere")],
                "customers": [], "online_presence": [],
                "ideas": [{"idea": "Subscription box", "why": "repeat orders", "basis": "S1"},
                          {"idea": "Wholesale", "why": "cafes", "basis": "S7"}]}
        ok, bad, ideas = build("Example Bakery", PAGES, data)
        md = render("Example Bakery", PAGES, ok, bad, ideas, ["search failed for 'x': 403"])
        self.assertIn("Sells bread [S1]", md)
        self.assertNotIn("40 stores [S1]", md)
        self.assertIn("Has 40 stores", md.split("## Could not verify")[1])
        self.assertIn("[basis not verified]", md)
        self.assertIn("Run note: search failed", md)
        self.assertIn("Nothing verified", md)  # empty sections say so

    def test_gather_records_failures(self):
        def bad_search(q): raise OSError("blocked")
        def fetch(u): return TEXT if "good" in u else ""
        pages, notes = gather("X", ["https://good.test", "https://empty.test"], bad_search, fetch)
        self.assertEqual(len(pages), 1)
        self.assertTrue(any("search failed" in n for n in notes))
        self.assertTrue(any("too little text" in n for n in notes))

    def test_no_pages_means_no_brief(self):
        with self.assertRaises(SystemExit):
            run("X", MockClient([]), [], lambda q: [], lambda u: "")

    def test_end_to_end_with_mock(self):
        import json
        reply = json.dumps({"sells": [claim("Sells sourdough and croissants", "sourdough bread, croissants")],
                            "customers": [], "online_presence": [claim("Delivers online", "Order online for delivery")],
                            "ideas": [{"idea": "Corporate breakfast boxes", "why": "offices in Dubai", "basis": "S1"}]})
        md = run("Example Bakery", MockClient([reply]), ["https://example.test/a"], lambda q: [], lambda u: TEXT)
        self.assertIn("Delivers online [S1]", md)
        self.assertIn("## Sources", md)


if __name__ == "__main__":
    unittest.main()

````
