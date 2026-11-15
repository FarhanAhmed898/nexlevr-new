# Paste into GitHub

Create each file below (Add file > Create new file, type the path in the name box).

## `README.md`

````
# Agent with four guardrails

A tool-using agent loop wrapped in four limits that are enforced **in code**, so the model cannot talk its way past them.

| # | Guardrail | Where | What happens when it trips |
|---|-----------|-------|----------------------------|
| 1 | Step limit (`max_steps`) | `run()` loop | run ends: `STOPPED: step limit` |
| 2 | Allowed tools (`allowed_tools`) | tool list shown to the model **and** checked again at execution | unlisted tool never offered; if requested anyway it returns `Blocked` |
| 3 | Cost cap (`max_cost_usd`) | checked before every model call, from token usage | run ends: `STOPPED: cost cap` |
| 4 | Human approval (`needs_approval`) | checked at execution, after the allow-list | denied (or no approver) = tool does not run |

Default policy: `lookup_price` is allowed and read-only. `send_email` and `publish_post` need approval and are off the allow-list until you add them. `delete_file` exists in the schema list but is never allowed.

## Run
```
python3 demo.py                                   # one short demo per guardrail, offline
python3 -m unittest discover -s tests -t .        # 10 tests
```
To use with a real model, pass any client with `create(messages, tools)` returning an Anthropic-style reply (`content`, `stop_reason`, `usage`), and use `cli_approver` for real yes/no prompts.

## Honest notes
- The demo and tests use a **scripted mock**, not a language model, and were **not run against the live API**.
- "Posts, sends, publishes" are **simulated**: an approved action appends to `outbox.jsonl`. Nothing leaves the machine. Wiring a real email or social API is a separate step and should keep the same approval check.
- Prices per million tokens in `guardrails.py` are **assumptions**. Replace them with your model's real prices. The cap is checked before each call, so one call can overshoot it by that call's cost.
- Order matters: a tool that is not allowed never reaches the human, so people are not asked to approve things that were never permitted.

````

## `agent.py`

````
"""Tool-using agent with guardrails. Real side effects are simulated: approved sends and posts
are appended to outbox.jsonl, nothing leaves the machine."""
import json
import sys
from pathlib import Path

from guardrails import Guardrails

HERE = Path(__file__).parent
OUTBOX = HERE / "outbox.jsonl"


def lookup_price(product_name):
    import csv
    for row in csv.DictReader(open(HERE / "prices.csv", encoding="utf-8")):
        if product_name.lower() in row["product"].lower():
            return {"found": True, "product": row["product"], "price_aed": float(row["price_aed"])}
    return {"found": False}


def send_email(to, subject, body):
    _out({"kind": "email", "to": to, "subject": subject, "body": body}); return {"sent": True}


def publish_post(text):
    _out({"kind": "post", "text": text}); return {"published": True}


def _out(rec):
    with open(OUTBOX, "a") as f:
        f.write(json.dumps(rec) + "\n")


FUNCS = {"lookup_price": lookup_price, "send_email": send_email, "publish_post": publish_post}
SCHEMAS = {
    "lookup_price": {"name": "lookup_price", "description": "Price in AED of a product.",
                     "input_schema": {"type": "object", "properties": {"product_name": {"type": "string"}}, "required": ["product_name"]}},
    "send_email": {"name": "send_email", "description": "Send an email (needs human approval).",
                   "input_schema": {"type": "object", "properties": {"to": {"type": "string"}, "subject": {"type": "string"}, "body": {"type": "string"}}, "required": ["to", "subject", "body"]}},
    "publish_post": {"name": "publish_post", "description": "Publish a social post (needs human approval).",
                     "input_schema": {"type": "object", "properties": {"text": {"type": "string"}}, "required": ["text"]}},
    "delete_file": {"name": "delete_file", "description": "Delete a file.",
                    "input_schema": {"type": "object", "properties": {"path": {"type": "string"}}, "required": ["path"]}},
}


