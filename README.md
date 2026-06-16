# FitFindr — AI Thrift Shopping Agent

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

```bash
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

## Where to Start

1. **Read `planning.md` and fill it out before writing any code.**
2. Verify the data loads correctly by running `python utils/data_loader.py`.
3. Build and test each tool individually before connecting them through your planning loop.

Your implementation files go in this same directory. There's no required file structure for your agent code — organize it however makes sense for your design.

---

## Tool Inventory

### Tool 1: search_listings
- **Inputs:** `description` (str), `size` (str or None), `max_price` (float or None)
- **Output:** A list of matching listing dicts sorted by relevance, each containing: id, title, description, category, style_tags, size, condition, price, colors, brand, and platform. Returns an empty list if no matches are found.
- **Purpose:** Searches the mock listings dataset by scoring keyword overlap against title, description, category, and style_tags fields. Filters by price and size before scoring.

### Tool 2: suggest_outfit
- **Inputs:** `new_item` (dict), `wardrobe` (dict)
- **Output:** A non-empty string with 1-2 outfit suggestions referencing named wardrobe pieces, or general styling advice if the wardrobe is empty.
- **Purpose:** Uses the Groq LLM to suggest complete outfit combinations pairing the thrifted item with pieces from the user's existing wardrobe.

### Tool 3: create_fit_card
- **Inputs:** `outfit` (str), `new_item` (dict)
- **Output:** A 2-4 sentence casual Instagram/TikTok style caption mentioning the item name, price, and platform once each.
- **Purpose:** Uses the Groq LLM with higher temperature to generate a varied, authentic-sounding social media caption for the outfit.

---

## How the Planning Loop Works

The planning loop in `run_agent()` follows this conditional logic:

1. The user query is parsed by the LLM to extract description, size, and max_price. These are stored in `session["parsed"]`.
2. `search_listings()` is called with the parsed parameters. If it returns an empty list, `session["error"]` is set and the agent returns immediately — `suggest_outfit` and `create_fit_card` are never called.
3. If results are found, `session["selected_item"]` is set to `results[0]` and `suggest_outfit()` is called.
4. The outfit suggestion is stored in `session["outfit_suggestion"]` and passed into `create_fit_card()`.
5. The completed session is returned with `session["fit_card"]` set.

The agent does not call all three tools unconditionally — it stops early if search returns nothing.

---

## State Management

The agent uses a session dictionary as the single source of truth for all data within one interaction:

- `session["query"]` — original user query
- `session["parsed"]` — extracted description, size, and max_price
- `session["search_results"]` — full list returned by search_listings()
- `session["selected_item"]` — top result, passed directly into suggest_outfit()
- `session["wardrobe"]` — user's wardrobe, passed directly into suggest_outfit()
- `session["outfit_suggestion"]` — string from suggest_outfit(), passed into create_fit_card()
- `session["fit_card"]` — final caption string
- `session["error"]` — error message if any step fails; None on success

No data is re-entered by the user between steps. Each tool reads its inputs from the session and writes its output back into the session.

---

## Error Handling

| Tool | Failure mode | Agent response |
|------|-------------|----------------|
| search_listings | No results match the query | Sets session["error"] and returns immediately without calling the other tools. User sees: "No listings matched your search. Try a broader description, a different size, or a higher price limit." |
| suggest_outfit | Wardrobe is empty | Does not crash — calls the LLM for general styling advice instead of wardrobe-specific combinations. Always returns a non-empty string. |
| create_fit_card | Outfit input is empty or whitespace | Returns "Could not generate a fit card — outfit description was missing." without calling the LLM. |

### Concrete example from testing
When I ran `search_listings("designer ballgown", size="XXS", max_price=5)`, it returned an empty list without raising an exception. The agent set `session["error"]` and returned early. The Gradio interface displayed the error message and the outfit and fit card panels stayed empty.

---

## Spec Reflection

**One way the spec helped during implementation:**
Writing the planning loop section in planning.md before touching agent.py forced me to think through every conditional branch before writing any code. When I implemented run_agent(), I already knew exactly what to check after each tool call because I had described those branches in the spec. This made the implementation much faster and cleaner.

**One way implementation diverged from the spec:**
The spec described query parsing using simple string splitting. During implementation I switched to using the LLM to parse the query instead, because string splitting was not reliable enough for natural language queries like "looking for a vintage tee under thirty dollars." The LLM consistently extracted the right description, size, and price from varied phrasings.

---

## AI Usage

**Instance 1**
- *What I gave the AI:* The Tool 1 spec block from planning.md including inputs, return value, and failure mode. I asked Claude to implement search_listings() using load_listings() from the data loader.
- *What it produced:* A complete function that filters by price and size then scores by keyword overlap.
- *What I changed or overrode:* I directed Claude to split the description string into keywords dynamically so any description would work, not just ones with pre-defined terms.

**Instance 2**
- *What I gave the AI:* The Planning Loop section, State Management section, and Architecture diagram from planning.md. I asked Claude to implement run_agent() in agent.py.
- *What it produced:* A complete planning loop with LLM-based query parsing and conditional branching.
- *What I changed or overrode:* The generated query parser did not handle the case where the LLM returned "None" as a string instead of skipping the field. I added a check for "None" in the size and max_price parsing to handle this correctly.