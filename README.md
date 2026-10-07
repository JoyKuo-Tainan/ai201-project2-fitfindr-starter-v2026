# FitFindr

> ### 👋 Start here
>
> **New to this repo? Read [RUNNING.md](RUNNING.md) first** — setup, every
> command, and what to do when something breaks.
>
> Once `python test.py` passes:
>
> ```bash
> python app.py listings --full -n 6      # read the data (Milestone 1)
> python app.py fields                    # what you can filter on
> python app.py ask 'vintage graphic tee under $30'
> ```
>
> All three tools are stubs, so that last command will do nothing useful yet.
> That's the starting position.
>
> **The rest of this file is your submission.** Fill it in as you go.

---

<!-- ─────────────────────────────────────────────────────────────────────────
     HOW TO USE THIS FILE

     This is your submission. Fill each section in as you finish the milestone
     it belongs to — don't leave it all to the end.

     Unit 3 asks for the first five sections. Unit 4 adds the five below them.
     Leave the unit 4 sections alone until then; they're here so you know
     what's coming.

     Everything is pasted as TEXT. No screenshots, no images, no video links.
     A typed block of output gets full credit; a picture of the same output
     gets none.
     ───────────────────────────────────────────────────────────────────────── -->

<!-- ═══════════════════════ UNIT 3 — THE BUILD ═══════════════════════ -->

## What This Does

<!-- Three or four sentences: what a user asks for, and what they get back. -->

FitFindr is a thrift-shopping agent. The user describes what they want in
plain words, optionally with a size and a budget, for example `python app.py
ask 'vintage graphic tee under $30 size M'`. It searches the secondhand
listings for the best match within that size and budget, then suggests one or
two outfits built around it, using pieces from the user's own wardrobe when
it has them. Finally it writes a short social-media "fit card" caption naming
the item, its price, and where to buy it. If nothing matches, it stops before
the outfit step and tells the user what to change in their search, but it
never suggests going over budget.


---

## Tool Inventory

<!-- Four lines per tool. This is worth 2 points and it's the single most
     common place students lose them.

     "Returns a list" earns NOTHING. The description has to say what is IN
     the list.

     The empty case isn't optional either — it's the thing your loop branches
     on, and if you don't decide it here you'll discover it as a crash in
     Milestone 5. -->

### `search_listings`

- **What it does:** Searches the thrift listings for items whose title, description, category, style tags, colors, or brand share keywords with the description. Title matches count double. It filters by size and maximum price when those are given.
- **Inputs:** `description` (str): keywords such as "vintage graphic tee"; `size` (str | None): matched case-insensitively against each part of a slash-separated size, so "M" matches "S/M" but "S" does not match "US 9", and "One Size" matches any size; None skips the size filter; `max_price` (float | None): an inclusive price ceiling; None skips the price filter.
- **Returns:** A list of up to `config.SEARCH_RESULT_LIMIT` listing dicts, highest keyword score first. Each dict has `id`, `title`, `description`, `category`, `style_tags` (list), `size`, `condition`, `price` (float), `colors` (list), `brand` (str or None), and `platform`.
- **When it has nothing:** Returns an empty list `[]`, not None and not an exception. This happens when no listing passes the filters or none shares a keyword with the description. The loop branches on this and stops with a "no matches" message.

### `suggest_outfit`

- **What it does:** Asks the model for one or two outfits built around a thrifted item. When the user has a wardrobe, the outfits name specific pieces they already own; the prompt tells the model not to invent pieces.
- **Inputs:** `new_item` (dict): a listing dict from `search_listings`, using its title, category, colors, style_tags, size, condition, description, and brand if there is one; `wardrobe` (dict): has an `items` key holding a list of wardrobe item dicts (`id`, `name`, `category`, `colors`, `style_tags`, `notes`), following `data/wardrobe_schema.json`. The list may be empty.
- **Returns:** A non-empty string with one or two outfit suggestions, a short paragraph each, saying which pieces to wear together and why the combination works.
- **When it has nothing:** If `wardrobe['items']` is empty (or the wardrobe is None), it still returns a non-empty string of general styling advice using common pieces, never "" and never an exception. If the model returns an empty response, it returns a short fallback suggestion instead.

### `create_fit_card`

