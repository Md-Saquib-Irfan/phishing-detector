# Phishing Website Detector

Two tools, same detection logic, two contexts:

| File | Use for |
|---|---|
| `phishing-scanner.html` | Open directly in a browser — no install. Good for a live classroom demo. |
| `phishing_detector.py` | CLI / lab exercise. Can optionally fetch and inspect live page HTML. |

## How it scores (heuristic, explainable — not a black-box model)

Each check adds points to a 0–100 risk score, so every point in the final
number traces back to a named reason. That's deliberate for a training
context: students should see *why* something was flagged, not just a
verdict.

**URL structure & lexical checks** (both tools):
- Raw IP address as host
- Missing HTTPS
- `@` in the URL (classic redirect trick)
- Excessive length / subdomain nesting
- Known URL shorteners
- Abused TLDs (`.tk`, `.xyz`, `.top`, etc.)
- Credential-harvesting keywords (`login`, `verify`, `suspend`, ...)
- **Typosquatting** — Levenshtein distance against a brand list, checked
  against both the full domain and hyphen-separated chunks
  (catches things like `paypa1-secure-login.tk`)
- High-entropy subdomains (auto-generated phishing infra)
- Punycode / IDN homograph encoding (`xn--...`)

**Live content checks** (`phishing_detector.py --fetch` only — needs
`requests` + `beautifulsoup4`, and the HTML tool can't do this due to
browser CORS restrictions):
- Redirect chains to a different host
- Forms submitting credentials to a third-party domain
- Hidden/zero-size iframes
- Favicon served from a domain impersonating a known brand
- Obfuscated JavaScript (`eval`, `unescape`, `fromCharCode`)
- Login forms served over plain HTTP

## CLI usage

```bash
# URL-only analysis (no network request made)
python phishing_detector.py "http://paypa1-secure.tk/verify"

# Also fetch and inspect the live page
pip install requests beautifulsoup4
python phishing_detector.py "https://example.com" --fetch
```

Exit code is `1` if score ≥ 40 (suspicious or worse), `0` otherwise — handy
for scripting into a CI check or a bulk-URL triage script.

## Score bands

| Score | Verdict |
|---|---|
| 70–100 | HIGH RISK — likely phishing |
| 40–69 | SUSPICIOUS — proceed with caution |
| 15–39 | LOW RISK — minor red flags |
| 0–14 | LIKELY SAFE |

## Known limitations (worth calling out in a lab setting)

- Brand list is a short demo set (~22 names) — a real deployment needs a
  Tranco/Alexa top-N list.
- No real-time reputation lookup (Safe Browsing / VirusTotal API) — this
  is purely structural/lexical + on-page heuristics.
- Typosquat detection is edit-distance based, so it won't catch every
  visual homograph (e.g. Cyrillic look-alike characters that Punycode
  doesn't obviously flag) without a dedicated confusable-character table.
- Heuristic scores can both false-positive (a legit site with `login` in
  the URL and a hyphenated domain) and false-negative (a well-crafted
  phishing page with a clean-looking domain). Treat the score as a
  triage signal, not a verdict.

## Extending it

Good next additions for a course module: WHOIS domain-age lookup (freshly
registered domains are a strong phishing signal), a real confusable/
homograph character table, and hooking up Google Safe Browsing's API for
a reputation cross-check.
