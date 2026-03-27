---
name: slidehub
description: "Use this skill whenever the user wants to create, find, assemble, or manage presentation slides — and the SlideHub MCP server is connected. This includes: generating new slides from text, searching for existing slides in a company library, building multi-slide decks, filling placeholders in templates, downloading presentations, or asking questions about available slide content. Trigger whenever the user mentions 'slides', 'deck', 'presentation', 'pitch deck', 'slide library', 'SlideHub', or asks to create visual content that could be a slide. ALSO trigger when the user asks to 'make a slide about X' or 'create a presentation for Y' — even if they don't mention SlideHub — because existing slides in their library may already cover the topic and should be checked first. This skill encodes the intelligent routing logic that makes SlideHub feel like a presentation copilot, not just a set of API tools."
---

# SlideHub Skill

This skill teaches Claude how to use SlideHub's MCP tools intelligently — not just what each tool does, but *when* and *why* to use them, and in what order.

## The Golden Rule: Search Before You Generate

**Every time a user asks to create, build, or make slides, FIRST search their SlideHub library.** Their company has invested time curating on-brand slides. Generating from scratch should be a fallback, not the default.

```
User asks for slides about X
    │
    ├─ search_assets(searchTerm: "X") or ai_chat_asset_search(searchTerm: "X")
    │
    ├─ Results found?
    │   ├─ YES → Present options with get_slide_preview
    │   │        → Ask: "I found these existing slides. Want to use one, or generate fresh?"
    │   │        → If slides have placeholders → get_placeholders → fill_placeholders
    │   │
    │   └─ NO → Proceed to generation pipeline
    │
    └─ User explicitly says "generate new" or "create from scratch"
        → Skip search, go straight to generation
```

---

## First Interaction: Know Your User

On the first SlideHub interaction in a conversation, establish context silently (no need to narrate this to the user):

1. **`validate_access`** — Who is this user? What company?
2. **`get_slide_libraries`** — What slide, icon, and image libraries are available?
3. **`get_team_information`** — What teams exist? (useful for filtering later)

Cache these results mentally for the rest of the conversation. This grounds everything in the right brand and team context.

---

## Decision Router