- **What it does:** Asks the model to write a short social-media caption, in the buyer's voice, about the thrifted item styled in the suggested outfit. It mentions the item, its price and its platform once each, and describes the vibe.
- **Inputs:** `outfit` (str): the outfit suggestion returned by `suggest_outfit`; `new_item` (dict): the listing dict for the item, using its `title`, `price` (float), `platform`, `style_tags` (list), and `colors` (list).
- **Returns:** A 2–4 sentence caption string that reads like a real post rather than a product description. With temperature at 0.9 and the cache off, it comes out differently on each run.
- **When it has nothing:** If `outfit` is empty or only whitespace, it returns "Can't write a fit card without an outfit."


---

## Planning Loop

<!-- Your branch rule, stated as a rule — the condition AND both paths — plus
     the file and function that holds it.

     Like this:
       "If search_listings returns an empty list, put a message in the session
        and stop. Otherwise take the first result and go to suggest_outfit."
        — agent.py::run_agent

     The grader checks your code against what you claim here, so the file and
     function have to be real. -->

**Branch rule:** If `search_listings` returns an empty list, put a
"no matching listings" error in `session["error"]` and stop: return the
session without calling `suggest_outfit` or `create_fit_card`, so
`outfit_suggestion` and `fit_card` stay `None`. Otherwise, take the first
result, store it in `session["selected_item"]`, and go to `suggest_outfit`,
then `create_fit_card`.

The price ceiling is a budget, so it is a hard limit. The search always keeps
it, and the "no matching listings" error never suggests spending more.
`_nothing_found_message` re-runs the search without the size filter, still
within budget, to decide which of two messages to give:

- **Nothing matches the description within budget, even with no size
  filter:** the only suggestion is broader words, since changing the size
  can't help.
  ```
  Nothing in the listings matched description 'graphic tee' under $5 even before filtering by size.
  Things to change: try broader words — 'jacket' finds more than 'cropped corduroy jacket'.
  ```
- **Something matches within budget, but not in that size:** it suggests
  dropping or changing the size.
  ```
  Nothing in the listings matched description 'graphic tee' in size XXS under $30.
  Things to change: drop the size, or try a neighbouring one.
  ```

**Where it lives:** `agent.py::run_agent` (the message comes from
`agent.py::_nothing_found_message`)

**How the query is parsed:** Regex, in `agent.py::parse_query`, with no model
call. One pattern pulls out a price ("under $30" → `max_price=30.0`) and
another pulls out a size ("size M", "size US 9", "W29", or a trailing ", M").
The matched text is removed, and what's left becomes the `description`. Regex
costs nothing and gives the same answer every time. Its limit is phrasing it
doesn't know: "nothing over thirty dollars" sets no price ceiling.

**What moves through the session:** In order:
1. `query`: the raw text the user typed.
2. `parsed`: `{description, size, max_price}` from `parse_query`.
3. `search_results`: the list returned by `search_listings` (may be `[]`).
4. *The branch:* if `search_results` is empty, `error` is set and the run stops here.
5. `selected_item`: `search_results[0]`.
6. `outfit_suggestion`: `suggest_outfit(selected_item, wardrobe)`.
7. `fit_card`: `create_fit_card(outfit_suggestion, selected_item)`.

`error` stays `None` on a full run. It is set in two cases:
- **No matching listings:** `search_listings` returned `[]`, so the run stops
  after step 4 (shown under Branch rule).
- **Model unavailable:** `generate()` raised `ModelUnavailable`, for example
  because of a bad API key or no network. The search results are kept, and
  the error says how many listings were found and to check `GEMINI_API_KEY`
  in `.env`.

---

## Sample Run

<!-- Two things go here.

     1. One FULL query and its output, pasted as text.
     2. Your three per-tool terminal tests — the command and what it printed. -->

**One full query**

