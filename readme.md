# Autonomiczny Pipeline Kwalifikacji i Outreachu Leadów ----(Scroll for English)----

To repo zawiera kompletny zestaw automatyzacji end-to-end, zaprojektowany, aby przekształcić surowe dane z Google Maps w hiper-spersonalizowany, cold outreach "ludzkiej jakości" oferując chatboty dla klinik fizjoterapii.

W przeciwieństwie do tradycyjnych scraperów, które zalewają CRM-y kontaktami z zupełnie innej bajki, ten system działa z precyzją człowieka – od scrapowania po pisanie maili. Priorytetem jest tu **sygnał nad szumem** i **trafność nad ilością**, co gwarantuje, że każdy wygenerowany email opiera się na zweryfikowanej, operacyjnej rzeczywistości.

## Główna Filozofia

Cel tej automatyzacji najlepiej podsumowuje hasło: **kosztowo efektywna inteligencja**.

Odrzuciliśmy standardowe podejście polegające na rzucaniu drogich LLM-ów na surowe dane. Zamiast tego zbudowaliśmy system typu waterfall (kaskadowy), gdzie dane muszą "zasłużyć" na przetworzenie przez kolejną, droższą warstwę. Lead jest wzbogacany tylko wtedy, gdy przejdzie przez ścisłe bramki kwalifikacyjne, a email jest pisany tylko wtedy, gdy wykryty zostanie legitny sygnał biznesowy (jak nowe zatrudnienie czy niedawna recenzja).

Wynikiem jest pipeline, który działa za grosze na leada, ale wypluwa copy, które unosi brwi, bo wygląda na rzetelnie zresearchowane.

## Architektura

System składa się z trzech odrębnych, powiązanych ze sobą workflow-ów.

### 1. Sieć: Wprowadzenie i Wstępne Czyszczenie (The Dragnet)
**Workflow:** `Scraper-Sheets`

Proces zaczyna się od scrapera Google Maps uruchamianego w Dockerze (`gosom/google-maps-scraper`). To narzędzie zarzuca szeroką sieć, wyciągając surowe dane biznesowe i – co kluczowe – szczegółowe recenzje użytkowników z docelowych obszarów.

Gdy surowy JSON zostanie wygenerowany, ten workflow działa jak bramkarz:
* **Wprowadzenie:** Parsuje masywne pliki JSON (newline-delimited).
* **Sanityzacja:** Natychmiast odrzuca firmy "duchy" – te bez strony www, ze słabymi ocenami (<4.0) lub statusem "Zamknięte".
* **Kompresja:** Obcina dane recenzji do absolutnego minimum (Imię + Treść), wyrzucając zbędne metadane, aby oszczędzać tokeny na dalszych etapach.
* **Storage:** Zakwalifikowane, surowe leady są doklejane do CRM-a (Google Sheets) na potrzeby kolejnej fazy.

### 2. Sędzia: Głęboka Kwalifikacja i Mapowanie Strony (The Judge)
**Workflow:** `New_Qualifier`

To "mózg" operacji. Zakłada, że surowe dane mogą być mylące i dąży do ich weryfikacji. Zamiast scrapować tylko stronę główną, system zachowuje się jak człowiek nawigujący po witrynie.

* **Inteligentna Nawigacja:** Używa **Firecrawl** do zmapowania całej struktury strony, a następnie lekkiego LLM-a (OpenAI `gpt-5-nano`), aby zidentyfikować konkretne podstrony, które najprawdopodobniej zawierają informacje o właścicielu (np. `/meet-the-team`, `/our-story`, `/leadership`).
* **Celowana Ekstrakcja:** Scrapuje tylko te wysokowartościowe podstrony, czyszcząc HTML do gęstego Markdowna, aby usunąć szum nawigacyjny.
* **Trybunał:** Zagregowany kontekst trafia do **Google Gemini** ze ścisłym "Promptem Konstytucyjnym". Ten prompt wymusza twarde reguły dyskwalifikacji:
    * Odrzuca ogólnokrajowe sieciówki, franczyzy i usługi vendorów.
    * Weryfikuje tytuł "Właściciela" w tekście strony (zapobiegając halucynacjom opartym wyłącznie na recenzjach).
    * Upewnia się, że klinika skupia się głównie na docelowej niszy (Fizjoterapia), a nie na ogólnym wellness.

Tylko zweryfikowani, niezależni właściciele przechodzą do finałowego etapu.

### 3. Domykacz: Copywriting Oparty na Sygnałach (The Closer)
**Workflow:** `email_gen`

Ostatni workflow to copywriter. Nie używa szablonów. Wykorzystuje **Hierarchię Sygnałów**, aby zbudować "Most" między rzeczywistością prospekta a rozwiązaniem (Automatyzacja Administracji).

* **Wykrywanie Sygnału:**
    * **Priorytet A (Operacje):** Scrapuje posty z ostatnich 4 miesięcy z **Facebooka** używając **Apify**. Szuka *ostrych* wyzwalaczy administracyjnych: nowi pracownicy, warsztaty dla społeczności lub przerwy świąteczne.
    * **Priorytet B (Reputacja):** Jeśli aktywność w socialach jest martwa, spada do **Recenzji Google**. Analizuje 5-gwiazdkowe opinie, aby znaleźć *przewlekłe* wyzwalacze: duża popularność, konkretne pochwały dla personelu lub wzmianki o "zajętej recepcji".
* **Logika "Mostu":**
    * AI ma wyraźny zakaz pisania generycznych komplementów.
    * Musi skonstruować logiczny argument: *"Widziałem [Sygnał X], co zazwyczaj powoduje [Ból Administracyjny Y]."*
    * *Przykład:* "Widziałem, że niedawno dołączył do zespołu dr Smith. Większość właścicieli mówi mi, że taka faza wzrostu zazwyczaj podbija liczbę telefonów do recepcji z pytaniami o grafik."