| User intent | Primary action | Tool sequence |
|---|---|---|
| "Make a slide about X" | Search first, then generate | `search_assets` → (if empty) generation pipeline |
| "Find slides about X" | Search | `search_assets` or `ai_chat_asset_search` |
| "Do we have anything on X?" | Semantic search | `ai_chat_asset_search` |
| "Generate a slide from this text" | Generate (explicit) | Generation pipeline |
| "Build me a deck about X" | Multi-slide assembly | See [Deck Assembly](#deck-assembly) |
| "Update the numbers in slide X" | Fill placeholders | `get_placeholders` → `fill_placeholders` |
| "Download that presentation" | Download | `download_presentation` |
| "Show me our key decks" | Key presentations | `get_key_presentations` |
| "What categories/tags do we use?" | Taxonomy | `get_base_label_information` |
| "Combine these slides into one deck" | Combine | `combine_slides` |

---

## Search: Choosing the Right Tool

SlideHub has two search tools. Use the right one:

**`search_assets`** — Keyword/metadata search. Best for:
- Specific topics: "ESG", "quarterly review", "team intro"
- Filtering by category or tag (pass `categoryIds` or `tagIds`)
- Browsing: "show me all slides tagged 'sales'"

**`ai_chat_asset_search`** — Semantic/AI search. Best for:
- Vague or conceptual queries: "do we have anything about our company values?"
- Questions about content: "which slides mention our pricing model?"
- When keyword search returned nothing useful — try semantic as a second pass

**Search strategy:**
1. Start with `search_assets` for direct topic matches
2. If results are sparse or irrelevant, follow up with `ai_chat_asset_search`
3. Always show results to the user via `get_slide_preview` or `get_asset_details` — let them see thumbnails, not just titles

---

## Generation: Pipeline vs. Quick Generate

Two paths exist. Route based on the situation:

### Quick Generate (`quick_generate`)
Single tool call. Use when:
- The content is simple and straightforward (one clear topic)
- Speed matters more than perfect template matching
- The user says "just make a quick slide"
- Content is under ~200 words

### Full Pipeline
Six steps, more control. Use when:
- Content is complex, long, or multi-faceted
- The user cares about quality (client-facing, investor deck, board meeting)
- You want to control template selection
- Content is >200 words or includes data/charts

**Full pipeline sequence:**

```
validate_text_input(content)
    → Is valid?
        → evaluate_content(content)
            → Returns: updatedContent, explicitLayoutPreference, batchId
                → find_best_inspiration(content: updatedContent, explicitLayoutPreference, batchId)
                    → Returns: inspirationId, batchId
                        → generate_slide(content: updatedContent, inspirationId, batchId)
                            → Returns: jobId
                                → check_job_status(jobId)  ← poll every 5s
                                    → When completed → get_slide_result(jobId)
```

**Critical details:**
- Pass `updatedContent` (not original content) from `evaluate_content` onward
- Pass `explicitLayoutPreference` from `evaluate_content` to `find_best_inspiration`
- The `batchId` from `evaluate_content` is authoritative — thread it through all steps
- Poll `check_job_status` every 5 seconds. If 10 polls pass without completion, tell the user it's taking longer than expected and stop
- Always finish with `get_slide_result(jobId)` to render the visual preview

### Using the Right Library

Before generating, consider which library to use:
- **`"off-the-shelf"`** — SlideHub's default template library. Good starting point.
- **`"company-{companyId}"`** — The user's company templates. Preferred for on-brand output.
- **`"team-{teamId}"`** — Team-specific templates. Use when context is team-specific.

Call `get_slide_libraries` to discover available options. If the user's company has a library, prefer it over off-the-shelf.

---

## Placeholder Workflow

Many slides in a company library are templates with fillable placeholders (e.g., `[Company Name]`, `[Revenue]`, `[Date]`). This is the most efficient path for recurring content.

**When to use placeholders:**
- User wants to "update" or "fill in" an existing slide
- A search result has `hasPlaceholders: true`
- User asks for something that matches a known template pattern (deal sheet, monthly report, team intro)

**Sequence:**
```
get_placeholders(inspirationIds: [slideId])
    → Returns: [{mapId, identifier, type, defaultValue}]
        → Ask user for values (or infer from context)
            → fill_placeholders(inspirationIds: [slideId], values: {"mapId": "value"})
                → Returns: {downloadUrl, tempPresentationId}
```

**Key detail:** The `values` object uses `mapId` (as string) as keys, NOT the identifier/label. So if `get_placeholders` returns `{mapId: 58, identifier: "[Company Name]"}`, you pass `{"58": "Acme Corp"}`.

---

## Deck Assembly

When a user asks for a full presentation (not just one slide), think in terms of assembly:

### Strategy 1: Search-and-Combine (preferred)
Best when the company library is likely to have relevant slides.

1. Break the deck request into logical sections/topics
2. Search for each section: `search_assets` per topic
3. Present found slides to user, identify gaps
4. Generate only the missing slides
5. Combine everything: `combine_slides(inspirationIds: [...])` for library slides, or `combine_slides(tempPresentationIds: [...])` for generated slides

### Strategy 2: Generate-and-Combine
Best when creating something entirely new.

1. Break the deck into individual slide contents
2. Generate each slide (pipeline or quick_generate)
3. Wait for all jobs to complete
4. Combine: `combine_slides(tempPresentationIds: [jobId1, jobId2, ...])`

### Strategy 3: Key Presentations as Starting Points
When the user is vague ("help me make a sales deck"):

1. Call `get_key_presentations` to see curated company decks
2. Present them: "Your company has these key presentations — any of these a good starting point?"
3. Build from there

**Important `combine_slides` constraints:**
- Use `inspirationIds` for library slides OR `tempPresentationIds` for generated slides — not both in one call
- `tempPresentationIds` only works with single-slide temp presentations from `generate_slide`
- For slides with placeholders, use `fill_placeholders` with multiple `inspirationIds` instead of combining after filling

---

## Previewing and Presenting Results

Always make results visual:

- **After search:** Use `get_slide_preview(assetId)` to show thumbnails of found slides
- **After generation:** Use `get_slide_result(jobId)` to show the rendered result with download button
- **For details:** Use `get_asset_details(assetIds)` to get descriptions, categories, tags
- **Batch previews:** When showing multiple results, call `get_slide_result` with `jobIds` array (not separate calls)

---

## Taxonomy and Organization

Users may ask about how their content is organized:

- **`get_base_label_information`** — Returns categories, subcategories, and tags for the company
- **`get_team_information`** — Returns team structure
- Use these to power smarter searches (filter by `categoryIds` or `tagIds` in `search_assets`)
- When a user asks "show me all our sales slides," translate that to the right category/tag filter

---

## Error Handling and Edge Cases

**Generation takes too long:** After 10 polls of `check_job_status`, stop polling and inform the user. Do NOT try to diagnose or work around it.

**Search returns nothing:** Try `ai_chat_asset_search` as a fallback. If still nothing, proceed to generation. Frame it positively: "I didn't find existing slides on this topic, so let me generate a fresh one."

**User not authenticated:** If `validate_access` fails, tell the user they need to connect their SlideHub account. Do not attempt other tool calls.

**Content too short/long:** `validate_text_input` will catch this. Relay the validation message and help the user adjust their content.

---

## What NOT to Do

- **Don't generate before searching.** The library-first approach is the core value proposition.
- **Don't narrate tool mechanics.** Say "Let me check your slide library" not "I'll call search_assets with the searchTerm parameter."
- **Don't use `download_presentation` after `check_job_status`.** The status response already includes the download URL on completion.
- **Don't mix `inspirationIds` and `tempPresentationIds` in `combine_slides`.**
- **Don't poll indefinitely.** 10 polls max on `check_job_status`.
- **Don't skip `evaluate_content` in the full pipeline.** It optimizes the content and provides the `explicitLayoutPreference` that improves template matching.
