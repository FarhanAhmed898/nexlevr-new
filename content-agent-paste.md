# Paste into GitHub

Create each file below (Add file > Create new file, type the path in the name box).

## `README.md`

````
# Multi-step content agent

Planner -> Researcher -> Writer -> Reviewer. It produces a weekly content pack for one brand (a blog draft, 3 social posts, a newsletter), logs every tool call, and the reviewer rejects claims that have no source.

```
python3 demo.py                                   # offline, scripted; writes out/demo/
python3 -m unittest discover -s tests -t .        # 14 tests
export ANTHROPIC_API_KEY=...
python3 run.py "Your brand" --sources sources --out out/week-1   # real model calls
```
Standard library only. Put your own brand notes as `.md` files in `sources/` (one fact per line).

## The four steps
| Step | Does | Guardrail |
|------|------|-----------|
| Planner | picks the angle, 3 to 4 research questions, one brief per deliverable | JSON only |
| Researcher | the model chooses tool calls; code runs them | only `search_notes` and `read_source` are allowed, both read-only and limited to `sources/`; anything else is blocked and logged |
| Writer | writes the pack from the notes only, with a note id like `[N2]` after each factual sentence | cannot see anything but the notes |
| Reviewer | **code, not a model**: checks every claim sentence | see below; one revision round, then anything still flagged is marked NEEDS A HUMAN CHECK in the files |

**Tool log:** every call (agent, tool, arguments, ok or failed, result size) is appended to `tool_calls.jsonl` and summarised in `review.md`. Blocked and failed calls are logged too.

**What the reviewer catches:** a sentence counts as a claim if it has a number, a `%`, or words like *only, first, best, proven, faster, lighter, injuries, studies*. A claim fails if it has (1) no `[N#]` citation, (2) a citation to a note that does not exist, (3) a number that is not in the cited note, or (4) a cited note that shares no wording with it.

## What was tested, honestly
- The 14 offline tests and the demo pass. The demo uses a **scripted client, not a language model**: I wrote draft 1 with four planted problems (an unsourced "clinically proven" claim, a made-up citation, a wrong number, a claim the source does not support) and the reviewer caught all four. Draft 2 is also scripted, so the demo shows the reviewer and the logging working, **not the quality of real writing**.
- **Never run against the live API** (no key in the build sandbox). Expect to tune prompts on the first real run.
- Brand facts in `sources/` are **invented demo data** for a fictional "Smart Cleats" concept. `examples/demo-pack/` is the demo output, not research.

## Limits
- The reviewer checks that a claim is sourced and matches the note's numbers and words. It cannot tell if a source is true, or catch a sentence that reuses a real note's words to say something else. A person should read the pack before it is posted.
- A short claim sentence with one shared word can pass. Opinion and tone are not checked.
- Nothing is posted or sent. Publishing would need the approval step from the guardrails project.

````

## `demo.py`

````
"""Offline demo. A SCRIPTED client plays all four agents' model calls (it is not a language model).
Draft 1 contains four planted problems so you can watch the reviewer catch them.   python3 demo.py"""
import json
import shutil
from content_agent.llm import ScriptedClient
from content_agent.pipeline import run

PLAN = {"angle": "Walk off the pitch without changing shoes",
        "questions": ["How do the studs retract?", "When can people buy?", "Who is it for?"],
        "briefs": {"blog": "Explain the retractable studs", "social": "Three short posts", "newsletter": "Waitlist update"}}
CALLS = {"calls": [
    {"tool": "search_notes", "args": {"query": "retractable studs walking hard floors"}},
    {"tool": "search_notes", "args": {"query": "waitlist prototype run pairs"}},
    {"tool": "search_notes", "args": {"query": "players switch outdoor indoor courts weight grams"}},
    {"tool": "delete_source", "args": {"name": "brand_facts.md"}}]}   # not an allowed tool: must be blocked and logged

