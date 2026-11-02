# Summary Tables

Aggregate results of the final dataset (version 2.1) for the first batch of 464 complaints; the full collection contains over 1,100 complaints and is still being processed. No case-level data: counts and rates only.

## Corpus

| Stage | Count |
|---|---|
| PDF downloads | 664 |
| Cases | 484 |
| Complaints (one per case; 20 cases had none) | 464 |
| Self-represented complaints | 65 (14%) |
| Individual plaintiffs (organizations, estates and one misfiled document excluded) | 640 |
| Complaints OCR'd (image-only scans) | 22, plus 8 partly scanned (version 2.1) |

## Extraction Quality

| Variable | Accuracy vs. gold set (version 1) | Final recall (95% CI) | Final precision (95% CI) | F1 (95% CI) | Accuracy, final dataset (95% CI) |
|---|---|---|---|---|---|
| Gender | 91% (73–98%) | 93% (90–96%) | 99% | 0.96 (0.94–0.98) | 93% |
| Race | 96% (79–99%) | 80% (73–87%) | 95% | 0.87 (0.84–0.91) | 86% (81–90%) |
| Age | 100% (86–100%) | not estimated | 100% (89–100%) | — | — |

Gold set: 19 complaints, 23 plaintiffs. Final recall and precision cover versions 1–2.1 and come from fresh hand-checked samples. Age precision: 30 of 30 sampled ages correct.

## Recall by Version

| Variable | Version 1 | Version 2 | Version 2.1 |
|---|---|---|---|
| Gender | 74% | 93% (90–96%) | — |
| Race | 78% | 81% (70–95%) | 80% (73–87%) |

## Plaintiff Demographics

**Gender** (stated for 561 of 640, 88%)

| Gender | Plaintiffs | Share of stated |
|---|---|---|
| Male | 409 | 73% |
| Female | 150 | 27% |
| Nonbinary | 2 | <1% |
| Not stated | 79 | — |

**Race** (stated for 352 of 640, 55%)

| Race | Plaintiffs | Share of stated |
|---|---|---|
| Black | 270 | 76.7% |
| Hispanic/Latino | 34 | 9.7% |
| White | 15 | 4.3% |
| Person of color (unspecified) | 14 | 4.0% |
| Multiple | 5 | 1.4% |
| Asian | 4 | 1.1% |
| Middle Eastern/North African | 3 | 0.9% |
| Native American | 3 | 0.9% |
| Pacific Islander | 2 | 0.6% |
| Other (as stated) | 2 | 0.6% |
| Not stated | 288 | — |

**Age** (stated for 99 of 640, 15%): median 34; range 1–83; 11 under 18, 11 aged 65 or over.

Stated shares are lower bounds: about 7% of stated genders and 20% of stated races are estimated to be missed. The data describe plaintiffs in complaints alleging racial profiling, not everyone stopped by police.
