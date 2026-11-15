# Paste-into-GitHub guide

Add file > Create new file, type the path in the heading, paste the code, Commit.

## 1. `README.md`

~~~~~
# Your first tool-using AI

A short script where an AI model can call **one function**, `lookup_price`, which reads a price from a local file (`prices.csv`). Standard library only: no installs.

```bash
python tool_agent.py "How much is the steel flask?" --mock          # offline demo, no key needed
export ANTHROPIC_API_KEY=...                                         # for the real model
python tool_agent.py "How much is the steel flask?"
python -m unittest discover -s tests -t .                            # 14 offline tests
```

Output of the offline demo (the `--mock` client stands in for the model):

```
[turn 1] model stop_reason = tool_use
         model asks: lookup_price({"product_name": "steel flask"})
         we ran it -> {"found": true, "product": "Steel flask 500 ml", "price_aed": 59.0, "in_stock": true}
[turn 2] model stop_reason = end_turn

ANSWER: Steel flask 500 ml costs AED 59.00 and is in stock.
```

## The key idea

**The model never runs your function.** It can only *ask* for it, in a structured message. Your code runs the function and sends the result back. The model then reads the result and writes its answer. Everything below is that one idea, step by step.

## The tool-use loop, step by step

The loop is `run_agent()` in `tool_agent.py`.

**Step 0. Write the function and describe it.**
`lookup_price(product_name)` reads `prices.csv`. `TOOLS` describes it to the model: a name, a plain-English description of *when* to use it, and a JSON schema for its input. The model only ever sees this description, never your code.

**Step A. Send the question and the tool list.**
`client.create(messages, TOOLS, SYSTEM)` sends the customer's question plus the tool description to the model.

**Step B. The model decides.**
If it can answer without help, it replies with text and `stop_reason: "end_turn"`. If it needs the price, it replies with a `tool_use` block, `{"name": "lookup_price", "input": {"product_name": "steel flask"}, "id": "toolu_..."}`, and `stop_reason: "tool_use"`. That means: *please run this and tell me what happened.* Nothing has run yet.

**Step C. Your code runs the function.**
`_run_tool()` looks the name up in `FUNCTIONS`, calls it with the model's arguments, and captures the result. If anything goes wrong (unknown tool, bad arguments, missing file) it returns a short `Error: ...` with `is_error: true` instead of crashing, so the model can react.

**Step D. Send the result back.**
The reply goes into the conversation as a `tool_result` block that carries the same `tool_use_id`, so the model can match answer to request. The history now reads: user question, assistant `tool_use`, user `tool_result`. The API requires that exact order.

**Step E. The model answers.**
The loop calls the model again with the longer history. This time it has the price, so it writes the final sentence and `stop_reason` is `end_turn`. The loop returns that text. If it asked for another tool instead, the loop would go round again.

```
question ──> model ──tool_use──> your code runs lookup_price ──tool_result──> model ──text──> answer
               ^                                                               |
               └─────────────── repeat while stop_reason == "tool_use" ────────┘
```

## Safety rules built into the loop

- **Turn limit** (`max_turns=5`): a model that keeps calling tools cannot loop forever.
- **Errors go back to the model as results,** not as crashes, and never as a raw traceback.
- **The function is read-only.** The model can look things up but cannot change anything.
- **Treat the model's arguments as untrusted input.** `lookup_price` checks its argument and only reads one fixed file, so the model cannot ask it to open another.
- **The system prompt says "answer only from tool results"** so the model does not invent a price when the lookup finds nothing.

## Files

| File | What it is |
| --- | --- |
| `tool_agent.py` | The function, the tool description, the loop, the real API client and the offline mock |
| `prices.csv` | The local file the function reads |
| `tests/test_agent.py` | 14 offline tests: the lookup, the loop, error handling, the turn limit, the shape of the history |

## Limits

- **Not run against the live API.** The real client is tested only against a local fake server, which checks the request shape. Run it once with your key before relying on it.
- The `--mock` client is a small script, not a language model. It exists to show the loop and to test it offline.
- One tool, one product per question. To add a tool, add its description to `TOOLS` and its function to `FUNCTIONS`.
~~~~~

