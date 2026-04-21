# Lead Qualification & Outreach Pipeline (n8n)

Solo project. End-to-end n8n pipeline that takes raw Google Maps data, filters it down to independent physiotherapy clinic owners, and generates cold outreach emails grounded in a recent business signal.

Sanitized public extract of a private project — credentials, live prompts, and target domains removed. Commit history lives in the private source repo.

---

## Scope & status

- Self-hosted n8n via Docker. Ran in production long enough to validate the pipeline end-to-end on real leads.
- Three separate workflows, coupled by Google Sheets as the shared store.
- AI-assisted during development (Claude Code for iteration, n8n's own AI nodes for orchestration). The architecture, gating logic, and prompt constraints are my design.

---

## Why this exists

Standard lead-gen tools scrape everything, pay for expensive LLM calls on every row, and still produce generic "loved your website!" cold emails. I wanted the opposite: cheap filtering upfront, LLM spend only on leads that pass hard gates, and emails that reference something concrete and recent.

The pipeline is structured as a waterfall — each stage has the right to drop a lead. By the time a lead reaches the email generator, it's been through two filtering layers and a dedicated signal-detection step.

---

## Architecture

Three workflows, each a separate n8n export:

### 1. Scraper-Sheets — ingest & filter

- `gosom/google-maps-scraper` in Docker pulls business data + reviews from Google Maps for a target query/region
- n8n parses the newline-delimited JSON output
- Drops anything with no website, rating under 4.0, or "Closed" status
- Strips review payloads down to name + text only (token cost control for later stages)
- Appends survivors to Google Sheets

### 2. New_Qualifier — verify it's a real independent owner

This is the expensive stage, so it runs only on pre-filtered rows.

- Firecrawl maps the full site structure for each lead
- A cheap LLM (OpenAI, small model) picks which subpages are most likely to contain owner/team info (`/about`, `/team`, `/meet-the-team`, etc.)
- Only those pages get scraped, converted to Markdown to strip nav/footer noise
- The aggregated Markdown goes to Gemini with a hard-constrained prompt that must:
  - Reject national chains, franchises, and vendor platforms
  - Confirm the "owner" title appears in the site text, not just inferred from reviews
  - Confirm the clinic's primary focus is physiotherapy, not general wellness

If any rule fails, the lead is dropped before it ever hits the email stage.

### 3. email_gen — write the outreach

Signal-first, template-free. The LLM is explicitly told it cannot send an email without a concrete recent signal to reference.

Signal hierarchy:

1. **Operational (priority):** Apify scrapes the last ~4 months of Facebook posts. Looks for new hires, workshops, extended hours, holiday closures.
2. **Reputation (fallback):** If socials are dead, analyze 5-star Google reviews for chronic signals — staff praise, front-desk bottleneck mentions, high traffic.

The generator's job is to build a single-sentence bridge: "I saw X, which usually causes Y" — where Y is a specific admin pain the offered chatbot solves. No generic compliments allowed by the prompt.

If no signal is strong enough, the node returns `ERROR` and the lead is skipped instead of sending a weak email. This was added after seeing the first batch of emails drift toward "love what you're doing!" filler when no real hook existed.

---

## Tech stack

- **Orchestration:** n8n, self-hosted via Docker
- **Store:** Google Sheets (simple, shared across workflows, good enough at this scale)
- **Scraping:** `gosom/google-maps-scraper` (Docker), Firecrawl (site mapping), Apify (Facebook posts)
- **LLMs:**
  - Google Gemini for the qualification reasoning and email generation (better long-context behavior for multi-page Markdown input)
  - OpenAI small model for cheap routing decisions (subpage selection)

---

## What's not in the public version

- Actual prompts (the constraint rules are the IP here)
- Client list / target regions
- API keys and n8n credentials
- The Sheets schema

The workflow JSON files show node structure and wiring but are sanitized.

---

## What's deliberately not here

- No automated tests — n8n workflow logic is tested by running it on known good/bad leads
- No CI/CD — this is a single-host Docker setup, deployed by restart
- No queueing infra — n8n's own execution queue is sufficient for the volume

---

---

# 🇵🇱 Wersja polska

**Solo project.** Pipeline n8n który bierze surowe dane z Google Maps, filtruje je do niezależnych właścicieli klinik fizjoterapii i generuje cold maile oparte na konkretnym, świeżym sygnale biznesowym.

Publiczny, oczyszczony wyciąg z prywatnego projektu — credentiale, prawdziwe prompty i domeny usunięte.

## Po co to istnieje

Standardowe narzędzia do lead-genu scrapują wszystko, płacą za drogie wywołania LLM na każdym rekordzie i i tak wypluwają generyczne "podoba mi się twoja strona!". Chciałem odwrotnie: tanie filtrowanie z góry, wydatek na LLM tylko na leadach które przeszły twarde bramki, i maile referujące coś konkretnego i świeżego.

Struktura to kaskada — każdy etap ma prawo odrzucić leada. Zanim lead dociera do generatora maili, przeszedł przez dwie warstwy filtrowania i osobny etap detekcji sygnału.

## Architektura (skrót)

1. **Scraper-Sheets** — scraper Google Maps w Dockerze, odrzuca firmy bez strony / z oceną <4.0 / zamknięte, zapisuje do Sheets
2. **New_Qualifier** — Firecrawl mapuje stronę, tani LLM wybiera podstrony o właścicielu, Gemini weryfikuje że to niezależna klinika fizjoterapii (nie sieciówka, nie wellness, tytuł właściciela musi być w tekście strony)
3. **email_gen** — detektor sygnału (Facebook przez Apify, fallback do recenzji Google), generator musi zbudować zdanie "widziałem X, co zwykle powoduje Y". Bez sygnału zwraca ERROR zamiast wysyłać generyka.

## Stack

n8n (self-hosted, Docker) · Google Sheets · `gosom/google-maps-scraper` · Firecrawl · Apify · Google Gemini · OpenAI (small model do routingu)

## License

Source-available for reference. Not licensed for redistribution or commercial use.
