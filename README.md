# Who Sues Over Racial Profiling? Extracting Plaintiff Demographics from Court Complaints with a Validated RAG Pipeline

> **Work in progress.** The full collection contains over 1,100 complaints. The pipeline has been built and validated on a first batch of 464 complaints; all results below describe that batch. Processing the remaining complaints and the encounter variables are the next iterations.

An end-to-end pipeline that turns 664 messy court downloads into a structured dataset of **plaintiffs' gender, race and age** in racial-profiling and police-misconduct lawsuits. Encounter variables (setting, traffic stop) are extracted experimentally and are the focus of the next iteration. Every extracted value is tied to a verbatim quote and page, and precision, recall, F1 and accuracy are measured against hand-coded data.

| | |
|---|---|
| **Corpus** | First batch: 664 PDF downloads → **464 complaints** (2018–2025) → **640 individual plaintiffs**, of over 1,100 complaints in the full collection |
| **Method** | Hybrid retrieval (BM25 + dense embeddings + cross-encoder reranker), GPT-4o structured extraction with quote-level citation checks, a targeted second pass for missing demographics (version 2) and race-specific fixes (version 2.1) |
| **Validation** | A hand-coded gold set (version 1), a full-library quality check, and two fresh evaluation samples (versions 2 and 2.1): about 200 hand-checked rows, plus human review of every value added after version 1 |
| **Cost and speed** | About $0.05 per complaint; the full library in about 100 minutes on a T4 GPU (about 13 seconds per complaint) |

## The Problem

Research on racial profiling litigation depends on coding court complaints by hand: reading each filing to record who the plaintiffs are and what happened. This takes minutes to hours per complaint and doesn't scale to hundreds of filings. Large language models read quickly, but they can return plausible values the document doesn't support. In a dataset recording race and gender, an unsupported value is worse than a missing one, because nobody can see it's wrong. This pipeline extracts the values automatically **and** checks each one against the text before accepting it.

---

## Key Results

### Accuracy

| Variable | Accuracy vs. gold set (version 1) | Final recall (95% CI) | Final precision | **F1** (95% CI) |
|---|---|---|---|---|
| **Gender** | 91% (73–98%) | 74% → **93%** (90–96%) | 99% | **0.96** (0.94–0.98) |
| **Race** | 96% (79–99%) | 78% → **80%** (73–87%) | 95% | **0.87** (0.84–0.91) |
| **Age** | 100% (86–100%) | not estimated¹ | 100% (89–100%) | — |

*Gold-set accuracy is the share of plaintiffs where the pipeline matched the hand coding, including correct "not stated" answers (19 complaints, 23 plaintiffs). Final recall and precision cover the whole automated pipeline (versions 1–2.1). They are estimated from fresh hand-checked samples: about 30 recall rows and 20 precision rows per variable in version 2, and 67 more race rows in version 2.1. Precision combines the gold-set precision of version-1 values with the precision of values added later. Accuracy on the final dataset is 93% for gender and 86% for race.*

*¹ Age is stated for only 15% of plaintiffs, too few for a random sample of "not stated" values to measure recall (the gold set had 5 stated ages, all found). Age precision comes from a random sample of 30 of the 99 stated ages, each checked against the PDF: all 30 were correct. A full-text scan for missed ages is planned for the next iteration.*

**In short:** when the pipeline reports a demographic value, it is almost always right: precision is 99% for gender and 95% for race. Version 1 missed about a quarter of stated values, so I added a targeted second pass. It raised gender recall from 74% to 93% and brought race to about 80%, with a narrower confidence interval (see Stages 5 and 6 below).

### Who the Plaintiffs Are

Among the **640 individual plaintiffs** (organizations, estates and one misfiled document excluded):

