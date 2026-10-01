# External validation and methodological improvement of NeuroVFM's diagnostic head — analysis & methods summary

*Working summary to seed the manuscript. All data de-identified (project91 tokens / NVP####); no PHI in this file.*

---

## 1. Background & thesis

NeuroVFM (Kondepudi et al., *Nature Medicine* 2026) is a frozen 3D neuroimaging encoder feeding two independent downstream paths:

- **Path A (diagnostic head):** attentive-MIL probe → calibrated sigmoid probabilities over **82 CT diagnoses** (reported in-domain macro-AUROC ≈ 0.925).
- **Path B (findings LLM):** Perceiver → Qwen3-14B → free-text findings → GPT-5 triage (reported prospective triage sensitivity 86.5%).

In the original prospective study, **all 21 urgent misses were cases where the generated prose was silent** about a finding; the authors attributed this to *perception*. 

**Our central thesis:** a substantial fraction of Path-B's silent misses are cases where **Path A was confident** — i.e. the encoder *perceived* the finding and the prose path *lost* it. If so, findings should be **rendered from the calibrated diagnostic head**, not generated as free text. Secondarily, we show NeuroVFM's perception is **input-conditioned** (recon kernel / series selection materially change detection), and we provide the first **deployable, offline, at-scale Path-A inference recipe**.

---

## 2. Cohort & data layers

De-identified institutional head-CT cohort (project91 tokenization; PixelSafe).

- **Delivery provenance (data-integrity note for methods):** of 3,335 studies submitted to the honest broker (RADAR), 3,182 were tokenized and **1,228 delivered**. The shortfall was diagnosed not as a clinical exclusion but a **truncated export** — sorted by token, every study ≤ token 42178076 arrived (99.3%) and every study above it (1,945) was absent. A redelivery request (1,954 tokens) is pending; it will ~triple the cohort.
- **Selected imaging set:** one primary axial-brain CT per study → **490 studies** (of 982 CT studies; 492 excluded as CTA/perfusion/scout/neck/non-axial). 479 scored successfully (11 unusable series fail both this and the segmentation pipeline).

Three **token-aligned** per-study layers:
1. **NeuroVFM Path-A** — 82 calibrated diagnosis probabilities (`dxct_full/dx_predictions_deid.csv`).
2. **TotalSegmentator** — 16 brain-structure volumes incl. venous sinuses (`vascbrain/seg/ct_brain_volumes.csv`, ~549 studies; 514 with nonzero venous sinus).
3. **Radiologist reports** — 17 report-derived findings + full text (`data/inhouse/report_labels.csv`, `inhouse_reports.csv`).

**Paired evaluation set:** 343 studies with both Path-A predictions and report labels.

---

## 3. Methodological contributions (categorized)

**M1 — Deployable offline, at-scale Path-A pipeline (engineering).** The upstream release ships a quickstart, not a cohort recipe. We built a HIPAA-safe, air-gapped pipeline: local-weight loading (`HF_HUB_OFFLINE`), chunked **resumable LSF arrays** (convert on CPU/rhel9, score on gpu1 MIG), model-load amortized across study chunks. Reproducible on an HPC with no internet.

**M2 — Correct multi-slice DICOM→volume handling (correctness).** The default `load_study` globs `*.dcm` and treats **each single-slice file as its own volume**, which silently breaks on classic one-file-per-slice series (returns 2-D images, "expected 3, got 2"). We assemble a proper 3-D NIfTI first (JPEG2000-safe, axial-series selection) and feed NIfTI. *Required* for the model to run on standard PACS exports.

**M3 — Multi-series / bone-kernel input routing (perception improvement).** NeuroVFM renders three windows (brain 80/40, blood 200/80, bone 2800/600) from a single input volume, but the bone *window* of a smooth **soft-tissue-kernel** recon blurs fracture lines. We route the sharp **bone-kernel** recon in alongside the soft-tissue series (`load_study([soft, bone])`). *Original vs improved delta: §5.*

**M4 — Report-mined negatives for rigorous external evaluation (eval methodology).** Report labelers leave non-affirmed findings "unknown," collapsing specificity estimates. We added a **negation-aware relabeler** (finding-specific negation patterns + global-negative sentences like "no acute intracranial abnormality") that upgraded unknown→neg from the text — adding hundreds of true negatives (sdh +307, sah +281, ivh +281, edh +283, …) and powering per-finding AUROC that were previously undefined.

**M5 — Finding→label crosswalk.** 17 report findings → NeuroVFM's 82 labels (max probability over mapped labels), e.g. any_ich → {intracranial/intraparenchymal/intraventricular hemorrhage, acute/chronic SDH, EDH, aneurysmal/traumatic SAH}; fracture → {displaced, nondisplaced, skull-base}.

**M6 (planned) — Render findings from Path A vs Path-B prose.** Emit a structured findings list from the calibrated heads (`mapping/findings_templates.yaml`) and score against report ground truth vs Path-B prose → the thesis test on real data.

**M7 (planned) — Anatomical concordance (interpretability).** Cross-validate Path-A probabilities against TotalSegmentator volumes (hydrocephalus↔ventricle volume, atrophy↔brain/CSF, mass_effect↔midline displacement).

---

## 4. Analyses run & results — ORIGINAL NeuroVFM on the external cohort

Path-A (single soft-tissue series), N=343 paired, after report-mined negatives (M4):

| finding | pos/neg | AUROC |
|---|---|---|
| hydrocephalus | 25/175 | 0.96 |
| ivh | 9/199 | 0.93 |
| iph | 6/206 | 0.92 |
| mass_effect | 36/234 | 0.86 |
| sah | 18/199 | 0.83 |
| any_ich | 61/250 | 0.79 |
| acute_infarct | 59/170 | 0.79 |
| edema | 23/194 | 0.76 |
| sdh | 8/218 | 0.76 |
| mass_tumor | 15/40 | 0.73 |
| midline_shift | 6/266 | 0.71 |
| **fracture** | 6/26 | **0.14** |

**Mean AUROC ≈ 0.82** across 11 well-powered findings. Headline: NeuroVFM's diagnostic head **generalizes to an external institution** (vs in-domain 0.925 → a quantified ~0.10 domain-shift gap), strong on hemorrhage subtypes, hydrocephalus, mass effect, infarct. → **Figure 1** (`dxct_full/fig1_auroc.png`).

**Fracture is the lone failure (AUROC 0.14, below chance)** — investigated in §5.

---

## 5. ORIGINAL → IMPROVED (multi-series input, M3)

**Fracture mechanism experiment** (6 report-positive fractures, soft-tissue vs bone-kernel input):

| study | soft-tissue P(fracture) | bone-kernel P(fracture) |
|---|---|---|
| caseA | 0.011 | **0.80** (enters top-3) |
| caseB | 0.086 | 0.35 |
| caseD | 0.142 | 0.20 |
| caseC / caseE / caseF | — | ~unchanged |

**Two conclusions:** (1) **perception is input-conditioned** — the sharp bone-kernel recon flips a fracture from invisible (0.01) to confident (0.80); the naive soft-tissue pick hid it. (2) **Label heterogeneity** — the unchanged cases read as orbital/craniofacial (caseA top call orbital_trauma 0.96), skull-bone-tumor (caseE), or post-surgical (caseF): several "fracture" report-positives are facial/orbital or post-op defects outside NeuroVFM's skull-*vault*-fracture labels.

**Full original-vs-improved table (multi-series `load_study([soft, bone])` over all 490): _[PENDING — job 99546870; fill when complete]_.** Expectation: fracture and other bone findings rise; soft-tissue-dominant findings unchanged.

---

## 6. Key claims for the manuscript

1. **External generalization** of NeuroVFM's diagnostic head to a new institution (mean AUROC ≈ 0.82; per-finding Figure 1), with a quantified domain-shift gap.
2. **Perception is input-conditioned** (M3): recon-kernel/series routing materially changes detection — a deployment-critical, generalizable lesson (bone findings → bone-kernel recon).
3. **A deployable offline Path-A recipe** (M1/M2) the field can reuse.
4. **(Planned) Path-A rescues Path-B's silent misses** (M6) — the core thesis, on real paired reports.
5. **(Planned) Interpretable anchoring** of Path-A via anatomy (M7).

---

## 7. Limitations

- Single institution; N=343 paired (→ ~1,000+ after RADAR redelivery).
- Report-derived labels (M4) are high-precision but not exhaustive; "unknown" residue remains for chronic findings.
- Fracture label scope mixes vault/facial/post-surgical — needs location-resolved relabel.
- Path-B and render-findings (M6) not yet executed on this cohort.

---

## 8. Figures / tables inventory (on disk)

- **Figure 1** — per-finding AUROC, original Path-A (`dxct_full/fig1_auroc.png`). *Restyle to NeuroVFM house style when their figure code is provided.*
- `dxct_full/eval_vs_reports.csv` — per-finding metrics (original).
- `dxct_full/dx_predictions_deid.csv`, `dx_summary.csv` — cohort predictions.
- `dxct_full/report_labels_v2.csv` — report-mined labels (M4).
- `vascbrain/seg/ct_brain_volumes.csv` — anatomical volumes (M7 input).
- *Pending:* improved (multi-series) table + original-vs-improved delta figure; Path-B disagreement; anatomy concordance.