DRAFT1 = {
 "blog": "## Walk off the pitch without changing shoes\nSmart Cleats is a concept football boot with retractable spikes [N1]. The studs retract by 4 mm so the sole sits flat on tile [N3]. Our boots are clinically proven to cut injuries by 40%. Amateur players often switch between outdoor pitches and indoor courts in one week [N7].",
 "posts": ["Studs out on grass, studs in for the changing room. Retract them by 4 mm [N1].",
           "Our boots are lighter than any rival [N9].",
           "Join the waitlist. The first prototype run is planned for 200 pairs [N12]."],
 "newsletter": "Thanks for joining us. The first prototype run is 300 pairs [N4]. Only the waitlist is open for now [N5]."}

DRAFT2 = {
 "blog": "## Walk off the pitch without changing shoes\nSmart Cleats is a concept football boot with retractable spikes [N1]. The studs retract by 4 mm so the sole sits flat on tile [N3]. Amateur players often switch between outdoor pitches and indoor courts in one week [N7].",
 "posts": ["Studs out on grass, studs in for the changing room. Retract them by 4 mm [N1].",
           "Design target weight is 290 grams per boot in a size 42 [N9].",
           "Join the waitlist. The first prototype run is planned for 200 pairs [N4]."],
 "newsletter": "Thanks for joining us. The first prototype run is planned for 200 pairs [N4]. Only the waitlist is open for now [N5]."}

client = ScriptedClient({"planner": [json.dumps(PLAN)], "researcher": [json.dumps(CALLS)],
                         "writer": [json.dumps(DRAFT1), json.dumps(DRAFT2)]})
shutil.rmtree("out/demo", ignore_errors=True)
r = run("Smart Cleats (demo facts)", "2026-11-16", client, "sources", "out/demo")
print(f"Caught in draft 1: {len(r['first_issues'])}")
for i in r["first_issues"]:
    print(f"  ({i['where']}) {i['reason']}: {i['sentence'][:70]}")
print(f"Still flagged after revision: {len(r['final_issues'])}")
print(f"Tool calls logged: {len(r['log'])}  (see out/demo/tool_calls.jsonl)")

````

## `run.py`

````
"""python3 __main__.py "Brand name" --sources sources --out out/week-1   (needs ANTHROPIC_API_KEY)"""
import argparse
import datetime
from content_agent.llm import AnthropicClient
from content_agent.pipeline import run

ap = argparse.ArgumentParser()
ap.add_argument("brand"); ap.add_argument("--sources", default="sources"); ap.add_argument("--out", default="out/latest")
a = ap.parse_args()
r = run(a.brand, datetime.date.today().isoformat(), AnthropicClient(), a.sources, a.out)
print(f"Wrote {a.out}. Caught in draft 1: {len(r['first_issues'])}. Still flagged: {len(r['final_issues'])}.")

````

## `content_agent/__init__.py`

````

````

## `content_agent/agents.py`

````
import json
import re

PLANNER = ("You plan one week of content for a brand. Return JSON only: "
           '{"angle":"...","questions":["3 to 4 factual questions to research"],'
           '"briefs":{"blog":"...","social":"...","newsletter":"..."}}')
RESEARCHER = ('You research with tools. Available: search_notes{"query"}, read_source{"name"}. '
              'Return JSON only: {"calls":[{"tool":"search_notes","args":{"query":"..."}}]}. At most 6 calls.')
WRITER = ("Write the weekly content pack from the research notes ONLY. After every factual sentence, add the note id "
          "like [N2]. If the notes do not support a claim, leave it out. No medical or injury claims. "
          'Return JSON only: {"blog":"~200 words markdown","posts":["p1","p2","p3"],"newsletter":"~120 words"}.')


def _json(raw):
    m = re.search(r"\{.*\}", raw, re.S)
    if not m:
        raise ValueError(f"no JSON in model reply: {raw[:120]!r}")
    return json.loads(m.group(0))


def plan(client, brand, week):
    return _json(client.complete("planner", PLANNER, f"Brand: {brand}\nWeek of: {week}"))


def research(client, toolbox, plan_):
    """The model chooses calls; code runs them through the allowed ToolBox; notes keep exact source text."""
    calls = _json(client.complete("researcher", RESEARCHER, json.dumps(plan_["questions"])))["calls"][:6]
    notes, errors = {}, []
    for c in calls:
        out, err = toolbox.call("researcher", c.get("tool", ""), c.get("args", {}))
        if err:
            errors.append(f"{c.get('tool')}: {err}")
            continue
        for item in out:
            if not any(n["text"] == item["text"] for n in notes.values()):
                notes[f"N{len(notes) + 1}"] = item
    return notes, errors