def run(task, client, g: Guardrails):
    # Guardrail 2a: the model is only shown tools on the allowed list.
    tools = [SCHEMAS[n] for n in SCHEMAS if g.permitted(n)]
    messages = [{"role": "user", "content": task}]
    for step in range(1, g.max_steps + 1):            # Guardrail 1: step limit
        if g.over_budget():                           # Guardrail 3: cost cap, checked before every call
            g.note("stopped_cost", spent=round(g.spent_usd, 4))
            return f"STOPPED: cost cap ${g.max_cost_usd:.3f} reached (spent ${g.spent_usd:.4f})."
        reply = client.create(messages, tools)
        g.add_usage(reply.get("usage", {}))
        messages.append({"role": "assistant", "content": reply["content"]})
        if reply["stop_reason"] != "tool_use":
            return "".join(b.get("text", "") for b in reply["content"])
        results = []
        for b in (b for b in reply["content"] if b["type"] == "tool_use"):
            ok, why = g.check(b["name"], b["input"])  # Guardrails 2b and 4: enforced at execution
            if ok:
                try:
                    out, err = json.dumps(FUNCS[b["name"]](**b["input"])), False
                except Exception as e:
                    out, err = f"Error: {e}", True
            else:
                out, err = f"Blocked: {why}", True
            results.append({"type": "tool_result", "tool_use_id": b["id"], "content": out, **({"is_error": True} if err else {})})
        messages.append({"role": "user", "content": results})
    g.note("stopped_steps", steps=g.max_steps)
    return f"STOPPED: step limit ({g.max_steps}) reached."


def cli_approver(name, args):
    print(f"\nAPPROVAL NEEDED: {name} {json.dumps(args)}")
    return input("Type 'yes' to allow: ").strip().lower() == "yes"

````

## `demo.py`

````
"""Short demo of each guardrail. Offline: a scripted mock plays the model.   python3 demo.py"""
import agent
from agent import run
from guardrails import Guardrails
from mock import Script

agent.OUTBOX.unlink(missing_ok=True)
def banner(t): print(f"\n=== {t} ===")

banner("1. STEP LIMIT: model loops forever on lookup_price, stopped at 3 steps")
g = Guardrails(max_steps=3)
print(run("price?", Script([("lookup_price", {"product_name": "flask"})]), g))

banner("2. ALLOWED TOOLS: model tries delete_file, which was never offered")
g = Guardrails()
print(run("clean up", Script([("delete_file", {"path": "/"}), "Could not delete."]), g))
print("log:", g.log)

banner("3. COST CAP: $0.003/step against a $0.005 cap")
g = Guardrails(max_cost_usd=0.005, price_in_per_m=1.0, price_out_per_m=5.0)
print(run("price?", Script([("lookup_price", {"product_name": "flask"})], tokens=(1000, 400)), g))

banner("4a. HUMAN APPROVAL: denied, nothing published")
g = Guardrails(allowed_tools=frozenset({"lookup_price", "publish_post"}), approve=lambda n, a: False)
print(run("post it", Script([("publish_post", {"text": "Sale today!"}), "Not posted."]), g))
print("outbox exists:", agent.OUTBOX.exists())

banner("4b. HUMAN APPROVAL: approved, simulated post written to outbox.jsonl")
g = Guardrails(allowed_tools=frozenset({"lookup_price", "publish_post"}), approve=lambda n, a: True)
print(run("post it", Script([("publish_post", {"text": "Sale today!"}), "Posted."]), g))
print("outbox:", agent.OUTBOX.read_text().strip())
agent.OUTBOX.unlink(missing_ok=True)

````

## `guardrails.py`

````
"""Four guardrails around a tool-using agent. Enforced in code, not by asking the model nicely.

1. Step limit      max_steps model calls per run
2. Allowed tools   only these tools are shown to the model AND only these can run
3. Cost cap        estimated spend (from token usage) may not pass max_cost_usd
4. Human approval  tools in `needs_approval` run only if a person says yes
"""
from dataclasses import dataclass, field
from typing import Callable, Optional


@dataclass
class Guardrails:
    max_steps: int = 6
    allowed_tools: frozenset = frozenset({"lookup_price"})
    needs_approval: frozenset = frozenset({"send_email", "publish_post"})
    max_cost_usd: float = 0.05
    # ASSUMED prices (USD per million tokens). Replace with your model's real prices.
    price_in_per_m: float = 1.0
    price_out_per_m: float = 5.0
    approve: Optional[Callable[[str, dict], bool]] = None  # None = deny everything
    spent_usd: float = 0.0
    log: list = field(default_factory=list)

    def note(self, event, **kw):
        self.log.append({"event": event, **kw})

    # 3. cost
    def add_usage(self, usage: dict):
        self.spent_usd += (usage.get("input_tokens", 0) * self.price_in_per_m
                           + usage.get("output_tokens", 0) * self.price_out_per_m) / 1e6

    def over_budget(self) -> bool:
        return self.spent_usd >= self.max_cost_usd

    # 2. allow-list (side-effect tools must also be allowed to be proposed at all)
    def permitted(self, name: str) -> bool:
        return name in self.allowed_tools

    # 4. approval
    def check(self, name: str, args: dict):
        """Return (ok, reason). Order: allow-list first, then human approval."""
        if not self.permitted(name):
            self.note("blocked_not_allowed", tool=name)
            return False, f"tool '{name}' is not on the allowed list"
        if name in self.needs_approval:
            ok = bool(self.approve and self.approve(name, args))
            self.note("approved" if ok else "denied", tool=name, args=args)
            if not ok:
                return False, "a human denied this action (or no approver was available)"
        return True, ""

