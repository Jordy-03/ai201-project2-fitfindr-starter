# FitFindr — planning.md

> Complete this document before writing any implementation code.
> Your spec and agent diagram are what you'll use to direct AI tools (Claude, Copilot, etc.) to generate your implementation — the more specific they are, the more useful the generated code will be.
> Your planning.md will be reviewed as part of your submission.
> Update it before starting any stretch features.

---

## Tools

List every tool your agent will use. For each tool, fill in all four fields.
You must have at least 3 tools. The three required tools are listed — add any additional tools below them.

### Tool 1: search_listings

**What it does:**
Searches the mock listings dataset for secondhand items that match the user's keywords, optional size, and optional price ceiling. Returns a ranked list of matching listing dicts sorted by keyword relevance score (highest first).

**Input parameters:**
- `description` (str): Keywords describing what the user is looking for (e.g., "vintage graphic tee"). Matched case-insensitively against each listing's title, description, style_tags, category, and brand fields.
- `size` (str | None): Size string to filter by (e.g., "M", "XL"). Case-insensitive substring match against listing's size field. Pass None to skip size filtering.
- `max_price` (float | None): Maximum price inclusive (e.g., 30.0). Listings with price > max_price are excluded. Pass None to skip price filtering.

**What it returns:**
A list of listing dicts sorted by relevance score, highest first. Each dict contains: `id` (str), `title` (str), `description` (str), `category` (str), `style_tags` (list[str]), `size` (str), `condition` (str), `price` (float), `colors` (list[str]), `brand` (str | None), `platform` (str). Returns an empty list `[]` if nothing matches — never raises an exception.

**What happens if it fails or returns nothing:**
If the returned list is empty, `run_agent()` sets `session["error"]` to: `"No listings found for '[description]'[with size filter][under $max_price]. Try broader keywords, a different size, or a higher budget."` and returns the session immediately. `suggest_outfit` and `create_fit_card` are never called with empty input.

---

### Tool 2: suggest_outfit

**What it does:**
Calls the Groq LLM (llama-3.3-70b-versatile) to suggest 1–2 complete outfit combinations that pair the new thrifted item with pieces from the user's existing wardrobe. If the wardrobe is empty, returns general styling advice instead of crashing.

**Input parameters:**
- `new_item` (dict): A listing dict returned by `search_listings` — the item the user is considering buying. Key fields used in the prompt: `title`, `category`, `style_tags`, `colors`, `condition`.
- `wardrobe` (dict): A wardrobe dict with an `'items'` key containing a list of wardrobe item dicts. Each wardrobe item has: `id`, `name`, `category`, `colors`, `style_tags`, `notes`. May be empty — `wardrobe['items']` can be `[]`.

**What it returns:**
A non-empty string with 1–2 outfit suggestions. If wardrobe is populated, suggestions reference specific named pieces from the wardrobe. If wardrobe is empty, returns general styling advice about what kinds of items pair well with the new piece and what vibe it suits.

**What happens if it fails or returns nothing:**
If `wardrobe['items']` is empty, the LLM is prompted for general styling advice rather than wardrobe-specific pairings — the function never returns an empty string for this case. If the LLM call raises an exception, the function catches it and returns a fallback string: `"This [category] has a [style_tags[0]] vibe — try pairing it with straight-leg jeans and clean sneakers for a classic look."`.

---

### Tool 3: create_fit_card

**What it does:**
Calls the Groq LLM to generate a short, casual, shareable 2–4 sentence outfit caption — the kind someone would post as an Instagram or TikTok OOTD. Uses a higher temperature (0.9) to ensure varied output across different inputs.

**Input parameters:**
- `outfit` (str): The outfit suggestion string returned by `suggest_outfit`. Must be non-empty.
- `new_item` (dict): The listing dict for the thrifted item. Key fields used: `title`, `price`, `platform`, `style_tags`.

**What it returns:**
A 2–4 sentence string written in a casual, authentic first-person voice. The caption mentions the item name, price, and platform naturally (once each) and captures the outfit's specific vibe. Returns a different caption each call for different inputs due to higher LLM temperature. If `outfit` is empty or whitespace-only, returns the error string `"Unable to generate a fit card — no outfit suggestion was provided."` without calling the LLM.

**What happens if it fails or returns nothing:**
If `outfit` is empty or whitespace-only, returns the error string above immediately. If the LLM call raises an exception, catches it and returns: `"Fit card unavailable — try again in a moment."`.

---

### Additional Tools (if any)

<!-- Copy the block above for any tools beyond the required three -->

---

## Planning Loop

**How does your agent decide which tool to call next?**

The planning loop in `run_agent()` follows this specific conditional logic:

1. **Initialize session** — call `_new_session(query, wardrobe)` to create the session dict.

2. **Parse the query** — use the Groq LLM to extract three fields from the natural language query: `description` (str), `size` (str or null), `max_price` (float or null). Prompt the LLM to return a JSON object with exactly those three keys. Store the result in `session["parsed"]`. If parsing fails, fall back to using the full query string as `description` with `size=None` and `max_price=None`.