| | Stated in the complaint | Distribution, where stated |
|---|---|---|
| **Gender** | 88% (represented 89%, self-represented 79%) | **73% male**, 27% female, 2 nonbinary |
| **Race** | 55% (represented 55%, self-represented 59%) | **77% Black**, 10% Hispanic/Latino, 4% White, 4% person of color (unspecified), 1% multiple; small numbers of Asian, Middle Eastern/North African, Native American and Pacific Islander |
| **Age** | 15% (99 plaintiffs) | median **34**; range 1–83; 11 under 18, 11 aged 65 or over |

---

## Pipeline

```mermaid
flowchart LR
    A[664 PDF downloads] --> B[1. Sorting<br/>one complaint per case]
    B --> C[Screening<br/>wrong documents, page ranges]
    C --> D[OCR<br/>scanned complaints]
    D --> E[2. RAG extraction<br/>quotes + citation checks]
    E --> F[3. Validation<br/>gold set]
    F --> G[4. Quality check<br/>full library]
    G --> H[5. Version 2<br/>targeted second pass]
    H --> J[6. Version 2.1<br/>race fixes]
    J --> I[Final dataset<br/>values + sources]
```

### Stage 1: Sorting (`01_Separate_Complaints.ipynb`)

Court downloads mix complaints with dockets, orders, motions, exhibits and provider cover pages, often several documents in one PDF. The sorting notebook classifies each file (and page ranges within bundles), picks the most recent complaint per case, renames files to a standard pattern, and records every decision in a manifest, with hand overrides where the rules fail.

