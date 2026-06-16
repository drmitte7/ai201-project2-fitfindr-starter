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

**What it does:** Searches the mock listings dataset and returns items that match the user's description, size, and maximum price. It filters listings by comparing the description against title, style_tags, and category fields.
<!-- Describe what this tool does in 1–2 sentences -->

**Input parameters:** 
<!-- List each parameter, its type, and what it represents -->
- `description` (str): A plain-language description of the item the user is looking for (e.g. "vintage graphic tee")
- `size` (str): The user's size used to filter listings by the size field. Can be None if the user does not specify a size.
- `max_price` (float): The maximum price the user is willing to pay. Can be None if the user does not specify a price limit.

**What it returns:** A list of matching listing dictionaries sorted by relevance, each containing: id, title, description, category, style_tags, size, condition, price, colors, brand, and platform. Returns an empty list if no matches are found.
<!-- Describe the return value — what fields does a result contain? -->

**What happens if it fails or returns nothing:** The agent stores an error message in the session and returns early without calling suggest_outfit or create_fit_card. The user sees: "No listings matched your search. Try a broader description, a different size, or a higher price limit."
<!-- What should the agent do if no listings match? -->

---

### Tool 2: suggest_outfit

**What it does:** Given a thrifted item and the user's current wardrobe, suggests 1-2 complete outfit combinations using the LLM. If the wardrobe is empty, it offers general styling advice for the item instead.
<!-- Describe what this tool does in 1–2 sentences -->

**Input parameters:**
<!-- List each parameter, its type, and what it represents -->
- `new_item` (dict): A listing dict containing id, title, description, category, style_tags, size, condition, price, colors, brand, and platform.
- `wardrobe` (dict): A wardrobe dict with an 'items' key containing a list of wardrobe item dicts, each with name, category, colors, style_tags, and notes fields.

**What it returns:** A non-empty string with 1-2 specific outfit suggestions referencing named pieces from the wardrobe, or general styling advice if the wardrobe is empty.
<!-- Describe the return value -->

**What happens if it fails or returns nothing:** If the wardrobe is empty, the agent does not crash — it calls the LLM for general styling advice instead. If the LLM call fails, the agent returns the message: "Could not generate outfit suggestions. Please try again."
<!-- What should the agent do if the wardrobe is empty or no outfit can be suggested? -->

---

### Tool 3: create_fit_card

**What it does:** Generates a short 2-4 sentence shareable caption for the thrifted outfit — the kind of thing someone would post on Instagram.
<!-- Describe what this tool does in 1–2 sentences -->

**Input parameters:**
<!-- List each parameter, its type, and what it represents -->
- `outfit` (str): The outfit suggestion string returned by suggest_outfit.
- `new_item` (dict): The listing dict for the thrifted item, used to include the item name, price, and platform naturally in the caption.

**What it returns:** A 2-4 sentence string written in a casual, authentic social media voice that mentions the item name, price, and platform once each.
<!-- Describe the return value -->

**What happens if it fails or returns nothing:** If outfit is empty or whitespace-only, the agent returns the error message: "Could not generate a fit card — outfit description was missing." It does not call the LLM with empty input.
<!-- What should the agent do if the outfit data is incomplete? -->

---

### Additional Tools (if any)

<!-- Copy the block above for any tools beyond the required three -->

---

## Planning Loop

**How does your agent decide which tool to call next?** The planning loop follows this conditional logic:

1. Parse the user query to extract description, size, and max_price using the LLM.
   Store the result in session["parsed"].

2. Call search_listings() with the parsed parameters.
   Store results in session["search_results"].
   IF results is empty:
     - Set session["error"] = "No listings matched your search. Try a broader description, a different size, or a higher price limit."
     - Return the session immediately. Do NOT call suggest_outfit or create_fit_card.
   IF results is not empty:
     - Set session["selected_item"] = results[0] (top result)
     - Continue to step 3.