def write(client, plan_, notes, issues=None, previous=None):
    ctx = "\n".join(f"[{k}] ({v['source']}) {v['text']}" for k, v in notes.items())
    user = f"Plan: {json.dumps(plan_['briefs'])}\n\nNotes:\n{ctx}"
    if issues:
        user += ("\n\nYour previous draft:\n" + json.dumps(previous) + "\n\nThe reviewer rejected these sentences. "
                 "Fix or remove each one:\n" + "\n".join(f"- ({i['where']}) {i['sentence']} -> {i['reason']}" for i in issues))
    return _json(client.complete("writer", WRITER, user))

````

## `content_agent/llm.py`

````
import json
import os
import urllib.request


class AnthropicClient:
    def __init__(self, model=None):
        self.key = os.environ.get("ANTHROPIC_API_KEY")
        if not self.key:
            raise SystemExit("Set ANTHROPIC_API_KEY, or run demo.py (scripted, offline).")
        self.model = model or os.environ.get("CONTENT_MODEL", "claude-sonnet-5-5")

    def complete(self, role, system, user):
        body = json.dumps({"model": self.model, "max_tokens": 3000, "system": system,
                           "messages": [{"role": "user", "content": user}]}).encode()
        req = urllib.request.Request("https://api.anthropic.com/v1/messages", data=body, headers={
            "x-api-key": self.key, "anthropic-version": "2023-06-01", "content-type": "application/json"})
        with urllib.request.urlopen(req, timeout=180) as r:
            return "".join(b.get("text", "") for b in json.load(r)["content"] if b["type"] == "text")


class ScriptedClient:
    """Replays prepared replies by role. NOT a language model: used for the offline demo and tests."""
    def __init__(self, replies):
        self.replies = {k: list(v) for k, v in replies.items()}
        self.calls = []

    def complete(self, role, system, user):
        self.calls.append(role)
        return self.replies[role].pop(0)

````

## `content_agent/pipeline.py`

````
from pathlib import Path

from . import agents
from .review import review
from .tools import ToolBox, ToolLog


def run(brand, week, client, sources, out_dir, max_revisions=1):
    out = Path(out_dir)
    out.mkdir(parents=True, exist_ok=True)
    log = ToolLog(out / "tool_calls.jsonl")
    box = ToolBox(sources, log)
    plan_ = agents.plan(client, brand, week)
    notes, tool_errors = agents.research(client, box, plan_)
    if not notes:
        raise SystemExit("Researcher found nothing in the sources, so nothing was written.")
    pack = agents.write(client, plan_, notes)
    first_issues = review(pack, notes)
    issues = first_issues
    for _ in range(max_revisions):
        if not issues:
            break
        pack = agents.write(client, plan_, notes, issues, pack)
        issues = review(pack, notes)
    _save(out, brand, week, plan_, notes, pack, first_issues, issues, tool_errors, log)
    return {"pack": pack, "notes": notes, "first_issues": first_issues, "final_issues": issues, "log": log.entries}


def _save(out, brand, week, plan_, notes, pack, first, final, tool_errors, log):
    src = "\n".join(f"- [{k}] {v['source']}: \"{v['text']}\"" for k, v in notes.items())
    banner = "" if not final else "> NEEDS A HUMAN CHECK: the reviewer still flags sentences. See review.md.\n\n"
    (out / "blog.md").write_text(f"{banner}{pack['blog']}\n\n## Sources\n{src}\n")
    (out / "social.md").write_text(banner + "\n\n".join(f"**Post {i}**\n{p}" for i, p in enumerate(pack["posts"], 1)) + f"\n\n## Sources\n{src}\n")
    (out / "newsletter.md").write_text(f"{banner}{pack['newsletter']}\n\n## Sources\n{src}\n")
    def fmt(items): return "\n".join(f"- ({i['where']}) \"{i['sentence']}\": {i['reason']}" for i in items) or "- none"
    (out / "review.md").write_text(
        f"# Review: {brand}, week of {week}\n\nAngle: {plan_['angle']}\n\n"
        f"## Caught in draft 1 ({len(first)})\n{fmt(first)}\n\n## Still flagged after revision ({len(final)})\n{fmt(final)}\n\n"
        f"## Tool calls: {len(log.entries)} logged (see tool_calls.jsonl)\n"
        + "".join(f"{e['n']}. {e['agent']} -> {e['tool']}({e['args']}) {'ok' if e['ok'] else 'FAILED: ' + e['note']}\n" for e in log.entries)
        + ("\nTool errors: " + "; ".join(tool_errors) + "\n" if tool_errors else ""))

