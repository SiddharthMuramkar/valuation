# Damodaran open corpus: inventory and ingest cost

Ticket #2. Researched 2026-11-15. Inventory only, no crawls run. Counts marked ~ are estimates; verify before you build the scraper.

## Sources

| Source | URL pattern | Formats | Approx count | Update cadence |
|---|---|---|---|---|
| Home site | `pages.stern.nyu.edu/~adamodar/New_Home_Page/*.html` (mirror `people.stern.nyu.edu/adamodar/`) | HTML | `firecrawl map` returned 148 URLs (37 .html, 34 .htm, 55 .pdf, 21 .xls/.xlsx, 1 .pptx). The map is shallow; the real site has thousands of files. | ad hoc |
| Damodaran Online datasets | `.../New_Home_Page/data.html` -> `.../pc/datasets/*.xls(x)`, `.../New_Home_Page/datafile/*.html` | XLS/XLSX + HTML mirrors; US, Global, Europe, Japan, EM, China, India; historical archives | ~100 variables x ~7 regions = ~500-700 files, plus yearly archives | annual, first week of January; ERP / country risk also mid-year and monthly |
| Spreadsheets / tools | `.../New_Home_Page/tools.html`, `.../pc/*.xls` | XLS/XLSX models | ~100 | occasional |
| Papers | `.../New_Home_Page/papers.html` -> `.../pdfiles/papers/*.pdf`; SSRN `papers.ssrn.com/sol3/cf_dev/AbsByAuth.cfm?per_id=20838` | PDF | ~100+ on SSRN, ~50 on site (mostly the same papers) | ERP/country-risk papers yearly; others ad hoc |
| Class notes / slides | `.../New_Home_Page/teaching.html`, `.../pdfiles/**.pdf`, `.../pptfiles/` | PDF slides, PPTX, post-class tests | ~6-8 courses x ~30 sessions = ~1-2k PDFs | each semester |
| Blog "Musings on Markets" | `aswathdamodaran.blogspot.com/YYYY/MM/slug.html`; mirror `aswathdamodaran.substack.com/p/slug` | HTML | since 2008, ~40-60 posts/yr, ~800 posts | roughly weekly; "Data Update 1..N" series every January |
| YouTube / podcast | YouTube channel; audio feed `feeds.megaphone.fm/aswath-damodaran` | video; transcripts via captions | ~1k+ videos (Corp Fin 36 + Valuation 25 + Inv Phil playlists, plus full MBA class recordings each semester) | each semester + ad hoc |
| Books | damodaran.com/writing | commercial | - | out of scope |

Sources: pages.stern.nyu.edu/~adamodar/New_Home_Page/{data,tools,papers,guide}.html; SSRN author page per_id=20838; feeds.megaphone.fm/aswath-damodaran (playlist counts); aswathdamodaran.blogspot.com/2026/01 ("Data Update 3 for 2026"); blogspot 2025/01 Data Update 1 ("first week of each year").

## Size guess (text tokens)
- Blog ~800 posts x ~3k = ~2.5M
- Papers ~150 PDFs x ~25k = ~4M
- Slides/notes ~1.5k PDFs x ~8k = ~12M
- Datasets (flattened) = ~5M
- YouTube transcripts ~1k x ~12k = ~12M
- **Total ~35M tokens** (range 20-60M)

## Scrape cost
Units: ~1.5k HTML pages + ~2k PDFs (~40k PDF pages) + ~800 spreadsheets + transcripts.
- **Direct download** (static NYU files, blog archive): $0. Recommended for the PDFs and XLS files, which you then parse locally.
- **Firecrawl** (firecrawl.dev/pricing): 1 credit per page. PDFs likely bill per PDF page (unverified). ~3k HTML/blog pages = ~3k credits (Hobby, $16-19/mo for 5k). Parsing the PDFs too comes to ~45k credits: one month of Standard, $83-99 for 100k.
- **Parallel extract**: ~$1 per 1k URLs (parallel.ai/ai/compare/firecrawl-vs-parallel, 2026-11-03). ~3.5k URLs = **~$4**. PDF fidelity is unverified.
- **Octen extract**: no public per-call price found. Use it only as a fallback.
- **YouTube transcripts**: captions through yt-dlp, $0. ASR is only needed for videos without captions.

## Embedding cost (Voyage)
Source: docs.voyageai.com/docs/pricing (updated 2026-09-29). The first 200M tokens on voyage-4* are free per account; voyage-finance-2 gets 50M free.

| Model | $/M tokens | ~35M tokens |
|---|---|---|
| voyage-4-lite | 0.02 | $0 (free tier); $0.70 list |
| voyage-4 (recommended) | 0.06 | $0 (free tier); $2.10 list |
| voyage-4-large / context-4 | 0.12 | $0 (free tier); $4.20 list |
| voyage-finance-2 | 0.12 | $0 (within 50M free); $4.20 list |
| voyage-3.5 (older gen) | 0.06 | $2.10 |

## Bottom line
One-off total is **under ~$100, and realistically $0-20**: download the files directly, use Parallel or Firecrawl only for HTML, and embed within Voyage's free tier. Cost is not the constraint. The real work is PDF/XLS parsing quality, getting the YouTube transcripts, and the annual January refresh. Check the "Usage rules" section of data.html before redistributing any data.
