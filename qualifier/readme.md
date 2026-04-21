# Qualifier Workflow

Second stage of the n8n pipeline. Takes pre-filtered Google Maps leads from the scraper, maps each lead's website with Firecrawl, picks the few subpages most likely to contain owner or team info, and runs the aggregated text through Gemini against a strict disqualification script.

Sanitized public extract of a private project. Commit history lives in the private source repo.

---

## Scope & status

Sub-workflow — triggered by the scraper workflow upstream via `Execute Workflow`. This is the expensive stage (two LLMs, Firecrawl, multiple HTTP scrapes per lead), so it only runs on rows that have already cleared the cheap filters.

AI-assisted during development (Claude Code for scaffolding and iteration). The disqualification rules, the page-selection prompt, and the owner cross-reference logic are my design and were tuned iteratively on real leads.

---

## Why this stage exists

The whole pipeline is looking for independent physiotherapy clinic owners — not chains, not franchises, not vendor platforms that happen to use the right keywords in their copy. That call can't be made from Google Maps metadata alone; you have to actually read the site.

But reading every page of every site with an LLM is wasteful. Most of a clinic's site is services and booking copy. Owner and team info, if it exists at all, lives on one to three specific pages. So this workflow has a cheap routing step before the expensive analysis step.

---

## Architecture

![Qualifier workflow](./qualifier.png)

### Input gating

- Reads rows from the `CRM` sheet where the `Clean Name` column is empty (so the workflow never re-processes a lead)
- If `Company Website` is empty, marks the row `NO_WEBSITE` and stops

### Page selection — cheap model

- **Firecrawl** (`POST /v1/map`) crawls the target domain and returns every URL it can reach
- A cheap model (`gpt-5-nano`) receives the full URL list and picks the top 5 URLs most likely to contain owner or team info — prioritises `/about`, `/team`, `/staff`, `/leadership` and ignores `/login`, `/blog`, `/contact`
- This step exists purely to avoid running Gemini on 40 pages of pricing and booking copy

### Page scraping — targeted

- Loops over the 5 selected URLs
- `HTTP GET` with browser User-Agent headers, HTML → Markdown via n8n's `Markdown` node
- A small JS `Code` node strips navigation, footers, copyright lines, and social links — the Markdown conversion leaves a lot of boilerplate that would eat tokens at the next step
- Aggregates the cleaned text from all 5 pages into one context block

### Qualification — Gemini with a script

- **Google Gemini** (`gemini-2.0-flash`, temperature `0.2`) takes the aggregated Markdown plus the Google Maps reviews and applies three checks:
  1. **Hard disqualifiers** — rejects vendors, consulting firms, franchises, and chains with more than 15 locations
  2. **Signal dominance** — physiotherapy keywords must outweigh generic "therapy" (speech, occupational, mental health) so the workflow doesn't accidentally qualify the wrong discipline
  3. **Owner identification** — must find `Owner` / `Founder` / `CEO` in the site text, then cross-reference against names praised in Google reviews before returning a name
- Returns either the owner's full name or a standardised SKIP reason (`SKIP: Chain clinic`, `SKIP: Cannot verify owner from website data`, `SKIP: Insufficient depth in physical therapy services`, etc.)

### Routing

| Outcome | Destination sheet | Data recorded |
| --- | --- | --- |
| Qualified | `THE QUALIFIER` | Owner name, website, reviews, city, country |
| Disqualified | `VERIF_NEEDED` | SKIP reason, website, reviews |
| Either way | `CRM` (main sheet) | Status update, so the lead never reprocesses |

---

## Why the two-step AI (cheap router + strict judge)

The naive version is to dump the whole site into Gemini and let it figure out what matters. That works but burns tokens on navigation, pricing, archive pages, and blog posts. Splitting it into "pick the pages, then read the pages" means Gemini sees a tighter context and the cheap model absorbs the routing cost.

The owner cross-reference between site text and review names was added after seeing Gemini confidently return names that only appeared in reviews — which doesn't prove ownership, just that a practitioner gets praised. Requiring the name to also appear on the site killed that failure mode.

---

## What's deliberately not here

- No automated test harness — verified by running against a set of known-good and known-bad leads
- No retry on Firecrawl failures — a failed map call sends the lead to `VERIF_NEEDED`
- No PII storage beyond what Google Maps already exposes publicly
- No caching of Firecrawl results — re-qualifying a lead re-crawls

---

---

# 🇵🇱 Wersja polska

Drugi etap pipeline'u n8n. Bierze wstępnie przefiltrowane leady ze scrapera, mapuje stronę każdego z nich przez Firecrawl, wybiera te podstrony, które najpewniej zawierają informacje o właścicielu lub zespole, i analizuje ich treść przez Gemini według ścisłego zestawu reguł.

## Po co ten etap

Cały pipeline szuka niezależnych właścicieli klinik fizjoterapii — nie sieciówek, nie franczyz, nie platform vendorów, które używają odpowiednich słów kluczowych. Tego nie da się rozstrzygnąć z samych metadanych Google Maps, trzeba przeczytać stronę. Ale wrzucanie całej strony do LLM to marnotrawstwo — informacje o właścicielu (jeśli w ogóle istnieją) są na 1–3 konkretnych podstronach. Stąd tani model wybiera podstrony, a droższy model je waliduje.

## Jak to działa

Firecrawl mapuje całą domenę → tani model (`gpt-5-nano`) wybiera 5 URL-i najprawdopodobniej zawierających informacje o właścicielu lub zespole → te 5 stron zostaje zescrapowanych z nagłówkami przeglądarki, zamienionych na Markdown i oczyszczonych z boilerplate'u → Gemini (`gemini-2.0-flash`, temperatura 0.2) stosuje trzy twarde reguły: dyskwalifikatory, dominacja fizjoterapii nad innymi terapiami, weryfikacja właściciela przez porównanie nazwisk ze strony z nazwiskami z recenzji → wynik trafia do `THE QUALIFIER` albo `VERIF_NEEDED`, a `CRM` dostaje aktualizację statusu.

## Stack

n8n (self-hosted, Docker) · Firecrawl · OpenAI `gpt-5-nano` · Google Gemini `gemini-2.0-flash` · Google Sheets

## License

Source-available for reference. Not licensed for redistribution or commercial use.