3. **Call search_listings** — call `search_listings(description, size, max_price)` using the parsed fields. Store the result in `session["search_results"]`.
   - **Branch A (empty results):** If `session["search_results"] == []`, set `session["error"]` to a helpful message (see Tool 1 failure mode above) and **return the session immediately** — do not call `suggest_outfit` or `create_fit_card`.
   - **Branch B (results found):** Proceed to step 4.

4. **Select top item** — set `session["selected_item"] = session["search_results"][0]`.

5. **Call suggest_outfit** — call `suggest_outfit(session["selected_item"], session["wardrobe"])`. Store the result in `session["outfit_suggestion"]`.

6. **Call create_fit_card** — call `create_fit_card(session["outfit_suggestion"], session["selected_item"])`. Store the result in `session["fit_card"]`.

7. **Return session** — return the completed session dict.

The loop only branches at step 3. There are no other conditional exits. All other tool calls (suggest_outfit, create_fit_card) run unconditionally once search_listings succeeds, because their individual failure modes are handled inside the tools themselves (they return strings, never raise).

---

## State Management

**How does information from one tool get passed to the next?**

All state lives in a single `session` dict initialized by `_new_session()` at the start of each `run_agent()` call. The dict is mutated in-place at each step:

| Session key | Set in step | Used in step | Contains |
|---|---|---|---|
| `session["query"]` | Step 1 | Step 2 | Original natural language string |
| `session["parsed"]` | Step 2 | Step 3 | `{description, size, max_price}` extracted by LLM |
| `session["search_results"]` | Step 3 | Step 3 branch check | Full list of matching listing dicts |
| `session["selected_item"]` | Step 4 | Steps 5 & 6 | The top listing dict (results[0]) |
| `session["wardrobe"]` | Step 1 | Step 5 | User's wardrobe dict (passed into run_agent) |
| `session["outfit_suggestion"]` | Step 5 | Step 6 | Outfit suggestion string from LLM |
| `session["fit_card"]` | Step 6 | Returned to caller | Caption string from LLM |
| `session["error"]` | Step 3A | Checked by caller | Error message string, or None on success |

The session dict is returned at the end of `run_agent()`. `app.py`'s `handle_query()` reads `session["selected_item"]`, `session["outfit_suggestion"]`, and `session["fit_card"]` to populate the three Gradio output panels. It checks `session["error"]` first — if set, it displays the error in the top panel and leaves the other two blank.

No state persists between separate `run_agent()` calls. Each call starts fresh with `_new_session()`.

---

## Error Handling

For each tool, describe the specific failure mode you're handling and what the agent does in response.

| Tool | Failure mode | Agent response |
|------|-------------|----------------|
| search_listings | No results match the query | `session["error"]` is set to `"No listings found for '[description]'[with size filter][under $X]. Try broader keywords, a different size, or a higher budget."` The agent returns the session immediately without calling the remaining tools. The user sees this message in the top output panel. |
| suggest_outfit | Wardrobe is empty (`wardrobe['items'] == []`) | The LLM is called with a prompt asking for general styling advice (what pairs well, what vibe it suits) rather than wardrobe-specific combinations. Returns a non-empty string — no crash, no empty output. |
| create_fit_card | Outfit input is empty or whitespace-only | Returns the string `"Unable to generate a fit card — no outfit suggestion was provided."` immediately, without calling the LLM. This prevents a useless LLM call and gives the user a clear signal. |

---

## Architecture

```
User Query (natural language)
    │
    ▼
run_agent(query, wardrobe)
    │
    ▼
Step 1: _new_session(query, wardrobe)
    │   session["query"] = query
    │   session["wardrobe"] = wardrobe
    │
    ▼
Step 2: LLM parse query → {description, size, max_price}
    │   session["parsed"] = {description, size, max_price}
    │
    ▼
Step 3: search_listings(description, size, max_price)
    │   session["search_results"] = [...]
    │
    ├── results == [] ──► session["error"] = "No listings found..."
    │                              │
    │                              ▼
    │                         RETURN session   ◄─── error path ends here
    │
    │   results not empty
    ▼
Step 4: session["selected_item"] = results[0]
    │
    ▼
Step 5: suggest_outfit(selected_item, wardrobe)
    │   session["outfit_suggestion"] = "..."
    │
    │   (if wardrobe empty → LLM gives general advice, no crash)
    │
    ▼
Step 6: create_fit_card(outfit_suggestion, selected_item)
    │   session["fit_card"] = "..."
    │
    │   (if outfit empty → returns error string, no crash)
    │
    ▼
RETURN session
    │
    ▼
handle_query() in app.py reads session fields
    ├── session["error"] set?  → show error in panel 1, leave panels 2 & 3 blank
    └── session["error"] None? → populate all 3 output panels
```

---

## AI Tool Plan

**Milestone 3 — Individual tool implementations:**