```
$ python app.py ask 'hot festival top for women under $25'

output:
[1] parse_query
      in:  hot festival top for women under $25
      out: dict with keys: description, size, max_price
[2] search_listings (via MCP)
      in:  dict with keys: description, size, max_price
      out: 3 items: Low-Top Canvas Sneakers — Off-White, Mesh Long-Sleeve Top — Black, Crochet Halter Top — Cream
      →    3 match(es)
[3] select_item
      out: Low-Top Canvas Sneakers — Off-White ($20.0, poshmark)
[4] suggest_outfit
      in:  Low-Top Canvas Sneakers — Off-White ($20.0, poshmark)
      out: Here are two ways to style the off-white low-top canvas sneakers using pieces from your existing wardrobe:  **…
      →    10 wardrobe item(s)
[5] create_fit_card
      in:  Low-Top Canvas Sneakers — Off-White ($20.0, poshmark)
      out: Just scored these cream canvas sneakers on Poshmark for only $20, and they’re already my go-to for effortless …

  Found:    Low-Top Canvas Sneakers — Off-White — $20.0 on poshmark

  Outfit:   Here are two ways to style the off-white low-top canvas sneakers using pieces from your existing wardrobe:

**Outfit 1:** Pair the off-white low-top canvas sneakers with your **white ribbed tank top** and **baggy straight-leg jeans, dark wash**. Add the **blackcrossbody bag** to complete the look. This combination works because the slim, minimal profile of the canvas sneakers balances the volume of the baggy jeans, while the white tank and off-white shoes create a cohesive, effortless palette that leans into an easy streetwear aesthetic.

**Outfit 2:** Style the off-white low-top canvas sneakers with the **wide-leg khaki trousers**, the **white ribbed tank top**, and layer the **vintage black denim jacket** on top. This works because the off-white colorway of the shoes naturally complements the earthy tan trousers, while the slightly cropped black jacket adds structure to the relaxed, wide-leg silhouette without overpowering it.

  Fit card: Just scored these cream canvas sneakers on Poshmark for only $20, and they’re already my go-to for effortless streetwear. I’ve been living inthem with baggy dark denim and a simple white tank, but they look just as good dressed up with wide-leg khakis and a vintage jacket. Such a steal for a minimalist staple I'll wear with literally everything!
```

**The three tools, tested one at a time**

```
$ python -c "from tools import search_listings; print(search_listings('graphic tee', max_price=30))"

output:
[{'id': 'lst_006', 'title': 'Graphic Tee — 2003 Tour Bootleg Style', 'description': 'Vintage-style bootleg tee with faded graphic. Slightly boxy fit. 100% cotton, soft and worn-in.', 'category': 'tops', 'style_tags': ['graphic tee', 'vintage', 'grunge', 'streetwear', 'band tee'], 'size': 'L', 'condition': 'good', 'price': 24.0, 'colors': ['black'], 'brand': None, 'platform': 'depop'}, {'id': 'lst_002', 'title': 'Y2K Baby Tee — Butterfly Print', 'description': 'Super cute early 2000s baby tee with butterfly graphic. Fitted crop length. Tag says medium but fits like a small.', 'category': 'tops', 'style_tags':['y2k', 'vintage', 'graphic tee', 'cottagecore'], 'size': 'S/M', 'condition': 'excellent', 'price': 18.0, 'colors': ['white', 'pink', 'purple'], 'brand':None, 'platform': 'depop'}, {'id': 'lst_033', 'title': 'Vintage Band Tee — Faded Grey', 'description': 'Faded grey band-style tee with distressed graphic. Crew neck. Fits boxy. Well-loved but no holes or major damage.', 'category': 'tops', 'style_tags': ['vintage', 'grunge', 'band tee', 'graphic tee', 'streetwear'], 'size': 'L', 'condition': 'fair', 'price': 19.0, 'colors': ['grey', 'charcoal'], 'brand': None, 'platform': 'depop'}, {'id': 'lst_015', 'title': 'Vintage Graphic Hoodie — Faded Black', 'description': 'Faded black pullover hoodie with barely-visible vintage graphic on the chest. Cozy interior. Some pilling but adds to the worn-in look.', 'category': 'tops', 'style_tags': ['vintage', 'grunge', 'graphic', 'streetwear'], 'size': 'L', 'condition': 'fair', 'price': 26.0, 'colors': ['black', 'charcoal'], 'brand': None, 'platform': 'depop'}, {'id': 'lst_017', 'title': 'Mesh Long-Sleeve Top — Black', 'description': 'Sheer black mesh long-sleeve. Great for layering under a graphic tee or over a bralette. Stretchy material, fits true to size.', 'category': 'tops', 'style_tags': ['y2k', 'grunge', 'goth', 'layering'], 'size': 'S/M', 'condition': 'excellent', 'price': 15.0, 'colors': ['black'], 'brand': None, 'platform': 'depop'}, {'id': 'lst_011', 'title': 'Low-Rise Cargo Pants — Khaki', 'description': 'Y2K era low-rise cargo pants. Lots of pockets. Khaki color, slightly distressed at the hems. Great for layering with a long tee.', 'category': 'bottoms', 'style_tags': ['y2k', 'cargo', '2000s', 'streetwear'], 'size': 'W29', 'condition': 'fair', 'price': 27.0, 'colors': ['khaki', 'tan'], 'brand': None, 'platform': 'poshmark'}]

```