````

## `content_agent/review.py`

````
"""The reviewer is code, not a second opinion from the model: it checks every claim sentence
for a citation that exists and that actually supports the numbers and words in the sentence."""
import re

CITE = re.compile(r"\[(N\d+)\]")
CLAIM_WORDS = {"only", "first", "best", "proven", "guarantee", "guaranteed", "faster", "lighter", "reduces",
               "reduce", "prevents", "prevent", "clinically", "tested", "most", "always", "never", "leading",
               "award", "studies", "research", "percent", "safer", "cheaper", "injuries", "injury"}


def sentences(text):
    return [s.strip() for s in re.split(r"(?<=[.!?])\s+|\n+", text) if s.strip()]


def is_claim(s):
    if s.rstrip().endswith("?"):
        return False
    words = set(re.findall(r"[a-z]+", s.lower()))
    return bool(re.search(r"\d", CITE.sub("", s))) or "%" in s or bool(words & CLAIM_WORDS)


def _nums(s):
    return set(re.findall(r"\d+(?:\.\d+)?", CITE.sub("", s)))


def _words(s):
    """Word stems (first 6 letters of words with 5+ letters), so 'retract' matches 'retractable'."""
    return {w[:6] for w in re.findall(r"[a-z]{5,}", CITE.sub("", s).lower())}


def check_text(label, text, notes):
    """notes: {id: {"source","text"}}. Returns a list of issues."""
    issues = []
    for s in sentences(text):
        if not is_claim(s):
            continue
        ids = CITE.findall(s)
        if not ids:
            issues.append({"where": label, "sentence": s, "reason": "claim with no source"})
            continue
        missing = [i for i in ids if i not in notes]
        if missing:
            issues.append({"where": label, "sentence": s, "reason": f"cites {missing[0]}, which does not exist"})
            continue
        support = " ".join(notes[i]["text"] for i in ids)
        bad = [n for n in _nums(s) if n not in _nums(support)]
        if bad:
            issues.append({"where": label, "sentence": s, "reason": f"number {bad[0]} is not in the cited source"})
        elif len(_words(s) & _words(support)) < min(2, max(1, len(_words(s)))):
            issues.append({"where": label, "sentence": s, "reason": "cited source does not say this"})
    return issues


def review(pack, notes):
    issues = check_text("blog", pack["blog"], notes)
    for i, p in enumerate(pack["posts"], 1):
        issues += check_text(f"post {i}", p, notes)
    issues += check_text("newsletter", pack["newsletter"], notes)
    return issues

````

## `content_agent/tools.py`

````
"""Tools the researcher may use. Every call is logged, including blocked and failed ones."""
import json
import re
import time
from pathlib import Path


class ToolLog:
    def __init__(self, path=None):
        self.entries, self.path = [], path
        if path:
            Path(path).parent.mkdir(parents=True, exist_ok=True)
            Path(path).write_text("")

    def add(self, agent, tool, args, ok, result_chars, note=""):
        e = {"n": len(self.entries) + 1, "ts": round(time.time(), 3), "agent": agent, "tool": tool,
             "args": args, "ok": ok, "result_chars": result_chars, "note": note}
        self.entries.append(e)
        if self.path:
            with open(self.path, "a") as f:
                f.write(json.dumps(e) + "\n")


