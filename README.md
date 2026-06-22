# FitFindr — Starter Kit

This starter kit contains everything you need to begin Project 2.

## What's Included

```
ai201-project2-fitfindr-starter/
├── data/
│   ├── listings.json          # 40 mock secondhand listings
│   └── wardrobe_schema.json   # Wardrobe format + example wardrobe
├── utils/
│   └── data_loader.py         # Helper functions for loading the data
├── planning.md                # Your planning template — fill this out first
└── requirements.txt           # Python dependencies
```

## Setup

**macOS / Linux:**
```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

**Windows:**
```bash
python -m venv .venv
source .venv/Scripts/activate
pip install -r requirements.txt
```

Set your Groq API key in a `.env` file (get a free key at [console.groq.com](https://console.groq.com)):
```
GROQ_API_KEY=your_key_here
```

## The Mock Listings Dataset

`data/listings.json` contains 40 mock secondhand listings across categories (tops, bottoms, outerwear, shoes, accessories) and styles (vintage, y2k, grunge, cottagecore, streetwear, and more).

Each listing has: `id`, `title`, `description`, `category`, `style_tags`, `size`, `condition`, `price`, `colors`, `brand`, and `platform`.

Load it with:
```python
from utils.data_loader import load_listings
listings = load_listings()
```

## The Wardrobe Schema

`data/wardrobe_schema.json` defines the format your agent uses to represent a user's existing wardrobe. It includes:

- `schema`: field definitions for a wardrobe item
- `example_wardrobe`: a sample wardrobe with 10 items you can use for testing
- `empty_wardrobe`: a starting template for a new user

Load an example wardrobe with:
```python
from utils.data_loader import get_example_wardrobe
wardrobe = get_example_wardrobe()
```

## Tool Inventory

### `search_listings(description, size, max_price)`

Searches the mock listings dataset for secondhand items matching the user's keywords, optional size, and optional price ceiling. Scores each listing by keyword overlap across title, description, style tags, category, and brand, then returns results sorted by relevance.

| Parameter | Type | Meaning |
|---|---|---|
| `description` | `str` | Keywords describing the item (e.g. "vintage graphic tee") |
| `size` | `str \| None` | Size to filter by, case-insensitive substring match. Pass `None` to skip. |
| `max_price` | `float \| None` | Maximum price inclusive. Pass `None` to skip. |

**Returns:** `list[dict]` sorted by relevance score. Each dict contains: `id`, `title`, `description`, `category`, `style_tags`, `size`, `condition`, `price`, `colors`, `brand`, `platform`. Returns `[]` on no match (never raises).

---

### `suggest_outfit(new_item, wardrobe)`

Calls the Groq LLM (llama-3.3-70b-versatile) to suggest 1-2 complete outfit combinations pairing the new thrifted item with pieces from the user's wardrobe. Falls back to general styling advice if the wardrobe is empty.

| Parameter | Type | Meaning |
|---|---|---|
| `new_item` | `dict` | A listing dict returned by `search_listings` |
| `wardrobe` | `dict` | Wardrobe dict with an `items` key containing wardrobe item dicts. May be empty. |

**Returns:** `str` with outfit suggestions. Never empty (falls back to general advice or a hardcoded string if the LLM call fails).

---

### `create_fit_card(outfit, new_item)`

Calls the Groq LLM at temperature 0.9 to generate a 2-4 sentence casual Instagram/TikTok-style caption for the outfit. Guards against empty input before calling the LLM.

| Parameter | Type | Meaning |
|---|---|---|
| `outfit` | `str` | Outfit suggestion string from `suggest_outfit` |
| `new_item` | `dict` | The listing dict for the thrifted item |

**Returns:** `str` caption mentioning the item name, price, and platform. If `outfit` is empty or whitespace, returns `"Unable to generate a fit card — no outfit suggestion was provided."` immediately.

---

## How the Planning Loop Works

`run_agent()` in `agent.py` orchestrates the three tools through a session dict. There is one conditional branch: after `search_listings`, if results are empty the agent sets an error and returns immediately without calling the remaining tools.

Step by step:

1. Initialize the session dict with `_new_session(query, wardrobe)`.
2. Use the Groq LLM to parse the natural language query into `{description, size, max_price}`. Store in `session["parsed"]`. Fall back to `{description: query, size: None, max_price: None}` if parsing fails.
3. Call `search_listings(description, size, max_price)`. Store results in `session["search_results"]`.
   - If results are empty: set `session["error"]` and return the session. `suggest_outfit` and `create_fit_card` are never called.
   - If results are not empty: continue.
4. Set `session["selected_item"] = session["search_results"][0]`.
5. Call `suggest_outfit(selected_item, wardrobe)`. Store in `session["outfit_suggestion"]`.
6. Call `create_fit_card(outfit_suggestion, selected_item)`. Store in `session["fit_card"]`.
7. Return the session.

## State Management

All state lives in a single `session` dict created at the start of each `run_agent()` call. Each tool's output is written into the session before the next tool reads from it. No state persists between separate calls.

| Key | Set in | Used in | Contains |
|---|---|---|---|
| `session["parsed"]` | Step 2 | Step 3 | `{description, size, max_price}` from LLM parse |
| `session["search_results"]` | Step 3 | Step 3 (branch check) | Full list of matching listing dicts |
| `session["selected_item"]` | Step 4 | Steps 5 and 6 | Top listing dict (`results[0]`) |
| `session["wardrobe"]` | Step 1 | Step 5 | User's wardrobe dict |
| `session["outfit_suggestion"]` | Step 5 | Step 6 | Outfit suggestion string from LLM |
| `session["fit_card"]` | Step 6 | Returned to caller | Caption string from LLM |
| `session["error"]` | Step 3 (on failure) | Checked by caller | Error message string, or `None` on success |

## Interaction Walkthrough

**User query:** "vintage graphic tee under $30"

**Step 1 — Query parsing**
- Tool: Groq LLM (inside `run_agent`)
- Input: `"vintage graphic tee under $30"`
- Why: Extracts structured parameters from natural language without requiring a form.
- Output: `{"description": "vintage graphic tee", "size": null, "max_price": 30.0}`

**Step 2 — `search_listings("vintage graphic tee", size=None, max_price=30.0)`**
- Input: description, size=None, max_price=30.0
- Why: Retrieval step that grounds the interaction in actual available inventory.
- Output: Ranked list of matching listings. Top result: Y2K Baby Tee at $18 from Depop.

**Step 3 — `suggest_outfit(selected_item, wardrobe)`**
- Input: Y2K Baby Tee dict, example wardrobe (10 items)
- Why: The wardrobe context is what makes the suggestion personal rather than generic.
- Output: 1-2 outfit suggestions referencing specific wardrobe pieces (baggy jeans, chunky sneakers).

**Step 4 — `create_fit_card(outfit_suggestion, selected_item)`**
- Input: outfit suggestion string, Y2K Baby Tee dict
- Why: Needs the full outfit context to write a specific, authentic caption.
- Output: 2-4 sentence Instagram-style caption mentioning the item, price, and platform.

**Final output to user:**
- Panel 1: Item title, price, platform, condition, size, style tags
- Panel 2: Outfit suggestions referencing specific wardrobe pieces
- Panel 3: Shareable fit card caption

---

## Error Handling and Fail Points

| Tool | Failure mode | Agent response |
|------|-------------|----------------|
| `search_listings` | No listings match the query | `session["error"]` is set to `"No listings found for '[description]'... Try broader keywords, a different size, or a higher budget."` The agent returns early. The user sees this in the first output panel; the other two panels are blank. Tested with `search_listings("designer ballgown", size="XXS", max_price=5)` which returned `[]` with no exception. |
| `suggest_outfit` | `wardrobe["items"]` is empty | The LLM is prompted for general styling advice instead of wardrobe-specific pairings. Returns a non-empty string. Tested by calling `suggest_outfit(item, get_empty_wardrobe())` directly and confirming general outfit ideas were returned without an exception. |
| `create_fit_card` | `outfit` is empty or whitespace | Returns `"Unable to generate a fit card — no outfit suggestion was provided."` immediately without calling the LLM. Tested by calling `create_fit_card("", item)` and confirming the exact error string was returned. |

---

## Spec Reflection

**One way planning.md helped during implementation:**
Writing out the state management table before touching `agent.py` made the planning loop almost mechanical to implement. Every session key name was already decided, so there were no naming decisions to make mid-implementation. It also made verification straightforward: each step in the code maps directly to a row in the table.

**One divergence from the spec:**
The spec described LLM query parsing as a straightforward step, but the LLM occasionally wraps its JSON response in markdown code fences. The implementation added a JSON extraction step using `find("{")` and `rfind("}")` that was not in the original spec. This was a small but necessary defensive measure discovered during testing.

---

## AI Usage

**Instance 1 — `suggest_outfit` implementation:**
Claude was given the Tool 2 spec block from `planning.md` (inputs, return value, failure mode) along with the wardrobe schema structure and the TODO steps from the stub. It produced a working implementation that correctly branched on `wardrobe["items"]` being empty. The prompt structure was then simplified: the generated version used a multi-turn message format which was unnecessary, so it was reduced to a single user message per call.

**Instance 2 — `run_agent` planning loop:**
Claude was given the Architecture diagram and both the Planning Loop and State Management sections from `planning.md`, along with the `_new_session()` function and the TODO steps from `agent.py`. The generated code correctly branched on empty search results and used the right session key names. One override was made: the generated version used a separate helper function to build the error message string, which was flattened inline to reduce unnecessary abstraction.

---

## Where to Start

1. **Read `planning.md` and fill it out before writing any code.**
2. Verify the data loads correctly by running `python utils/data_loader.py`.
3. Build and test each tool individually before connecting them through your planning loop.

Your implementation files go in this same directory. There's no required file structure for your agent code — organize it however makes sense for your design.
