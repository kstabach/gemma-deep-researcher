---
name: deep-researcher
description: Conducts in-depth research by querying Tavily and Brave Search in parallel for a given topic, returning normalized citations.
metadata:
  require-secret: true
  require-secret-description: 'Paste a JSON blob with both keys, e.g. {"tavily":"tvly-...","brave":"BSA..."}'
---

# Deep Researcher

## Instructions
When the user wants "deep research" or a "detailed report":
1. Call the `run_js` tool.
2. Pass a JSON string with a `topic` field containing the user's research subject. Optional: `max_results` (int, default 8).
3. The tool returns JSON of the form:
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
   `degraded` only appears when one engine failed but the other succeeded. If BOTH engines fail, the response is `{ "error": "both engines failed", "details": [...] }` — in that case, tell the user research is temporarily unavailable and stop.
4. Write a structured, multi-paragraph report using the snippets. Every factual claim must end with an inline citation in the form `[n]` matching `sources[].n`.
5. End the report with a `## Sources` section listing every citation as `[n] Title — URL`.
6. If `degraded` is present, add a short italic note at the end: `_Note: results drawn from <engine> only; <other> was unavailable._`

## Output format
- `# <Topic>` heading
- 3–5 paragraphs grouped by sub-theme
- Inline `[n]` citations
- `## Sources` footer

---