class ToolBox:
    """Allowed tools: search_notes(query), read_source(name). Both read only the sources folder."""
    ALLOWED = {"search_notes", "read_source"}

    def __init__(self, folder, log: ToolLog):
        self.folder, self.log = Path(folder), log

    def call(self, agent, tool, args):
        if tool not in self.ALLOWED:
            self.log.add(agent, tool, args, False, 0, "blocked: not an allowed tool")
            return None, f"tool '{tool}' is not allowed"
        try:
            out = getattr(self, tool)(**args)
            self.log.add(agent, tool, args, True, sum(len(x["text"]) for x in out))
            return out, ""
        except Exception as e:
            self.log.add(agent, tool, args, False, 0, str(e))
            return None, str(e)

    def read_source(self, name):
        p = (self.folder / Path(name).name)  # no path tricks: file name only
        lines = [l.strip() for l in p.read_text().splitlines() if l.strip() and not l.startswith("#")]
        return [{"source": p.name, "text": l} for l in lines]

    def search_notes(self, query):
        words = {w for w in re.findall(r"[a-z0-9]{4,}", query.lower())}
        hits = []
        for p in sorted(self.folder.glob("*.md")):
            for l in self.read_source(p.name):
                score = len(words & set(re.findall(r"[a-z0-9]{4,}", l["text"].lower())))
                if score:
                    hits.append((score, l))
        return [l for _, l in sorted(hits, key=lambda x: -x[0])[:4]]

````

## `examples/demo-pack/blog.md`

````
## Walk off the pitch without changing shoes
Smart Cleats is a concept football boot with retractable spikes [N1]. The studs retract by 4 mm so the sole sits flat on tile [N3]. Amateur players often switch between outdoor pitches and indoor courts in one week [N7].

## Sources
- [N1] brand_facts.md: "Smart Cleats is a concept football boot with retractable spikes. The studs retract by 4 mm for walking on hard floors."
- [N2] market_notes.md: "Traditional boots have fixed studs that damage indoor floors, so many venues ask players to change shoes."
- [N3] customer_faq.md: "Q: Can I walk into the changing room with the studs out? A: Retract them first. The studs retract by 4 mm and the sole sits flat on tile."
- [N4] brand_facts.md: "The first prototype run is planned for 200 pairs, and a waitlist is open on the website."
- [N5] customer_faq.md: "Q: When can I buy a pair? A: Only the waitlist is open. The first prototype run is planned for 200 pairs."
- [N6] customer_faq.md: "Q: Will the boots work on artificial turf? A: The prototype is being tested on natural grass and artificial turf. Test results are not published yet."
- [N7] market_notes.md: "Amateur players often switch between outdoor pitches and indoor courts in one week."
- [N8] brand_facts.md: "The boot is aimed at amateur players aged 16 to 30 who move between grass and indoor courts."
- [N9] brand_facts.md: "The design target weight is 290 grams per boot in a size 42."

````

## `examples/demo-pack/newsletter.md`

````
Thanks for joining us. The first prototype run is planned for 200 pairs [N4]. Only the waitlist is open for now [N5].

## Sources
- [N1] brand_facts.md: "Smart Cleats is a concept football boot with retractable spikes. The studs retract by 4 mm for walking on hard floors."
- [N2] market_notes.md: "Traditional boots have fixed studs that damage indoor floors, so many venues ask players to change shoes."
- [N3] customer_faq.md: "Q: Can I walk into the changing room with the studs out? A: Retract them first. The studs retract by 4 mm and the sole sits flat on tile."
- [N4] brand_facts.md: "The first prototype run is planned for 200 pairs, and a waitlist is open on the website."
- [N5] customer_faq.md: "Q: When can I buy a pair? A: Only the waitlist is open. The first prototype run is planned for 200 pairs."
- [N6] customer_faq.md: "Q: Will the boots work on artificial turf? A: The prototype is being tested on natural grass and artificial turf. Test results are not published yet."
- [N7] market_notes.md: "Amateur players often switch between outdoor pitches and indoor courts in one week."
- [N8] brand_facts.md: "The boot is aimed at amateur players aged 16 to 30 who move between grass and indoor courts."
- [N9] brand_facts.md: "The design target weight is 290 grams per boot in a size 42."

````

## `examples/demo-pack/review.md`

````
# Review: Smart Cleats (demo facts), week of 2026-11-16

Angle: Walk off the pitch without changing shoes

## Caught in draft 1 (4)
- (blog) "Our boots are clinically proven to cut injuries by 40%.": claim with no source
- (post 2) "Our boots are lighter than any rival [N9].": cited source does not say this
- (post 3) "The first prototype run is planned for 200 pairs [N12].": cites N12, which does not exist
- (newsletter) "The first prototype run is 300 pairs [N4].": number 300 is not in the cited source