* **Anty-Halucynacja:** Ostatnia warstwa logiczna waliduje emaila. Jeśli AI nie może znaleźć wystarczająco silnego sygnału, by zbudować sensowny most, zwraca kod `ERROR` zamiast wysyłać słabego, generycznego maila.

## Techstack

* **Orkiestracja:** n8n (Self-hosted)
* **Baza danych:** Google Sheets
* **Scraping:** Docker (`gosom/google-maps-scraper`), Firecrawl (Mapowanie Strony), Apify (Facebook)
* **Inteligencja:**
    * **Google Gemini (Flash/Pro):** Używany do złożonego wnioskowania i kreatywnego copywritingu ze względu na lepsze okno kontekstowe i stosunek jakości do ceny.
    * **OpenAI (GPT-4o-mini):** Używany do szybkich, tanich decyzji routingowych i selekcji linków.




# Autonomous Lead Qualification & Outreach Pipeline

This repository houses a complete, end-to-end automation suite designed to transform raw Google Maps data into hyper-personalized, "human-grade" cold outreach offering AI chatbots for physiotherapy clinics.

Unlike traditional "spray and pray" scrapers that flood CRMs with contacts from a completely different industry, this system operates with the precision of a human, from scraping to writing emails. It prioritizes **signal over noise** and **relevance over volume**, ensuring that every email generated is based on verified, operational reality.

## Core Philosophy

The focus of this automation is best summed up as **cost-efficient intelligence**.

We rejected the standard approach of throwing expensive LLMs at raw data. Instead, we built a waterfall system where data must "earn" the right to be processed by the next, more expensive tier. A lead is only enriched if it passes strict qualification gates, and an email is only written if a legitimate business signal (like a new hire or a recent review) is detected.

The result is a pipeline that runs for pennies per lead but outputs copy that raises eyebrows because it feels genuinely researched.

## The Architecture

The system is composed of three distinct, coupled workflows.

### 1. The Dragnet: Ingestion & Initial Cleaning
**Workflow:** `Scraper-Sheets`

The process begins with a self-hosted, Dockerized Google Maps scraper (`gosom/google-maps-scraper`). This tool casts a wide net, pulling raw business data and—crucially—detailed user reviews from target search areas.

Once the raw JSON is generated, this workflow acts as the gatekeeper:
* **Ingestion:** It parses massive newline-delimited JSON files.
* **Sanitization:** It immediately discards "ghost" businesses—those with no website, poor ratings (<4.0), or "Closed" status.
* **Compression:** It strips review data down to the bare essentials (Name + Text), discarding metadata to save token costs downstream.
* **Storage:** Qualified raw leads are appended to the CRM (Google Sheets) for the next phase.

### 2. The Judge: Deep Qualification & Site Mapping
**Workflow:** `New_Qualifier`

This is the "brain" of the operation. It assumes the raw data is misleading and seeks to verify it. Instead of scraping just the homepage, it behaves like a human user navigating a website.

* **Intelligent Navigation:** It uses **Firecrawl** to map the entire site structure, then uses a lightweight LLM (OpenAI `gpt-5-nano`) to identify the specific pages most likely to contain owner information (e.g., `/meet-the-team`, `/our-story`, `/leadership`).
* **Targeted Extraction:** It scrapes only those high-value pages, cleaning the HTML into dense Markdown to remove navigation noise.
* **The Tribunal:** The aggregated context is fed to **Google Gemini** with a strict "Constitutional Prompt." This prompt enforces hard disqualification rules:
    * It rejects nationwide chains, franchises, and vendor services.
    * It verifies the "Owner" title against the website text (preventing hallucinations based solely on reviews).
    * It ensures the clinic is predominantly focused on the target niche (Physical Therapy), not general wellness.

Only verified, independent owners are passed to the final stage.

### 3. The Closer: Signal-Based Copywriting
**Workflow:** ` email_gen`

The final workflow is the copywriter. It does not use templates. It uses a **Signal Hierarchy** to construct a "Bridge" between the prospect's reality and the solution (Admin Automation).

* **Signal Detection:**
    * **Priority A (Operations):** It scrapes the last 4 months of **Facebook** posts using **Apify**. It looks for *acute* admin triggers: new staff hires, community workshops, or holiday closures.
    * **Priority B (Reputation):** If social activity is dead, it falls back to **Google Reviews**. It analyzes 5-star reviews to find *chronic* admin triggers: high popularity, specific praise for staff, or "busy front desk" mentions.
* **The "Bridge" Logic:**
    * The AI is explicitly forbidden from writing generic compliments.
    * It must construct a logical argument: *"I saw [Signal X], which usually causes [Admin Pain Y]."*
    * *Example:* "I saw you recently added Dr. Smith to the team. Most owners tell me that growth phase usually spikes the front-desk call volume with scheduling questions."
* **Anti-Hallucination:** A final logic layer validates the email. If the AI cannot find a strong enough signal to build a valid bridge, it outputs an `ERROR` code rather than sending a weak, generic email.

## The Tech Stack

* **Orchestration:** n8n (Self-hosted)
* **Database:** Google Sheets
* **Scraping:** Docker (`gosom/google-maps-scraper`), Firecrawl (Site Mapping), Apify (Facebook)
* **Intelligence:**
    * **Google Gemini (Flash/Pro):** Used for complex reasoning and creative copywriting due to its superior context window and reasoning-to-cost ratio.
    * **OpenAI (GPT-4o-mini):** Used for fast, low-cost routing decisions and link selection.
