---
name: deep-researcher
description: Conducts in-depth research by querying Tavily and Brave Search in parallel for a given topic, returning normalized citations.
metadata:
  require-secret: true
  require-secret-description: 'Paste a JSON blob with both API keys, e.g. {"tavily":"tvly-...","brave":"BSA..."}'
---

# Deep Researcher

## Instructions
When the user wants "deep research" or a "detailed report":
1. Call the `run_js` tool using `index.html` and a JSON string for `data` with the following fields:
   - **topic**: Required. The research subject as a clear noun phrase.
   - **max_results**: Optional integer, default 8.
2. The tool returns JSON of the form:
   ```json
   {
     "topic": "...",
     "sources": [
       { "n": 1, "title": "...", "url": "...", "snippet": "...", "source": "tavily" | "brave" }
     ],
     "degraded": [
       { "engine": "brave", "kind": "timeout|cors|http|network|parse", "message": "...", "status": 429 }
     ]
   }
   ```
   `degraded` only appears when one engine failed but the other succeeded. If BOTH engines fail, the response is `{ "error": "both engines failed", "details": [...] }` — tell the user research is temporarily unavailable, then list each engine's `kind` and `message` from `details` so the error can be diagnosed (e.g. "tavily: cors — tavily blocked (cors or network): Failed to fetch").
3. Write a structured, multi-paragraph report using the snippets. Every factual claim must end with an inline citation in the form `[n]` matching `sources[].n`.
4. End the report with a `## Sources` section listing every citation as `[n] Title — URL`.
5. If `degraded` is present, add a short italic note at the end: `_Note: results drawn from <engine> only; <other> was unavailable._`

## Output format
- `# <Topic>` heading
- 3–5 paragraphs grouped by sub-theme
- Inline `[n]` citations
- `## Sources` footer
