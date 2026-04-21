# Email Generation Workflow (email_gen)

Third and final stage of the n8n pipeline. Takes the few leads that have survived the scraper and qualifier, finds a recent operational signal from the clinic's Facebook page, falls back to Google review patterns if Facebook is dead, and generates a short cold-email opener that references that signal. If no signal is strong enough, the workflow writes an error code instead of sending a weak email.

Sanitized public extract of a private project. Commit history lives in the private source repo.

---

## Scope & status

Manual batch trigger. Runs on rows in the CRM sheet where the `Opener` column is empty. The prompt constraints, the signal hierarchy, and the pass/error validation gate are my design; Apify and the LLM nodes do the heavy lifting.

AI-assisted during development (Claude Code for iteration). The bridge logic — the specific "if X signal, then Y admin pain, then Z opener" mappings — is the IP I'm keeping out of the public version.

---

## Why this stage exists

A cold email that opens with "Great website!" has zero credibility. The generator's job is to write one concrete sentence that proves the sender actually looked at the lead before writing. That means finding a real, recent, specific signal — not just the site content the qualifier already chewed through.

Two things follow from that: find the freshest signal available, and refuse to write an email when no signal is strong enough.

---

## Architecture

![email_gen workflow](./email_gen.png)

### Input and link resolution

- Reads CRM rows where `Opener` is empty
- A regex step (`FB _LINK_EXTRACT`) pulls any Facebook URLs out of the lead's site HTML
- If multiple Facebook links are found (corporate HQ page vs. local clinic page, etc.), `gpt-4.1-nano` picks the right one by comparing URL structure to the clinic's name
- A competitor check scans URLs and page data for a known competitor's vendor string. If matched, the lead is flagged and skipped — they're probably already using that vendor

### Signal hierarchy

The workflow branches on data availability:

- **Path A — Facebook signal (priority):** if a valid local page exists, Apify scrapes the last ~4 months of posts (text, dates, like counts). Recent operational changes — new hires, workshops, extended hours, holiday closures, new services — create acute admin pain, which is exactly what the offered product addresses.
- **Path B — Review signal (fallback):** if there's no usable Facebook data, fall back to the Google reviews and cleaned site text the qualifier already gathered. This path relies on chronic signals: high review volume implies front-desk load; concentrated staff praise implies scheduling bottlenecks around specific practitioners.

Facebook wins priority because a recent concrete event beats a structural inference every time. But a lot of clinics have dead socials, so the fallback has to exist or the pipeline drops half its leads.

### Generation

- Google Gemini generates either a Facebook-signal opener or a review-signal opener
- Both prompts are constrained to a single template: `"I saw X, which usually causes Y"` where Y is a specific admin pain the offered product solves
- Output is capped at 35 words, no greetings, no "great website" padding

### Validation — pass / error gate

The critical part. Each Gemini prompt can return `ERROR` or `PASS` instead of an opener if it can't find a concrete enough hook.

| Result | Action |
| --- | --- |
| Clean opener, under 35 words, correct format | Write to `CRM`, copy row to `EXPORT_READY` for the mailing tool |
| `ERROR` / `PASS` | Write the error code to `CRM`, move the row to `VERIF_NEEDED` for manual review |

The error fallback was added after the first batch of openers drifted toward generic "love what you're doing!" filler when no real hook existed. Letting the LLM skip instead of improvise is worth the extra manual triage.

---

## Why Apify and not a direct Facebook scrape

Facebook's anti-bot defences shift constantly. Maintaining a scraper that survives contact with their rate-limiting and challenge pages is a full-time job on its own. Apify does that work as a managed service — I'm effectively buying "someone else keeps up with Facebook's changes" as a line item.

Higher per-lead cost than a raw scrape. Much lower maintenance burden. Worth it.

---

## Why Gemini for generation, not OpenAI

Two reasons. Long-context behaviour on multi-post Facebook summaries plus review text plus site context was better in my testing — OpenAI models in the same price tier tended to latch onto the first signal they saw and ignore the rest. And I was already using OpenAI for the cheap routing step, so splitting models kept the spend on the expensive step isolated from the routing spend.

Not a religious choice. If the qualitative gap closes I'd switch.

---

## Tools

- **Apify** — Facebook post extraction (last ~4 months)
- **OpenAI `gpt-4.1-nano`** — cheap link disambiguation
- **Google Gemini** — opener generation
- **Regex / JS `Code` nodes** — link cleanup, Markdown stripping
- **Google Sheets** — shared state across all three workflows

---

## What's deliberately not here

- No A/B testing on opener variants — one generation per lead
- No reply tracking — the mailing tool downstream handles that
- No retry on Apify timeouts — the lead drops to `VERIF_NEEDED` and I deal with it manually
- No deliverability tooling (warm-up, bounce handling, SPF/DKIM checks) — all of that sits in the mailing tool, not here

---

---

# 🇵🇱 Wersja polska

Trzeci i ostatni etap pipeline'u n8n. Bierze leady, które przeszły scraper i qualifier, szuka świeżego sygnału operacyjnego z Facebooka kliniki, a jeśli Facebook jest martwy, przechodzi na sygnał z recenzji Google. Generuje krótki opener cold-maila nawiązujący do tego sygnału. Jeśli żaden sygnał nie jest wystarczająco mocny, workflow zapisuje kod błędu zamiast wysyłać marnego maila.

## Po co to istnieje

Cold mail zaczynający się od "Super strona!" ma zerową wiarygodność. Generator ma napisać jedno konkretne zdanie, które udowadnia, że nadawca faktycznie spojrzał na leada przed pisaniem. To wymaga prawdziwego, świeżego, konkretnego sygnału — i odmowy wysyłki, jeśli takiego sygnału nie ma.

## Jak to działa

Regex wyciąga linki do Facebooka ze strony → `gpt-4.1-nano` wybiera właściwy lokalny profil, jeśli jest ich kilka → sprawdzenie pod kątem konkurencji i vendorów → rozgałęzienie: ścieżka FB (priorytet — Apify scrapuje ~4 miesiące postów) albo ścieżka recenzji Google (fallback) → Gemini generuje opener w formacie "widziałem X, co zwykle powoduje Y" z twardym limitem 35 słów → bramka walidacyjna: jeśli opener jest czysty, trafia do `CRM` i `EXPORT_READY`; jeśli LLM zwraca `ERROR` albo `PASS`, lead ląduje w `VERIF_NEEDED` do ręcznego przejrzenia.

## Stack

n8n (self-hosted, Docker) · Apify · OpenAI `gpt-4.1-nano` · Google Gemini · Google Sheets

## License

Source-available for reference. Not licensed for redistribution or commercial use.