- **664 PDFs → 484 cases → 464 complaints** (20 cases had no complaint in the download).
- **Screening:** all 464 selected complaints were screened; 72 were flagged and checked, **2 wrong documents were removed** (a magistrate's recommendation and a motion) and **1 page range was corrected**.
- **OCR:** 22 complaints were image-only scans whose court header stamp hid the missing text layer. They were OCR'd (OCRmyPDF/Tesseract) and re-extracted.

### Stage 2: Extraction (`02_Complaint_RAG_Extraction.ipynb`)

For each complaint:

1. **Cleaning.** Court (ECF) headers, pleading-paper line numbers and repeated footers are removed at ingestion, because they create competing page numbers and noise. The case number and filing date in the ECF header are kept.
2. **Chunking.** Each page is split into passages of about **350 tokens with a 50-token overlap**. Complaints are numbered paragraphs, each usually describing one allegation or one party, so the splitter breaks at paragraphs first, then lines, keeping a party's description whole. Small chunks keep short facts from being diluted by pages of allegations. The overlap keeps a plaintiff's name and description together when they straddle a boundary. Chunks never cross pages, so every citation points to one checkable page.
3. **Indexing.** The first 60 pages are indexed with **BM25** and **dense embeddings** (`text-embedding-3-small`). Complaints state the parties and the encounter early: of 668 verified age, gender and race values, 95% were cited from the first 20 pages and only 4 from pages 51–60. Most complaints are shorter than 60 pages and are indexed in full; in longer files, the later pages are mostly exhibits and other filings, which add noise and cost without adding party facts.
4. **Retrieval.** A plaintiff's race may appear in one sentence out of hundreds. Each group of fields (parties, setting, claims) is searched with its own phrasings, so the long, repetitive allegations don't crowd out the short parties section.
   - **BM25** matches exact terms: names, case numbers, legal phrases such as *"official capacity"*.
   - **Semantic search with MMR** matches paraphrases (*"African American"*, *"a Black man"*, *"a 34-year-old"*) and adds variety among results.
   - **A cross-encoder reranker** (`BAAI/bge-reranker-base`) reads each query together with each candidate and keeps the best 8. It is more accurate than embedding similarity, and fast enough for a few dozen candidates.
   - Caption pages are always included. In testing, this found a plaintiff's race on the last page of a 53-page PDF.
5. **Extraction.** **GPT-4o** (temperature 0) returns structured JSON: for every value, a **verbatim quote** of up to 40 words, the **page**, and a **basis** (explicit, inferred or not stated). Requiring a quote turns each answer into a claim code can check. Excerpts are labeled only with page numbers, so the page is the only number the model can cite.
6. **Citation check.** Every value must pass four checks:

   | Check | What it verifies |
   |---|---|
   | **Page** | The cited page was among the excerpts the model received |
   | **Quote** | The quote appears on that page (ignoring case, spacing and hyphenation; more tolerant for OCR text) |
   | **Support** | The quote contains the value: the race term as written, the stated age, a gender term |
   | **Subject** | The evidence term sits within a few words of the plaintiff's name or "Plaintiff" |

   Failed values go back to the model once with the reason; values that still fail are recorded as *failed*, with the reason, for review. The subject check was added after errors in testing: *"A **female officer** first asked if Plaintiff owns a gun"* had been accepted as the plaintiff's gender. A separate check, independent of the model, flags defendants named in the complaint but missing from the extraction. Every value, with its page, quote and check results, is kept in an audit table.

**Model choices.** GPT-4o follows detailed coding rules (explicit versus inferred, group statements, no inference from names) more reliably than smaller models, and is pinned because it is the model validated on the gold set. Retrieval uses the general-purpose `text-embedding-3-small` rather than a legal-domain embedder (e.g., Kanon 2, voyage-law-2): those lead benchmarks for searching case law across large collections, while this task searches one complaint at a time for plainly worded facts, with BM25 and the reranker covering exact terms.

Runs are parallel (4 workers) and resumable; each complaint's result is saved as it completes.

### Stage 3: Validation of Version 1 (`03_Validation.ipynb`)

A **gold set** of 21 randomly drawn complaints was hand-coded from the PDFs and compared with the pipeline's output field by field (19 complaints could be compared, covering 23 plaintiffs; results in [Key Results](#key-results)). The pipeline reported no incorrect gender or age, and its only race error was an incomplete value. The notebook also reports precision, recall and F1 for stated values and Cohen's kappa. It also covers the experimental encounter variables, which agreed less well with the hand coding (79% for traffic stop, 47% for location type) and are left for the next iteration.

All **34 disagreements were adjudicated** against the source text. The review corrected 11 errors in the hand coding, found 2 definitional differences, and traced the 21 pipeline errors to specific causes:

| Cause | Errors |
|---|---|
| Setting check too strict: correct values discarded | 8 |
| Retrieval missed the passage | 5 |
| Wrong incident or passage | 4 |
| Traffic stop wrong because the setting was missing | 3 |
| Partial race value (one of two stated races) | 1 |

These findings shaped version 2: a full-text second pass for gender and race, and recovery of the setting values the check had discarded.

### Stage 4: Quality Check of the Full Library

A structured review of the full-library output: unmapped race wordings (18), all "White" values (12), plaintiffs without a name after OCR (10), non-standard gender values (10), ages (9), plaintiffs in the 22 OCR'd complaints (23), and a **stratified random sample of 40 "not stated" values** to estimate recall. **73 corrections** were applied, each with its evidence. The sample showed that version 1 missed about a quarter of stated values (estimated recall about **74% for gender, 78% for race**).

### Stage 5: Version 2, a Targeted Second Pass (`04_Second_Pass_v2.ipynb`)

Version 2 leaves the validated version-1 results unchanged and **adds** values where they are missing:

1. **Full-text candidate scan:** for each plaintiff with a missing value, sentences from the whole complaint that mention the plaintiff (name, surname, Mr./Ms. Surname; "Plaintiff" in single-plaintiff complaints; "I" in self-written ones) together with a gender or race cue, plus group statements about "Plaintiffs."
2. **One GPT-4o call per plaintiff** (temperature 0, cached), with the coding rules in the prompt.
3. **Code safeguards** reject a value when its quote isn't among the candidates; when a group statement covers only some plaintiffs (*"the plaintiffs **who were arrested** are both African American women"*); when a pronoun stands for someone acting **on** the plaintiff (*"**he** informed the plaintiff"*); when a pronoun sits next to several plaintiffs' names; or, for race, when the quote describes a **population** (*"the rights of African Americans"*) or a quoted title.
4. **Human review** of all 124 new values (102 kept).
5. **Evaluation on fresh data:** a new stratified sample of plaintiffs still without a value (recall) and a sample of added values (precision).

| | Gender | Race |
|---|---|---|
| Values added | 81 | 21 |
| Model precision **before** safeguards and review | 81 of 83 (≈98%) | 21 of 41 (≈51%) |
| Precision **after** safeguards and review (evaluation sample) | 19 of 20 (95%) | 20 of 20 (100%) |
| Estimated recall, version 1 → version 2 | 74% → **93%** (90–96%) | 78% → **81%** (70–95%) |

For **gender**, most misses were pronouns next to the plaintiff's name that retrieval hadn't surfaced: exactly what a full-text scan finds. For **race**, the gain was small. Race is often stated **indirectly** (*"the racial minority of Plaintiff [name]"*), inside **population sentences**, or deep in **long self-written filings**, and the model on its own confused statements about populations with statements about the plaintiff. The safeguards made race additions reliable, at the cost of rejecting some correct ones.

Setting values that version 1 had discarded although their quote was verified (101 cases) were recovered and marked *for review*.

### Stage 6: Version 2.1, Targeted Fixes for Race (same notebook)

Version 2.1 addressed each race miss found in the version-2 evaluation: race terms **attached** to "plaintiff(s)" inside population sentences; indirect statements (*"the racial minority of Plaintiff X," "not white"*) coded *person of color*; attached sentences ranked first in long filings; "Race: X" fields on forms; a **per-page OCR screen**, which found 8 partly scanned complaints the first screen had missed; and a guard against confusing the plaintiff with a relative or decedent who shares the surname.

The fixes added only **3 values**. A fresh, larger sample (67 rows) put race recall at **80% (73–87%)**: no measurable change, but a much narrower interval than before (70–95%). The 13 misses fell into small groups, each needing its own fix:

| Type of miss | Example | Count |
|---|---|---|
| "Minority" phrasing about the plaintiff | *"had he not been of a minority race"* | 6 |
| Latin American origin terms | *"Mexican-American," "a Venezuelan man"* | 3 |
| First person without a recognized name | *"myself and other African American"* | 2 |
| Description led by a pronoun | *"…is Ms. [name]'s father. **He** is a 61-year-old, Black man"* | 1 |
| Handwritten form | "Race: Black," handwritten | 1 |

**The lesson:** the remaining race gap reflects how complaints are written (race is often stated indirectly, or not at all) more than a single fixable weakness. Nearly half the misses (6 of 13) are indirect statements that the model reads cautiously and that a stricter coding rule would exclude anyway.

---

## Coding Rules (abridged)

- **Gender:** explicit terms (*"man," "mother," "a nonbinary person"*), or inferred from pronouns and titles that clearly refer to the plaintiff. Where these conflict with another description, coding follows the plaintiff's stated pronouns and title, with a note. Never inferred from names or suffixes (Jr., Sr.).
- **Race:** coded when the complaint describes the plaintiff (explicit), or when a statement about the plaintiff clearly implies it (*"if I were a white individual," "Plaintiff is not white"*: inferred). Never from names, places, languages, statements about populations, or descriptions of other people (officers, suspects, decedents, relatives). A stated Latin American origin is coded Hispanic/Latino, as ethnicity. Race is recorded in the complaint's own words and coded with a multi-label scheme based on U.S. Census groups: *"Afro-Latina"* is coded both Black and Hispanic/Latino.
- **Age:** recorded only when stated, never calculated from dates; *"over the age of eighteen"* is not an age.
- **Group statements** (*"Plaintiffs are Black"*) apply to every plaintiff; statements about a subset apply only to that subset.
- **Exhibits** filed with the complaint (court records, the plaintiff's own forms) count when they describe the plaintiff, with the source noted.
- **Representative cases** (wrongful death, next friend): the decedent's or child's demographics are not assigned to the plaintiffs suing on their behalf.

---

## Limitations

- **First batch only.** The results describe 464 of over 1,100 complaints in the full collection. The shares may change as the remaining complaints are processed.
- **Complaints filed, not all encounters.** The data describe plaintiffs and incidents in complaints alleging racial profiling, not everyone police stop. Several filters stand between an encounter and this dataset: whether the person sued, whether they found a lawyer or filed pro se, which cases reached court, and which filings the Bloomberg Law search returned. The shares reported here (e.g., 77% of plaintiffs with a stated race are Black) describe who brings these claims to court, not who is stopped or how often profiling happens. The incidents are allegations from one side, not findings of fact.
- **Race is often unstated, even in racial-profiling cases.** Many complaints allege discrimination "based on race" without naming the plaintiff's race; a common New York state-court template never does.
- **Self-represented complaints** (about 14% of complaints) are often handwritten or on court forms, in the first person; handwriting is read poorly even after OCR, and several were coded by hand.
- **Proximity is a heuristic.** The subject check blocks most attributes of other people, but a phrase like *"Plaintiffs, along with a white officer…"* can still pass; the safeguards and human review catch some of these, not all.
- **Repeat filers and repeat incidents.** One plaintiff filed at least six near-identical complaints in different courts, and one encounter can produce several lawsuits; counts by complaint are not counts of people or incidents.
- **"Date filed" is the date of the document**, not of the case; for amended complaints, the case was opened earlier.

**Ethics.** Race, gender and age are sensitive even in public court filings. The pipeline records them only when the complaint states them, in its own words, with the supporting quote kept for every value.

---

## Next Steps

- **Process the full collection** (over 1,100 complaints) with the validated pipeline, and check accuracy on a fresh sample from the new complaints.
- **Validate the location variables.** Version 1 agreed with the gold set only 47% of the time on location type. Next: review the 101 settings recovered in version 2, keep quote-verified values in the setting check, and test on new complaints.
- **Validate traffic stop:** define it (moving or parked vehicle, passengers, incidents with several stages), run a targeted pass on blank values, and report sensitivity and specificity on a fresh sample.
- **Compare models and add an LLM judge:** accuracy, cost and latency across models, plus an LLM judge validated against the human labels collected here.
- **Link to external data**, such as population and stop data by agency, to turn complaint counts into rates.

---

## Repository Structure

```
├── notebooks/
│   ├── 01_Separate_Complaints.ipynb       # sorting, screening, OCR check, gold-set preparation
│   ├── 02_Complaint_RAG_Extraction.ipynb  # hybrid RAG extraction, citation checks, OCR, batch runs
│   ├── 03_Validation.ipynb                # gold-set comparison, adjudication, metrics, error analysis
│   └── 04_Second_Pass_v2.ipynb            # second pass (v2), race fixes (v2.1), safeguards, evaluations, F1, final dataset
├── results/
│   └── summary_tables.md                  # aggregate results (no case-level data)
├── requirements.txt
├── LICENSE
└── README.md
```

**Data are not included.** The complaints are public court records, but the case-level dataset contains names and quoted allegations about individuals and is kept private. The repository contains code and aggregate results only; notebook outputs are cleared, and no API keys are stored.

## Running the Pipeline

The notebooks run in Google Colab, with files in Google Drive; a T4 GPU is recommended for the reranker. You need an OpenAI API key: notebook 02 reads it from a `config.json` you upload to the session (`{"OPENAI_API_KEY": "sk-..."}`), and notebook 04 asks for it at runtime. Keys are never stored in the notebooks. Run the notebooks in order: sorting → extraction (a test run of 5 complaints, then the full library) → validation → version 2. Each long step is resumable, and model answers are cached, so re-running doesn't repeat paid calls. Case-level corrections (misread names, files that turned out not to be complaints) are read from a private `private_overrides.json` kept with the results; without it, the notebook runs and skips them.

Main components: LangChain, ChromaDB, PyMuPDF, `rank_bm25`, `sentence-transformers` (reranker), OCRmyPDF/Tesseract, OpenAI (`gpt-4o`, `text-embedding-3-small`).

---

**Author:** Dariya Tsyrenzhapova, Ph.D.