**Tool 1 (search_listings):** Give Claude the Tool 1 spec block above (inputs, return value, failure mode) plus the TODO steps from the `search_listings` stub in `tools.py`. Ask it to implement the function using `load_listings()` from `utils/data_loader.py`. Before running the generated code, verify: (1) it filters by all three parameters, (2) it scores by keyword overlap across title/description/style_tags/category/brand fields, (3) it returns `[]` not raises on no results. Test with three queries: a good match ("vintage graphic tee", no size, max_price=50), an impossible match ("designer ballgown", "XXS", 5.0), and a price-only filter ("jacket", None, 10.0).

**Tool 2 (suggest_outfit):** Give Claude the Tool 2 spec block above plus the `wardrobe_schema.json` structure and the TODO steps from the stub. Ask it to implement using Groq `llama-3.3-70b-versatile` with `GROQ_API_KEY` from `.env`. Verify before running: (1) checks `wardrobe['items']` for empty, (2) uses different prompts for empty vs. populated wardrobe, (3) returns a non-empty string in both cases. Test with `get_example_wardrobe()` and `get_empty_wardrobe()`.

**Tool 3 (create_fit_card):** Give Claude the Tool 3 spec block above and the TODO steps from the stub. Ask it to implement using Groq with `temperature=0.9`. Verify: (1) guards against empty/whitespace `outfit`, (2) returns error string not raises, (3) caption mentions item name, price, platform. Run the same inputs three times to confirm the outputs vary.

**Milestone 4 — Planning loop and state management:**

Give Claude the full Architecture diagram above and both the Planning Loop and State Management sections. Also give it the `_new_session()` function and the TODO steps from `run_agent()` in `agent.py`. Ask it to implement `run_agent()`. Before running: verify (1) it branches on empty `search_results`, (2) it stores each tool's output in the session dict by the exact key names in the State Management table, (3) it does NOT call `suggest_outfit` or `create_fit_card` when `search_results` is empty. Test both the happy path and the no-results path from the `__main__` block in `agent.py`.

---

## A Complete Interaction (Step by Step)

FitFindr takes a natural language query from the user and orchestrates three tools in sequence: it first searches a mock dataset of secondhand listings (`search_listings`), then asks the LLM to suggest outfit combinations using the found item and the user's wardrobe (`suggest_outfit`), and finally generates a shareable caption for the outfit (`create_fit_card`). Each tool is triggered by the successful output of the previous one — if `search_listings` returns no results, the agent stops immediately and tells the user what to try differently, without calling the remaining tools.

**Example user query:** "I'm looking for a vintage graphic tee under $30. I mostly wear baggy jeans and chunky sneakers. What's out there and how would I style it?"

**Step 1 — Parse query:**
The agent calls the LLM to extract structured parameters from the query.
- Input: the full user query string
- LLM output: `{"description": "vintage graphic tee", "size": null, "max_price": 30.0}`
- `session["parsed"]` = `{"description": "vintage graphic tee", "size": None, "max_price": 30.0}`

**Step 2 — Search listings:**
`search_listings("vintage graphic tee", size=None, max_price=30.0)` is called.
- Loads all 40 listings, filters to those priced ≤ $30.
- Scores remaining listings by keyword overlap: "vintage", "graphic", "tee" matched against title, description, style_tags, category, brand.
- Returns e.g. `[{"id": "lst_002", "title": "Y2K Baby Tee — Butterfly Print", "price": 18.0, "platform": "depop", "style_tags": ["y2k", "vintage", "graphic tee"], ...}, ...]`
- `session["search_results"]` = 2–3 matching items. Results not empty → continue.

**Step 3 — Select top item:**
`session["selected_item"]` = `results[0]` → the Y2K Baby Tee at $18 on Depop.

**Step 4 — Suggest outfit:**
`suggest_outfit(new_item=<Y2K Baby Tee dict>, wardrobe=<example wardrobe>)` is called.
- Wardrobe has 10 items including baggy straight-leg jeans, chunky white sneakers, black crossbody bag.
- LLM prompt includes the item details and wardrobe item names.
- Returns: `"Pair this Y2K baby tee with your baggy straight-leg jeans and chunky white sneakers for a throwback 2000s street look. Tuck the tee slightly and add your black crossbody bag to finish it off."`
- `session["outfit_suggestion"]` = the above string.

**Step 5 — Create fit card:**
`create_fit_card(outfit=<suggestion string>, new_item=<Y2K Baby Tee dict>)` is called.
- LLM generates a casual OOTD caption at temperature 0.9.
- Returns: `"found this y2k butterfly tee on depop for $18 and i genuinely can't stop wearing it 🦋 baggy jeans + chunky sneakers and the fit is just there. full look on my stories"`
- `session["fit_card"]` = the above caption.

**Final output to user:**
- **Panel 1 (Top listing):** "Y2K Baby Tee — Butterfly Print | $18.00 | depop | Condition: excellent | Size: S/M"
- **Panel 2 (Outfit idea):** "Pair this Y2K baby tee with your baggy straight-leg jeans and chunky white sneakers for a throwback 2000s street look. Tuck the tee slightly and add your black crossbody bag to finish it off."
- **Panel 3 (Fit card):** "found this y2k butterfly tee on depop for $18 and i genuinely can't stop wearing it 🦋 baggy jeans + chunky sneakers and the fit is just there. full look on my stories"