## 2. `.gitignore`

```
__pycache__/
*.pyc
```

## 3. `prices.csv`

```
product,price_aed,in_stock
AirRun 2 running shoes,349.00,yes
Steel flask 500 ml,59.00,yes
Cotton crew tee,45.00,no
USB-C charger 65W,119.00,yes
Wireless earbuds Pro,499.00,yes
```

## 4. `tests/__init__.py`

```

```

## 5. `tests/test_agent.py`

```
import json
import threading
import unittest
from http.server import BaseHTTPRequestHandler, HTTPServer

import tool_agent as t


class Lookup(unittest.TestCase):
    def test_exact(self): self.assertEqual(t.lookup_price("Steel flask 500 ml")["price_aed"], 59.0)
    def test_partial_and_case(self): self.assertEqual(t.lookup_price("steel flask")["product"], "Steel flask 500 ml")
    def test_out_of_stock(self): self.assertFalse(t.lookup_price("cotton crew tee")["in_stock"])
    def test_not_found_lists_products(self):
        r = t.lookup_price("laptop"); self.assertFalse(r["found"]); self.assertIn("USB-C charger 65W", r["available_products"])
    def test_bad_input(self):
        with self.assertRaises(ValueError): t.lookup_price("  ")


class Loop(unittest.TestCase):
    def test_tool_is_called_then_answer(self):
        out = t.run_agent("How much is the steel flask?", t.MockClient(), verbose=False)
        self.assertIn("AED 59.00", out); self.assertIn("in stock", out)

    def test_no_tool_needed(self):
        self.assertIn("prices", t.run_agent("Hello", t.MockClient(), verbose=False))

    def test_history_shape(self):  # the API needs assistant tool_use then a user tool_result with the same id
        seen = []
        class Spy(t.MockClient):
            def create(self, messages, tools, system): seen.append(json.loads(json.dumps(messages))); return super().create(messages, tools, system)
        t.run_agent("price of the steel flask", Spy(), verbose=False)
        last = seen[-1]
        self.assertEqual([m["role"] for m in last], ["user", "assistant", "user"])
        self.assertEqual(last[1]["content"][0]["id"], last[2]["content"][0]["tool_use_id"])

    def test_unknown_tool_becomes_error_result(self):
        r = t._run_tool({"id": "x", "name": "delete_everything", "input": {}}, False)
        self.assertTrue(r["is_error"]); self.assertIn("unknown tool", r["content"])

    def test_bad_arguments_become_error_result(self):
        r = t._run_tool({"id": "x", "name": "lookup_price", "input": {"wrong": 1}}, False)
        self.assertTrue(r["is_error"])

    def test_turn_limit(self):
        class Forever:
            def create(self, m, tl, s): return {"stop_reason": "tool_use", "content": [{"type": "tool_use", "id": "i", "name": "lookup_price", "input": {"product_name": "x"}}]}
        self.assertIn("turn limit", t.run_agent("q", Forever(), max_turns=3, verbose=False))

    def test_not_found_answer(self):
        self.assertIn("could not find", t.run_agent("price of the laptop", t.MockClient(), verbose=False))


class Backend(unittest.TestCase):
    """The real client against a local fake server: checks the request shape only."""
    def test_request_shape(self):
        got = {}
        class H(BaseHTTPRequestHandler):
            def do_POST(self):
                got["body"] = json.loads(self.rfile.read(int(self.headers["content-length"]))); got["key"] = self.headers["x-api-key"]
                out = json.dumps({"stop_reason": "end_turn", "content": [{"type": "text", "text": "ok"}]}).encode()
                self.send_response(200); self.send_header("content-length", str(len(out))); self.end_headers(); self.wfile.write(out)
            def log_message(self, *a): pass
        srv = HTTPServer(("127.0.0.1", 0), H); threading.Thread(target=srv.handle_request, daemon=True).start()
        import os; os.environ["ANTHROPIC_API_KEY"] = "k"; os.environ["ANTHROPIC_BASE_URL"] = f"http://127.0.0.1:{srv.server_port}"
        r = t.AnthropicClient().create([{"role": "user", "content": "hi"}], t.TOOLS, t.SYSTEM)
        self.assertEqual(r["stop_reason"], "end_turn"); self.assertEqual(got["key"], "k")
        self.assertEqual(got["body"]["tools"][0]["name"], "lookup_price"); self.assertIn("system", got["body"])

    def test_missing_key(self):
        import os; os.environ.pop("ANTHROPIC_API_KEY", None)
        with self.assertRaises(RuntimeError): t.AnthropicClient().create([], t.TOOLS, t.SYSTEM)


if __name__ == "__main__":
    unittest.main()
```