```
$ python -c "from tools import search_listings, suggest_outfit; from utils.data_loader import get_example_wardrobe; item = search_listings('graphic tee', max_price=30)[0]; print(suggest_outfit(item, get_example_wardrobe()))"

output:
**Outfit 1: Ultimate Grunge Streetwear**
Pair the graphic tee with your baggy straight-leg jeans, dark wash and black combat boots. Layer the vintage black denim jacket on top, and finish the look with the black crossbody bag. This combination leans heavily into the tee's vintage bootleg and grunge vibe; tucking the slightly boxy tee into the high-waisted, dark indigo jeans creates a balanced silhouette, while the combat boots and matching black denim jacket anchor the streetwear aesthetic.

**Outfit 2: Casual Contrast**
Wear the graphic tee tucked into your wide-leg khaki trousers, and pair them with your chunky white sneakers. Add the black crossbody bag to complete thelook. This outfit works because the earth tones of the khaki trousers ground the dark, edgy graphic of the tee, while the chunky white sneakers add a fresh, casual contrast that ties into the relaxed, boxy fit of the shirt.
```

```
$ python -c "from tools import search_listings, suggest_outfit, create_fit_card; from utils.data_loader import get_example_wardrobe; item = search_listings('graphic tee', max_price=30)[0]; outfit = suggest_outfit(item, get_example_wardrobe()); print(create_fit_card(outfit, item))"
"
output:
Finally tracked down this 2003 tour bootleg tee on depop for $24 and I'm obsessed with how heavy the grunge vibes are. Just styled it with baggy dark denim and combat boots for the ultimate street fit, but it also looks so good dressed down with khaki trousers. Such a lucky find!
```

---

## How I Used AI

<!-- Two specific moments. What you asked, what came back, what you changed.

     "I used Claude to help me code" is not enough.

     "I gave Claude my search_listings spec. It returned None on no match
     instead of an empty list, so I changed it" is the level we want. -->

**Moment 1**

- *What I asked for:* I asked Claude to draft criterion 5 in `criteria.md`,
  the "your choice" one, about whether the search respects a price ceiling.
- *What came back:* A criterion that only checked price: across 5 queries
  with a price ceiling, every listing in `session["search_results"]` has
  `price <=` the ceiling.
- *What I changed:* I made it check both price and size. Every query now has
  to state a price ceiling and a size, and every result has to pass both
  filters. I had Claude spell out what "matches the size" means (split on
  `/`, drop parentheticals, `One Size` always passes) so I can check it by
  hand against the data, which mixes `S/M`, `US 8.5`, and `W30 L30`. I also
  added a rule that at least 1 of the 5 queries must return a non-empty list.
  Without it, a filter that's too strict could return `[]` every time and
  still "pass".

**Moment 2**

- *What I asked for:* I asked Claude to handle the empty-list case of the
  branch rule in `agent.py::run_agent`. When `search_listings` returns `[]`,
  the run should stop before `suggest_outfit` and put an error in
  `session["error"]` that tells the user what to change.
- *What came back:* The branch stopped the run correctly, but the error was a
  generic "no results" message, and its suggestions included raising the
  price ceiling.
- *What I changed:* I told Claude the price ceiling is a budget, so it's a
  hard limit, and the message should never suggest spending more. Claude then
  wrote `agent.py::_nothing_found_message`. It re-runs the search without the
  size filter but keeps the budget, then picks one of two messages. If
  nothing matches within budget even without the size filter, it suggests
  broader words. If something matches within budget but not in that size, it
  suggests dropping the size or trying a neighbouring one. Both messages name
  the description, size, and budget the user actually searched for (examples
  under **Branch rule** above).

<!-- ═══════════════════════ UNIT 4 — THE TEST ═══════════════════════

     Don't fill these in during unit 3.
     ═══════════════════════════════════════════════════════════════════ -->

---

## Run Log — Before

<!-- Five criteria, five tries each, in this exact format.

     Five, because your criteria are written out of five. Mark each try PASS
     or FAIL, count the passes, and read that count against your target — a
     row targeting 4 of 5 with three PASS cells is MISSED (3/5).

     `python run_eval.py --label before` runs everything and writes the table
     into results/. Paste it here and fill in the verdicts. -->