3. Call suggest_outfit() with session["selected_item"] and session["wardrobe"].
   Store the result in session["outfit_suggestion"].
   IF the result is empty:
     - Set session["error"] = "Could not generate outfit suggestions. Please try again."
     - Return the session immediately.
   IF not empty:
     - Continue to step 4.

4. Call create_fit_card() with session["outfit_suggestion"] and session["selected_item"].
   Store the result in session["fit_card"].
   Return the completed session.
<!-- Describe the logic your planning loop uses. What does it look at? What conditions change its behavior? How does it know when it's done? -->

---

## State Management 

**How does information from one tool get passed to the next?** The agent uses a session dictionary as the single source of truth for all data within one interaction. The session is initialized at the start of run_agent() and passed through each step:

- `session["query"]` — stores the original user query string
- `session["parsed"]` — stores the extracted description, size, and max_price after query parsing
- `session["search_results"]` — stores the full list returned by search_listings()
- `session["selected_item"]` — stores results[0], the top listing, which is passed directly into suggest_outfit()
- `session["wardrobe"]` — stores the user's wardrobe dict, passed directly into suggest_outfit()
- `session["outfit_suggestion"]` — stores the string returned by suggest_outfit(), which is passed directly into create_fit_card()
- `session["fit_card"]` — stores the final caption string returned by create_fit_card()
- `session["error"]` — stores an error message string if any step fails; None on success

No data is re-entered by the user between steps. Each tool receives its inputs directly from the session dict, and writes its output back into the session dict before the next tool is called.
<!-- Describe how your agent stores and accesses state within a session. What data is tracked? How is it passed between tool calls? -->

---

## Error Handling

For each tool, describe the specific failure mode you're handling and what the agent does in response.

| Tool | Failure mode | Agent response |
|------|-------------|----------------|
| search_listings | No results match the query | Sets session["error"] = "No listings matched your search. Try a broader description, a different size, or a higher price limit." Returns session immediately without calling suggest_outfit or create_fit_card. |
| suggest_outfit | Wardrobe is empty | Does not crash — calls the LLM for general styling advice instead of wardrobe-specific combinations. Returns a non-empty string with general styling ideas. |
| create_fit_card | Outfit input is missing or incomplete | Returns the error message string "Could not generate a fit card — outfit description was missing." Does not call the LLM with empty input. |

---

## Architecture

User query
    │
    ▼
Planning Loop (run_agent)
    │
    ├─► Step 1: Parse query (LLM)
    │       │
    │       ▼
    │   session["parsed"] = {description, size, max_price}
    │       │
    ├─► Step 2: search_listings(description, size, max_price)
    │       │
    │       ├── results = []
    │       │       │
    │       │       ▼
    │       │   session["error"] = "No listings matched..."
    │       │   RETURN SESSION EARLY ◄─────────────────────┐
    │       │                                               │
    │       └── results = [item, ...]                       │
    │               │                                       │
    │               ▼                                       │
    │   session["selected_item"] = results[0]               │
    │               │                                       │
    ├─► Step 3: suggest_outfit(selected_item, wardrobe)      │
    │       │                                               │
    │       ├── wardrobe empty → general styling advice     │
    │       │                                               │
    │       └── wardrobe not empty → specific combinations  │
    │               │                                       │
    │               ▼                                       │
    │   session["outfit_suggestion"] = "..."                │
    │               │                                       │
    ├─► Step 4: create_fit_card(outfit_suggestion,          │
    │               selected_item)                          │
    │       │                                               │
    │       ├── outfit empty → error message ───────────────┘
    │       │
    │       └── outfit not empty → caption generated
    │               │
    │               ▼
    │   session["fit_card"] = "..."
    │               │
    ▼               ▼
RETURN COMPLETED SESSION