## Still flagged after revision (0)
- none

## Tool calls: 4 logged (see tool_calls.jsonl)
1. researcher -> search_notes({'query': 'retractable studs walking hard floors'}) ok
2. researcher -> search_notes({'query': 'waitlist prototype run pairs'}) ok
3. researcher -> search_notes({'query': 'players switch outdoor indoor courts weight grams'}) ok
4. researcher -> delete_source({'name': 'brand_facts.md'}) FAILED: blocked: not an allowed tool

Tool errors: delete_source: tool 'delete_source' is not allowed

````

## `examples/demo-pack/social.md`

````
**Post 1**
Studs out on grass, studs in for the changing room. Retract them by 4 mm [N1].

**Post 2**
Design target weight is 290 grams per boot in a size 42 [N9].

**Post 3**
Join the waitlist. The first prototype run is planned for 200 pairs [N4].

## Sources
- [N1] brand_facts.md: "Smart Cleats is a concept football boot with retractable spikes. The studs retract by 4 mm for walking on hard floors."
- [N2] market_notes.md: "Traditional boots have fixed studs that damage indoor floors, so many venues ask players to change shoes."
- [N3] customer_faq.md: "Q: Can I walk into the changing room with the studs out? A: Retract them first. The studs retract by 4 mm and the sole sits flat on tile."
- [N4] brand_facts.md: "The first prototype run is planned for 200 pairs, and a waitlist is open on the website."
- [N5] customer_faq.md: "Q: When can I buy a pair? A: Only the waitlist is open. The first prototype run is planned for 200 pairs."
- [N6] customer_faq.md: "Q: Will the boots work on artificial turf? A: The prototype is being tested on natural grass and artificial turf. Test results are not published yet."
- [N7] market_notes.md: "Amateur players often switch between outdoor pitches and indoor courts in one week."
- [N8] brand_facts.md: "The boot is aimed at amateur players aged 16 to 30 who move between grass and indoor courts."
- [N9] brand_facts.md: "The design target weight is 290 grams per boot in a size 42."

````

## `sources/brand_facts.md`

````
# Smart Cleats: brand facts (DEMO DATA, invented for this exercise)
Smart Cleats is a concept football boot with retractable spikes. The studs retract by 4 mm for walking on hard floors.
The boot is aimed at amateur players aged 16 to 30 who move between grass and indoor courts.
The design target weight is 290 grams per boot in a size 42.
The brand voice is friendly, direct and a little cheeky. It avoids medical or injury claims.
The first prototype run is planned for 200 pairs, and a waitlist is open on the website.

````

## `sources/customer_faq.md`

````
# Customer questions (DEMO DATA)
Q: Can I walk into the changing room with the studs out? A: Retract them first. The studs retract by 4 mm and the sole sits flat on tile.
Q: Will the boots work on artificial turf? A: The prototype is being tested on natural grass and artificial turf. Test results are not published yet.
Q: When can I buy a pair? A: Only the waitlist is open. The first prototype run is planned for 200 pairs.

````

## `sources/market_notes.md`

````
# Market notes (DEMO DATA)
Amateur players often switch between outdoor pitches and indoor courts in one week.
Traditional boots have fixed studs that damage indoor floors, so many venues ask players to change shoes.
Social posts that show a short before-and-after of a feature tend to be easier to follow than long captions.

````

## `tests/__init__.py`

````

````

## `tests/test_content.py`

````
import json, shutil, tempfile, unittest
from pathlib import Path
from content_agent.review import check_text, is_claim
from content_agent.tools import ToolBox, ToolLog
from content_agent.llm import ScriptedClient
from content_agent.pipeline import run

NOTES = {"N1": {"source": "a.md", "text": "The studs retract by 4 mm for walking on hard floors."},
         "N2": {"source": "a.md", "text": "The first prototype run is planned for 200 pairs."}}
SRC = Path(__file__).parent.parent / "sources"


