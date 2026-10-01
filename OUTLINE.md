# Standalone paper — outline

**Target:** neuroradiology / neurosurgery methods venue — *Radiology: Artificial Intelligence* (primary), alt. *AJNR*, *J Neurosurg*, *npj Digital Medicine*.
**Type:** Original research / methods. Full length (no 1,200-word cap). Reporting per CLAIM / TRIPOD-AI.

---

## Message
A neuroimaging foundation model's **diagnostic head generalizes to an external institution**, but clinical performance is **governed by how it is deployed** — input series/recon selection materially changes perception, and a reproducible offline pipeline is needed to run it at scale on routine PACS data. We provide external validation, a quantified deployment lesson, interpretable anatomical anchoring, and an open pipeline.

## Summary paragraph
We externally validated NeuroVFM's calibrated diagnostic head on N=343 institutional head CTs paired with radiologist reports (mean AUROC ≈ 0.82 vs in-domain 0.925), characterized a **perception failure that is an input-selection artifact** (fracture AUROC 0.14 → recovered with bone-kernel reconstruction), cross-validated predictions against automated anatomical volumetry, and release a HIPAA-safe, resumable inference pipeline.

## Structure
1. **Introduction** — foundation models in neuroimaging; external validation + deployment gaps; our contributions.
2. **Methods**
   - Cohort & provenance (institutional head-CT delivery; the RADAR token-truncation data-integrity note; selection to 490 primary axial-brain CTs; 343 report-paired).
   - Three aligned layers: Path-A 82-dx; TotalSegmentator 16 volumes; report-derived 17 findings (negation-mined labels, M4).
   - Pipeline (M1 offline/at-scale LSF; M2 multi-slice DICOM→NIfTI correctness).
   - Multi-series/bone-kernel routing (M3).
   - Crosswalk (M5); statistics (AUROC, bootstrap CIs, sens/spec at operating point).
3. **Results**
   - External validation per-finding (Fig 1).
   - Input-conditioned perception: original vs improved (Fig 2).
   - Anatomical concordance (Fig 3).
   - Pipeline throughput / reproducibility (Fig 4 / table).
4. **Discussion** — generalization + domain-shift gap; deployment-as-safety; limitations (single-site, N, label sparsity, fracture label scope); future (RADAR redelivery ~3× cohort; prospective).
5. **Data/Code availability** — this repo (de-identified tables + code); weights via upstream; raw imaging restricted (HIPAA).

## Figures
- **Figure 1 — External validation.** Per-finding AUROC forest/bar (reuse `dxct_full/fig1_auroc.png`, restyled), N annotations, in-domain 0.925 reference; inset mean ≈ 0.82. Bootstrap 95% CIs.
- **Figure 2 — Deployment governs perception (original vs improved).** (a) paired bars original (soft-tissue) vs improved (multi-series) AUROC per finding — fracture the headline mover; (b) example images: same fracture, soft-tissue window (occult) vs bone-kernel (overt) with P(fracture) 0.01→0.80; (c) scatter of per-study P change.
- **Figure 3 — Interpretable anchoring (Path-A × anatomy).** Path-A hydrocephalus probability vs TotalSegmentator ventricle volume (+ atrophy↔brain/CSF, mass_effect↔midline). Shows calibrated probs track measured anatomy → convergent validity.
- **Figure 4 — Deployable offline pipeline** schematic: DICOM→NIfTI (JPEG2000-safe, series pick) → encoder (3 windows) → dx head; resumable LSF arrays; throughput table (studies/GPU-hr), air-gapped/HIPAA notes.

## Extended/Supplementary
- Cohort provenance + the truncation audit (reconciliation table).
- Full 82-label cohort summary; per-finding operating points.
- Relabeling (M4) negation rules + counts; crosswalk (M5) table.
- Excluded-study accounting (11 unusable series fail both pipelines).

## Analysis status
- [x] External validation table + Fig 1 (original).
- [~] Improved multi-series run (jobs 99546869/99546870 in flight) → Fig 2a + delta table.
- [x] Fracture example (soft vs bone-kernel) → Fig 2b/c (6-case result in hand; pull render images).
- [ ] Anatomy concordance (M7) → Fig 3 (both tables exist; zero-GPU join).
- [x] Pipeline (M1/M2) documented → Fig 4.

## Repo layout (this directory)
- `manuscript.md`, `OUTLINE.md`, `figures/`, `data_deid/` (de-identified tables: dx_predictions_deid, eval_vs_reports, ct_brain_volumes_deid, report_labels_v2), `analysis/` (eval/relabel/concordance/make_fig scripts). **Never** `deid_map.csv` or token→PHI.