<!DOCTYPE html>
<html>
<body>
  <script>
    // Deep Researcher — calls Tavily + Brave in parallel, returns normalized citations.
    // Secret is a JSON blob: {"tavily":"tvly-...","brave":"BSA..."}

    const FETCH_TIMEOUT_MS = 15000;
    const MAX_ATTEMPTS = 2; // initial try + 1 retry

    // Typed error so the caller can categorize failures cleanly.
    class FetchError extends Error {
      constructor(kind, message, status) {
        super(message);
        this.kind = kind;       // 'timeout' | 'network' | 'cors' | 'http' | 'parse'
        this.status = status;   // HTTP status if kind === 'http'
      }
    }

    function sleep(ms) {
      return new Promise(resolve => setTimeout(resolve, ms));
    }

    // fetch + AbortController timeout + one retry on transient failures.
    // Returns the parsed JSON body. Throws FetchError with a typed kind.
    async function fetchJsonWithRetry(label, url, init) {
      let lastErr;
      for (let attempt = 1; attempt <= MAX_ATTEMPTS; attempt++) {
        const controller = new AbortController();
        const timer = setTimeout(() => controller.abort(), FETCH_TIMEOUT_MS);
        try {
          const res = await fetch(url, {
            ...init,
            mode: 'cors',
            credentials: 'omit',
            cache: 'no-store',
            signal: controller.signal
          });
          clearTimeout(timer);

          if (!res.ok) {
            // 4xx are terminal (auth/quota/bad request) — don't retry.
            // 5xx / 429 are transient — retry once.
            const transient = res.status >= 500 || res.status === 429;
            const err = new FetchError('http', label + ' http ' + res.status, res.status);
            if (!transient || attempt === MAX_ATTEMPTS) throw err;
            lastErr = err;
          } else {
            try {
              return await res.json();
            } catch (e) {
              throw new FetchError('parse', label + ' parse: ' + e.message);
            }
          }
        } catch (e) {
          clearTimeout(timer);
          if (e instanceof FetchError) {
            if (e.kind === 'parse') throw e;
            lastErr = e;
          } else if (e.name === 'AbortError') {
            lastErr = new FetchError('timeout', label + ' timeout after ' + FETCH_TIMEOUT_MS + 'ms');
          } else if (e instanceof TypeError) {
            // Browsers surface CORS and DNS/offline failures as a generic TypeError.
            // We tag it 'cors' because inside the AI Edge Gallery webview that's the
            // overwhelmingly likely cause — the device has network or the whole skill
            // wouldn't have loaded.
            lastErr = new FetchError('cors', label + ' blocked (cors or network): ' + e.message);
          } else {
            lastErr = new FetchError('network', label + ' network: ' + e.message);
          }
          if (attempt === MAX_ATTEMPTS) throw lastErr;
        }
        // Backoff before retry: 400ms, then 800ms (only ever reached once with MAX_ATTEMPTS=2)
        await sleep(400 * attempt);
      }
      throw lastErr;
    }

    async function searchTavily(apiKey, query, maxResults) {
      const data = await fetchJsonWithRetry('tavily', 'https://api.tavily.com/search', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          api_key: apiKey,
          query: query,
          search_depth: 'advanced',
          max_results: maxResults
        })
      });
      return (data.results || []).map(r => ({
        title: r.title || '',
        url: r.url || '',
        snippet: (r.content || '').slice(0, 500),
        source: 'tavily'
      }));
    }

    async function searchBrave(apiKey, query, maxResults) {
      const url = 'https://api.search.brave.com/res/v1/web/search?q='
        + encodeURIComponent(query) + '&count=' + maxResults;
      const data = await fetchJsonWithRetry('brave', url, {
        method: 'GET',
        headers: {
          'Accept': 'application/json',
          'Accept-Encoding': 'gzip',
          'X-Subscription-Token': apiKey
        }
      });
      const results = (data.web && data.web.results) || [];
      return results.map(r => ({
        title: r.title || '',
        url: r.url || '',
        snippet: (r.description || '').replace(/<\/?[^>]+(>|$)/g, '').slice(0, 500),
        source: 'brave'
      }));
    }

    function dedupeAndNumber(items) {
      const seen = new Set();
      const out = [];
      let n = 1;
      for (const it of items) {
        if (!it.url || seen.has(it.url)) continue;
        seen.add(it.url);
        out.push({ n: n++, title: it.title, url: it.url, snippet: it.snippet, source: it.source });
      }
      return out;
    }

    function formatError(engine, err) {
      if (err && err.kind) {
        return { engine: engine, kind: err.kind, message: err.message, status: err.status };
      }
      return { engine: engine, kind: 'unknown', message: (err && err.message) || String(err) };
    }

    window['ai_edge_gallery_get_result'] = async (data, secret) => {
      try {
        const { topic, max_results } = JSON.parse(data);
        if (!topic) throw new Error('missing "topic" field');

        let keys;
        try {
          keys = JSON.parse(secret);
        } catch (_) {
          throw new Error('secret must be JSON: {"tavily":"...","brave":"..."}');
        }
        if (!keys.tavily || !keys.brave) {
          throw new Error('secret JSON must contain both "tavily" and "brave" keys');
        }

        const perEngine = Math.max(1, Math.min(20, max_results || 8));

        const [tavilyRes, braveRes] = await Promise.allSettled([
          searchTavily(keys.tavily, topic, perEngine),
          searchBrave(keys.brave, topic, perEngine)
        ]);

        // Interleave so both engines are represented even if one is slower / shorter.
        const tav = tavilyRes.status === 'fulfilled' ? tavilyRes.value : [];
        const bra = braveRes.status === 'fulfilled' ? braveRes.value : [];
        const interleaved = [];
        for (let i = 0; i < Math.max(tav.length, bra.length); i++) {
          if (tav[i]) interleaved.push(tav[i]);
          if (bra[i]) interleaved.push(bra[i]);
        }

        const sources = dedupeAndNumber(interleaved);

        const errors = [];
        if (tavilyRes.status === 'rejected') errors.push(formatError('tavily', tavilyRes.reason));
        if (braveRes.status === 'rejected') errors.push(formatError('brave', braveRes.reason));

        // Hard-fail only if BOTH engines died and we have zero sources.
        if (sources.length === 0 && errors.length === 2) {
          return JSON.stringify({
            error: 'both engines failed',
            details: errors
          });
        }

        return JSON.stringify({
          result: JSON.stringify({
            topic: topic,
            sources: sources,
            degraded: errors.length > 0 ? errors : undefined
          })
        });
      } catch (e) {
        return JSON.stringify({ error: e.message });
      }
    };
  </script>
</body>
</html>