class Review(unittest.TestCase):
    def reasons(self, t): return [i["reason"] for i in check_text("x", t, NOTES)]

    def test_unsourced_claim(self): self.assertEqual(self.reasons("Clinically proven to cut injuries by 40%."), ["claim with no source"])
    def test_good_claim_passes(self): self.assertEqual(self.reasons("The studs retract by 4 mm on hard floors [N1]."), [])
    def test_fabricated_id(self): self.assertIn("does not exist", self.reasons("Runs are 200 pairs [N9].")[0])
    def test_wrong_number(self): self.assertIn("number 300", self.reasons("The first prototype run is 300 pairs [N2].")[0])
    def test_source_does_not_say_it(self): self.assertIn("does not say", self.reasons("Our boots are lighter than any rival [N1].")[0])
    def test_plain_voice_not_flagged(self): self.assertEqual(self.reasons("Join the waitlist today. Friendly and cheeky."), [])
    def test_question_not_a_claim(self): self.assertFalse(is_claim("Why wait 5 minutes?"))
    def test_each_issue_names_its_sentence(self):
        i = check_text("post 2", "Best boot ever. Fine line.", NOTES)[0]
        self.assertEqual((i["where"], i["sentence"]), ("post 2", "Best boot ever."))


class Tools(unittest.TestCase):
    def test_blocked_and_logged(self):
        log = ToolLog(); box = ToolBox(SRC, log)
        out, err = box.call("r", "delete_source", {"name": "x"})
        self.assertIsNone(out); self.assertFalse(log.entries[0]["ok"]); self.assertIn("blocked", log.entries[0]["note"])

    def test_path_trick_stays_inside_folder(self):
        box = ToolBox(SRC, ToolLog()); out, err = box.call("r", "read_source", {"name": "../../etc/passwd"})
        self.assertIsNone(out); self.assertTrue(err)

    def test_every_call_logged_to_file(self):
        d = tempfile.mkdtemp(); log = ToolLog(Path(d) / "l.jsonl"); box = ToolBox(SRC, log)
        box.call("r", "search_notes", {"query": "studs retract"}); box.call("r", "read_source", {"name": "nope.md"})
        lines = (Path(d) / "l.jsonl").read_text().splitlines()
        self.assertEqual([json.loads(l)["ok"] for l in lines], [True, False])


class Pipeline(unittest.TestCase):
    def test_demo_end_to_end(self):
        import runpy, io, contextlib, os
        os.chdir(Path(__file__).parent.parent)
        with contextlib.redirect_stdout(io.StringIO()):
            g = runpy.run_path("demo.py")
        r = g["r"]
        self.assertEqual(len(r["first_issues"]), 4)
        self.assertEqual(r["final_issues"], [])
        self.assertEqual(sum(not e["ok"] for e in r["log"]), 1)   # the blocked delete_source
        self.assertEqual(len(r["log"]), 4)
        review = Path("out/demo/review.md").read_text()
        self.assertIn("claim with no source", review); self.assertNotIn("NEEDS A HUMAN", Path("out/demo/blog.md").read_text())
        shutil.rmtree("out", ignore_errors=True)

    def test_unfixed_claims_are_flagged_in_output(self):
        bad = {"blog": "Proven by studies [N1].", "posts": ["a", "b", "c"], "newsletter": "ok"}
        plan = {"angle": "x", "questions": [], "briefs": {}}
        c = ScriptedClient({"planner": [json.dumps(plan)], "researcher": [json.dumps({"calls": [{"tool": "search_notes", "args": {"query": "studs retract"}}]})],
                            "writer": [json.dumps(bad), json.dumps(bad)]})
        d = tempfile.mkdtemp(); r = run("B", "w", c, SRC, d)
        self.assertTrue(r["final_issues"]); self.assertIn("NEEDS A HUMAN CHECK", (Path(d) / "blog.md").read_text())

    def test_no_notes_no_pack(self):
        plan = {"angle": "x", "questions": [], "briefs": {}}
        c = ScriptedClient({"planner": [json.dumps(plan)], "researcher": [json.dumps({"calls": [{"tool": "search_notes", "args": {"query": "zzzz"}}]})], "writer": []})
        with self.assertRaises(SystemExit): run("B", "w", c, SRC, tempfile.mkdtemp())


if __name__ == "__main__":
    unittest.main()

````