| Criterion | Target | Try 1 | Try 2 | Try 3 | Try 4 | Try 5 | Verdict |
|---|---|---|---|---|---|---|---|
| 1.  |  |  |  |  |  |  |  |
| 2.  |  |  |  |  |  |  |  |
| 3.  |  |  |  |  |  |  |  |
| 4.  |  |  |  |  |  |  |  |
| 5.  |  |  |  |  |  |  |  |

**Real output from one try**, pasted as text, naming the file and function
that produced it:

```

```

---

## Verdicts and Diagnoses

<!-- MET or MISSED per criterion against LAST UNIT's target, plus a sentence on
     how you decided.

     Then, for every miss: which of the four places it happened — a tool, the
     loop's branch, the session, or the model's output — AND the mechanism.

     Not a diagnosis:  "The fit card was bad."
     A diagnosis:      "The fit card criterion missed on 2 of 5 items. Both had
                        an empty brand field. My prompt puts the brand in the
                        first sentence, so the card opened with a blank and read
                        like a fragment. The tool worked; the prompt assumed a
                        field that isn't always there."

     Look for a pattern. Three misses on the same tool is one problem, not
     three. -->

| # | Criterion | Target | Verdict | How I decided |
|---|---|---|---|---|
| 1 |  |  |  |  |
| 2 |  |  |  |  |
| 3 |  |  |  |  |
| 4 |  |  |  |  |
| 5 |  |  |  |  |

**Diagnoses**



---

## Loop Trace

<!-- One full run, printed step by step, with the MCP call visible in it.

     `python app.py ask '...' --trace` once you've added the trace.step()
     calls in Milestone 2.

     Worth pasting BOTH the happy path and the empty-search path. The empty
     one should be visibly shorter, because it stops. If your two traces are
     the same length, your branch isn't working — and this is the fastest way
     anyone will ever find that out. -->

**Happy path**

```

```

**Empty search**

```

```

**On the MCP move:** <!-- what changed in your code, and whether anything
behaved differently afterwards. If the rewire didn't work, say exactly where it
broke — the error text and the last thing that worked. That earns the point in
full. -->



---

## The Improvement

<!-- What you changed, why your diagnosis pointed at it, and the after-run in
     the same table format. One change, measured properly.

     `python run_eval.py --label after` -->

**What I changed:**

**Which failure it was meant to fix:**

### Run Log — After

| Criterion | Target | Try 1 | Try 2 | Try 3 | Try 4 | Try 5 | Verdict |
|---|---|---|---|---|---|---|---|
| 1.  |  |  |  |  |  |  |  |
| 2.  |  |  |  |  |  |  |  |
| 3.  |  |  |  |  |  |  |  |
| 4.  |  |  |  |  |  |  |  |
| 5.  |  |  |  |  |  |  |  |

**Did it help, and how do I know:**

<!-- If it made things worse, say that. Honestly reported, that earns full
     credit and is more interesting than one that worked. -->



---

## What's Still Broken

<!-- For each criterion still missed: what you'd do, and why you stopped where
     you did. "I ran out of time" is fine if it's true. Pretending nothing is
     left is not. -->



<!-- ═════════════════════════════════════════════════════════════════════

     SUBMISSION CHECKLIST — unit 3

       [ ] criteria.md has five numbered criteria, each with a target
       [ ] Each criterion has a reason underneath it
       [ ] All five unit 3 sections above have real content
       [ ] Tool Inventory: all three tools, inputs WITH TYPES, a specific
           return value, and the empty case
       [ ] Planning Loop names the branch rule and agent.py::run_agent
       [ ] Sample Run: one full query plus the three per-tool tests, as text
       [ ] At least four new commits
       [ ] Repository URL submitted — WRITE IT DOWN, you submit the same one
           next unit

     SUBMISSION CHECKLIST — unit 4

       [ ] mcp_server.py exists with one tool registered
           (or a written record of exactly where the rewire broke)
       [ ] Run Log — Before, five criteria, five tries each
       [ ] Real output pasted underneath, naming file and function
       [ ] A verdict on every criterion
       [ ] A diagnosis for every miss, naming a place AND a mechanism
       [ ] Loop Trace, with the MCP call visible in it
       [ ] All three failure modes triggered and handled
       [ ] One improvement, with Run Log — After in the same format
       [ ] What's Still Broken
       [ ] At least four new commits
       [ ] The SAME repository URL as last unit

     Do not delete and recreate this repository. Your commit history is what
     shows your criteria existed before your results did.
     ═════════════════════════════════════════════════════════════════════ -->

---

📖 **How to run this project: [RUNNING.md](RUNNING.md)**