````

## `mock.py`

````
"""Scripted stand-in for the model (NOT a language model). Replays a list of replies and reports fake token usage."""
class Script:
    def __init__(self, steps, tokens=(500, 200)):
        self.steps, self.i, self.tokens = steps, 0, tokens

    def create(self, messages, tools):
        s = self.steps[min(self.i, len(self.steps) - 1)]; self.i += 1
        usage = {"input_tokens": self.tokens[0], "output_tokens": self.tokens[1]}
        if isinstance(s, str):
            return {"stop_reason": "end_turn", "content": [{"type": "text", "text": s}], "usage": usage}
        name, args = s
        return {"stop_reason": "tool_use", "usage": usage,
                "content": [{"type": "tool_use", "id": f"t{self.i}", "name": name, "input": args}]}

````

## `prices.csv`

````
product,price_aed,in_stock
AirRun 2 running shoes,349.00,yes
Steel flask 500 ml,59.00,yes
Cotton crew tee,45.00,no
USB-C charger 65W,119.00,yes
Wireless earbuds Pro,499.00,yes

````

## `tests/__init__.py`

````

````

## `tests/test_guardrails.py`

````
import sys, unittest
from pathlib import Path
sys.path.insert(0, str(Path(__file__).parent.parent))
import agent
from agent import run
from guardrails import Guardrails
from mock import Script

LOOK = ("lookup_price", {"product_name": "flask"})
POST = ("publish_post", {"text": "hi"})


class T(unittest.TestCase):
    def setUp(self): agent.OUTBOX.unlink(missing_ok=True)
    tearDown = setUp

    def test_happy_path(self):
        out = run("p", Script([LOOK, "It costs 59."]), Guardrails())
        self.assertEqual(out, "It costs 59.")

    def test_step_limit(self):
        g = Guardrails(max_steps=3); s = Script([LOOK])
        self.assertIn("step limit", run("p", s, g)); self.assertEqual(s.i, 3)

    def test_unlisted_tool_not_shown_and_blocked(self):
        seen = []
        class Spy(Script):
            def create(self, m, tools): seen.append([t["name"] for t in tools]); return super().create(m, tools)
        g = Guardrails()
        run("x", Spy([("delete_file", {"path": "/"}), "ok"]), g)
        self.assertEqual(seen[0], ["lookup_price"])
        self.assertEqual(g.log[0]["event"], "blocked_not_allowed")

    def test_cost_cap_stops_before_next_call(self):
        g = Guardrails(max_cost_usd=0.005); s = Script([LOOK], tokens=(1000, 400))  # $0.003 per call
        self.assertIn("cost cap", run("p", s, g)); self.assertEqual(s.i, 2)
        self.assertLess(g.spent_usd, 0.01)

    def test_cost_math(self):
        g = Guardrails(); g.add_usage({"input_tokens": 1_000_000, "output_tokens": 1_000_000})
        self.assertAlmostEqual(g.spent_usd, 6.0)

    def test_approval_denied_means_no_side_effect(self):
        g = Guardrails(allowed_tools=frozenset({"lookup_price", "publish_post"}), approve=lambda n, a: False)
        run("p", Script([POST, "ok"]), g)
        self.assertFalse(agent.OUTBOX.exists()); self.assertEqual(g.log[0]["event"], "denied")

    def test_no_approver_defaults_to_deny(self):
        g = Guardrails(allowed_tools=frozenset({"lookup_price", "publish_post"}))
        run("p", Script([POST, "ok"]), g)
        self.assertFalse(agent.OUTBOX.exists())

    def test_approval_granted_runs_once(self):
        asked = []
        g = Guardrails(allowed_tools=frozenset({"lookup_price", "publish_post"}), approve=lambda n, a: asked.append((n, a)) or True)
        run("p", Script([POST, "done"]), g)
        self.assertEqual(len(asked), 1); self.assertEqual(len(agent.OUTBOX.read_text().splitlines()), 1)

    def test_approval_tool_not_allowed_never_asks_human(self):
        asked = []
        g = Guardrails(approve=lambda n, a: asked.append(n) or True)  # publish_post not on allow-list
        run("p", Script([POST, "ok"]), g)
        self.assertEqual(asked, []); self.assertFalse(agent.OUTBOX.exists())

    def test_lookup_needs_no_approval(self):
        asked = []
        run("p", Script([LOOK, "ok"]), Guardrails(approve=lambda n, a: asked.append(n) or True))
        self.assertEqual(asked, [])


if __name__ == "__main__":
    unittest.main()

````
