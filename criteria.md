# Acceptance criteria — FitFindr

Five criteria that say what "working" means for this agent, written in unit 3
**before** any results existed.

An acceptance criterion names a target: a number, a count, a rate, or something
a person could plainly observe. *"The agent handles errors"* is an opinion.
*"When search returns nothing, the agent stops before calling the second tool,
in 5 of 5 tries"* is a criterion.

Under each one, write a sentence or two on **why that target** and not a
stricter one. A reason that says something about your tools, your loop, or the
data earns credit; *"80% seemed reasonable"* does not.

> Missing your own targets next unit costs you nothing. Setting a target so
> easy you can't miss it does.

**Two are written for you. You write three.**

---

## 1. A matching query completes all three tools

Given a query that matches at least one listing, the agent completes all three
tool calls and returns a fit card — in at least 4 of 5 tries.

**Why this target:**
<!-- Why 4 of 5 and not 5 of 5? Something about your search, probably —
     "my search is a plain keyword match and some phrasings will miss" is a
     real answer. -->
`search_listings` is a plain keyword-overlap score over 40 listings, and
`parse_query` is a regex. A phrasing the regex doesn't know ("nothing over
thirty dollars") or words that aren't in any title/tag ("retro" when the data
says "vintage") can miss a listing that really is there. Requiring 5 of 5 would
be grading the thesaurus, not the loop; 4 of 5 still fails if more than one
ordinary phrasing breaks.

---

## 2. An impossible query stops before the second tool

Given a query that matches no listings, the agent stops before calling
`suggest_outfit` and returns a message naming what to change — 5 of 5 tries.

**Why this target:**
<!-- Why is 5 of 5 reasonable here when criterion 1 isn't? What's different
     about this path? -->
This path never touches the model. `parse_query`, `search_listings` and the
`if not results:` branch are all deterministic, so the same query gives the same
empty list every time — there is no randomness to forgive. One miss would mean
the branch itself is wrong, not that the model had a bad day.

---

## 3. Something about state

<!-- YOU WRITE THIS ONE.

     How would you know that the item your search found is the same item the
     next tool received? Name something countable or observable.

     This is the criterion people find hardest, because state failure doesn't
     look like state failure — it looks like a tool problem. Something that
     compares session["selected_item"] against what actually reached
     suggest_outfit is the shape you're after. -->

For a matching query, the listing `id` in `session["selected_item"]` equals
`session["search_results"][0]["id"]`, **and** equals the `id` of the item
recorded as the input to both `suggest_outfit` and `create_fit_card` in the
trace — 5 of 5 tries, checked by comparing the ids, not by reading the captions.

**Why this target:**
The hand-off from search to the next two tools is plain Python reading one key
out of the session dict — no model, no parsing — so there's no legitimate reason
for it ever to differ. Comparing `id`s (not titles) matters because several
listings share similar titles, and a caption about "a vintage tee" would look
fine even if it was the wrong tee.


---

## 4. Something about the fit card

<!-- YOU WRITE THIS ONE.

     The fit card calls a model, so the same input can produce different words
     each time. That's not a bug — it's the nature of the tool. So what would
     make it acceptable?

     Think about what you'd actually be unhappy to see. A caption that never
     mentions the price? Two different items producing the same opening
     sentence? A card longer than a caption anyone would post? Any of those can
     be turned into a number. -->

Across 5 runs on 5 different matching items, at least 4 of the 5 fit cards:
(a) contain the item's exact price (e.g. `$24`) and its platform name,
(b) are 2–4 sentences and under 70 words, and
(c) no two of the 5 cards start with the same first sentence.

**Why this target:**
The card comes from the model at `TEMPERATURE = 0.9`, so the wording will drift
and it will sometimes round a price or drop the platform even when the prompt
asks for both — 4 of 5 allows one slip without letting that become the norm.
Price and platform are the two facts a reader would actually act on, and both
are string-checkable against the listing dict. The 70-word cap is "something
you'd post"; the distinct-opening rule catches a template-y prompt (or a stale
cache) that produces the same card for different items.


---

## 5. Your choice

<!-- YOU WRITE THIS ONE TOO.

     Pick something you actually care about getting right. Speed, the empty
     wardrobe path, what happens when the model can't be reached, whether the
     search respects a price ceiling — anything, as long as it names a number
     or an observable outcome. -->

### Search respects a price ceiling and a size match

Across 5 queries that each state both a price ceiling and a size (e.g.
"vintage tee under $30 size M", "jeans under $40 in W30", "boots size US 8
under $60"), **5 of 5:** every listing in `session["search_results"]`:
(a) has `price <= ` the stated ceiling (inclusive, so a `$30.00` item passes
    "under $30"), and
(b) matches the stated size — the listing's size string, split on `/` with
    parentheticals dropped, contains the wanted size (so "M" matches `M`,
    `S/M` and `M/L`), or is a `One Size` listing.

And at least 1 of the 5 queries must return a non-empty list, so the check
can't pass by returning nothing.

**Why this target:**
The filter is deterministic Python: `max_price` is a plain `>` comparison and
`_size_matches` is set overlap on tokens. Same query, same results, every time
— so one over-budget or wrong-size listing means the filter is broken, not
unlucky, and 5 of 5 is the only fair bar. The size rule is spelled out because
the data mixes formats (`S/M`, `US 8.5`, `W30 L30`, `XL (oversized)`), and
"matches" has to mean something I can check by hand against the listing dict.
`W30 L30` is the case I expect to be a real test: it isn't split on `/`, so a
"W30" query may not match it. The non-empty rule rules out the cheap pass,
where an over-strict filter returns `[]` and trivially "respects" everything.

---

<!-- ─────────────────────────────────────────────────────────────────────────
     UNIT 4 — read this before you change anything above.

     If a criterion turns out to be BROKEN rather than merely unmet, you can
     revise it, and that earns credit. But never delete or edit the original
     line. Add the revision underneath it, like this:

         ## 4. Something about the fit card

         The fit card is different every time.

         **Why this target:** ...

         > **Revised in unit 4:** For 5 different items, the 5 fit cards share
         > no opening sentence.
         >
         > **Why revised:** "different" wasn't checkable — two cards that
         > differed by one word still counted. The new version is something I
         > can actually score.

     That's a revision because the criterion couldn't be MEASURED.

     Lowering a target because you missed it is not a revision, and it costs
     you the point:

         ✗ "I said the empty search stops it 5 of 5 times, but I got 3 of 5,
            so 3 of 5 is more realistic."

     A number you missed stays where it is, gets diagnosed, and gets a fix
     attempted. That's where the points are.
     ───────────────────────────────────────────────────────────────────────── -->