<!-- Draw a diagram of your agent showing how the components connect:
     User input → Planning Loop → Tools (search_listings, suggest_outfit, create_fit_card)
                                                                          ↕
                                                                   State / Session
     Show what triggers each tool, how state flows between them, and where error paths branch off.
     ASCII art, a Mermaid diagram (https://mermaid.js.org/syntax/flowchart.html), or an embedded
     sketch are all fine. You'll share this diagram with an AI tool when asking it to implement
     the planning loop and each individual tool. -->

---

## AI Tool Plan

<!-- For each part of the implementation below, describe:
     - Which AI tool you plan to use (Claude, Copilot, ChatGPT, etc.)
     - What you'll give it as input (which sections of this planning.md, your agent diagram)
     - What you expect it to produce
     - How you'll verify the output matches your spec before moving on

     "I'll use AI to help me code" is not a plan.
     "I'll give Claude my Tool 1 spec (inputs, return value, failure mode) and ask it to implement
     search_listings() using load_listings() from the data loader — then test it against 3 queries
     before trusting it" is a plan. -->

**Milestone 3 — Individual tool implementations:** I will use Claude for each tool one at a time. For search_listings, I will give Claude the Tool 1 spec block (inputs, return value, failure mode) and ask it to implement the function using load_listings() from the data loader. I will verify by running 3 test queries and checking that results are filtered correctly by price, size, and keyword. For suggest_outfit, I will give Claude the Tool 2 spec block and ask it to implement the LLM call using Groq. I will verify by testing with both an example wardrobe and an empty wardrobe. For create_fit_card, I will give Claude the Tool 3 spec block and ask it to implement the LLM call with higher temperature. I will verify by running it 3 times on the same input and checking that outputs vary.

**Milestone 4 — Planning loop and state management:** I will give Claude the Planning Loop section, State Management section, and Architecture diagram from this planning.md and ask it to implement run_agent() in agent.py. I will verify by checking that: session["selected_item"] is the same dict passed into suggest_outfit, session["outfit_suggestion"] is the same string passed into create_fit_card, and the agent returns early with session["error"] set when search_listings returns an empty list.

---

## A Complete Interaction (Step by Step)

Write out what a full user interaction looks like from start to finish — tool call by tool call. Use a specific example query.

**Example user query:** "I'm looking for a vintage graphic tee under $30. I mostly wear baggy jeans and chunky sneakers. What's out there and how would I style it?"

**Step 1:** The agent parses the query using the LLM and extracts:
- description = "vintage graphic tee"
- size = None (not specified)
- max_price = 30.0
These are stored in session["parsed"] and passed into search_listings().
<!-- What does the agent do first? Which tool is called? With what input? -->

**Step 2:** search_listings("vintage graphic tee", size=None, max_price=30.0) is called.
It returns a list of matching listings under $30 that have "vintage", "graphic", or "tee" in their title, description, category, or style_tags.
The top result is stored in session["selected_item"] — for example:
"Faded Band Tee — $22, Depop, Good condition."
<!-- What happens next? What was returned from step 1? What tool is called now? -->

**Step 3:** suggest_outfit(session["selected_item"], session["wardrobe"]) is called.
The wardrobe has 10 items including baggy jeans and chunky sneakers.
The LLM returns: "Pair this faded band tee with your baggy straight-leg jeans and chunky sneakers for a classic 90s grunge look. Tuck the front corner slightly for shape and roll the sleeves once."
This is stored in session["outfit_suggestion"].
<!-- Continue until the full interaction is complete -->

**Step 4:**
create_fit_card(session["outfit_suggestion"], session["selected_item"]) is called.
The LLM generates a casual Instagram-style caption using the item name, price, and platform.
This is stored in session["fit_card"].

**Final output to user:**
- Top listing panel: "Faded Band Tee — $22 — Good condition — Depop"
- Outfit idea panel: "Pair this faded band tee with your baggy straight-leg jeans and chunky sneakers for a classic 90s grunge look."
- Fit card panel: "thrifted this faded band tee off depop for $22 and it was literally made for my wide-legs 🖤 full look in my stories"
<!-- What does the user actually see at the end? -->

**Error path:**
If search_listings returns an empty list (e.g. query: "designer ballgown size XXS under $5"):
session["error"] = "No listings matched your search. Try a broader description, a different size, or a higher price limit."
The agent returns immediately. suggest_outfit and create_fit_card are never called.