## 6. `tool_agent.py`

```
"""A first tool-using AI: the model may call ONE function, lookup_price, which reads prices.csv.

    python tool_agent.py "How much are the steel flasks?" --mock      # offline, no API key
    ANTHROPIC_API_KEY=... python tool_agent.py "How much are the steel flasks?"

Standard library only. The loop is in run_agent(); read it first.
"""
from __future__ import annotations

import argparse
import csv
import json
import os
import sys
import time
import urllib.error
import urllib.request
from pathlib import Path

PRICE_FILE = Path(__file__).with_name("prices.csv")

# ---------------------------------------------------------------- 1. the function ---
def lookup_price(product_name: str, price_file: Path = PRICE_FILE) -> dict:
    """Read-only lookup in a local CSV. Returns a dict the model can read."""
    if not isinstance(product_name, str) or not product_name.strip():
        raise ValueError("product_name must be a non-empty string")
    wanted = product_name.strip().lower()
    with open(price_file, newline="", encoding="utf-8") as f:
        rows = list(csv.DictReader(f))
    for row in rows:  # exact match first, then "the query is part of the name"
        if row["product"].lower() == wanted:
            return _found(row)
    for row in rows:
        if wanted in row["product"].lower() or row["product"].lower() in wanted:
            return _found(row)
    return {"found": False, "message": "no product with that name", "available_products": [r["product"] for r in rows]}


def _found(row: dict) -> dict:
    return {"found": True, "product": row["product"], "price_aed": float(row["price_aed"]), "in_stock": row["in_stock"] == "yes"}


# ---------------------------------------------------------------- 2. the tool description the model sees ---
TOOLS = [{
    "name": "lookup_price",
    "description": "Look up the current price (in AED) and stock status of one product in the shop's price list. "
                   "Use it whenever the customer asks about a price or whether something is in stock. Never guess a price.",
    "input_schema": {
        "type": "object",
        "properties": {"product_name": {"type": "string", "description": "Product name as the customer wrote it, e.g. 'steel flask'"}},
        "required": ["product_name"],
    },
}]
FUNCTIONS = {"lookup_price": lookup_price}
SYSTEM = "You are a shop assistant. Answer only from tool results. If the tool finds nothing, say so and list what is available."


# ---------------------------------------------------------------- 3. the loop ---
def run_agent(question: str, client, max_turns: int = 5, verbose: bool = True) -> str:
    messages = [{"role": "user", "content": question}]
    for turn in range(1, max_turns + 1):
        reply = client.create(messages, TOOLS, SYSTEM)                      # STEP A: ask the model
        messages.append({"role": "assistant", "content": reply["content"]})  # keep its reply in the history
        if verbose:
            print(f"[turn {turn}] model stop_reason = {reply['stop_reason']}")
        if reply["stop_reason"] != "tool_use":                               # STEP E: no tool wanted, so this is the answer
            return "".join(b["text"] for b in reply["content"] if b["type"] == "text")
        results = []
        for block in reply["content"]:
            if block["type"] != "tool_use":
                continue
            if verbose:
                print(f"         model asks: {block['name']}({json.dumps(block['input'])})")
            results.append(_run_tool(block, verbose))                        # STEP B + C: run it ourselves
        messages.append({"role": "user", "content": results})                # STEP D: hand the results back
    return "Stopped: the model kept calling tools and hit the turn limit."


def _run_tool(block: dict, verbose: bool) -> dict:
    func = FUNCTIONS.get(block["name"])
    try:
        if func is None:
            raise KeyError(f"unknown tool {block['name']!r}")
        output, is_error = json.dumps(func(**block["input"])), False
    except Exception as exc:  # the model never sees a traceback, only a short error it can react to
        output, is_error = f"Error: {exc}", True
    if verbose:
        print(f"         we ran it -> {output}")
    return {"type": "tool_result", "tool_use_id": block["id"], "content": output, **({"is_error": True} if is_error else {})}


# ---------------------------------------------------------------- model clients ---
class AnthropicClient:
    """Real Messages API call. Needs ANTHROPIC_API_KEY. Not run live in this repo's tests."""

    def __init__(self, model="claude-haiku-5-5", max_tokens=600):
        self.model, self.max_tokens = model, max_tokens
        self.key = os.environ.get("ANTHROPIC_API_KEY", "")
        self.base = os.environ.get("ANTHROPIC_BASE_URL", "https://api.anthropic.com").rstrip("/")

    def create(self, messages, tools, system):
        if not self.key:
            raise RuntimeError("ANTHROPIC_API_KEY is not set (use --mock to run offline)")
        body = json.dumps({"model": self.model, "max_tokens": self.max_tokens, "system": system,
                           "tools": tools, "messages": messages}).encode()
        req = urllib.request.Request(f"{self.base}/v1/messages", data=body, headers={
            "x-api-key": self.key, "anthropic-version": "2023-06-01", "content-type": "application/json"})
        for attempt in range(4):
            try:
                with urllib.request.urlopen(req, timeout=120) as r:
                    return json.load(r)
            except urllib.error.HTTPError as e:
                if e.code in (429, 500, 502, 503, 529) and attempt < 3:
                    time.sleep(2 ** attempt)
                    continue
                raise RuntimeError(f"API error {e.code}: {e.read().decode()[:300]}") from e


class MockClient:
    """Offline stand-in that behaves like the API for this one job, so the loop can run with no key.
    It asks for the tool when the question mentions price/cost/stock, then writes the answer from the tool result.
    It is NOT a language model."""

    def create(self, messages, tools, system):
        last = messages[-1]
        if isinstance(last["content"], str):
            q = last["content"].lower()
            if any(w in q for w in ("price", "cost", "how much", "stock")):
                name = q.replace("?", "").split(" the ")[-1].split(" for ")[-1].strip()
                return {"stop_reason": "tool_use", "content": [
                    {"type": "tool_use", "id": "toolu_mock_1", "name": "lookup_price", "input": {"product_name": name}}]}
            return {"stop_reason": "end_turn", "content": [{"type": "text", "text": "I can help with prices and stock."}]}
        result = last["content"][0]
        if result.get("is_error"):
            text = "Sorry, I could not look that up."
        else:
            data = json.loads(result["content"])
            if data["found"]:
                text = f"{data['product']} costs AED {data['price_aed']:.2f} and is {'in stock' if data['in_stock'] else 'out of stock'}."
            else:
                text = "I could not find that product. We have: " + ", ".join(data["available_products"]) + "."
        return {"stop_reason": "end_turn", "content": [{"type": "text", "text": text}]}


def main(argv=None) -> int:
    ap = argparse.ArgumentParser(description="Ask the shop assistant about a price.")
    ap.add_argument("question")
    ap.add_argument("--mock", action="store_true", help="use the offline mock instead of the real API")
    ap.add_argument("--model", default="claude-haiku-5-5")
    a = ap.parse_args(argv)
    client = MockClient() if a.mock else AnthropicClient(a.model)
    print("\nANSWER:", run_agent(a.question, client))
    return 0


if __name__ == "__main__":
    sys.exit(main())
```
