# Project Notes — Single-Cell Drug Perturbation Mini-Study

Detailed research notes for the five-task workflow. Each task is one notebook and answers one stage of the question: **how do drugs perturb single-cell transcriptional state, and can we predict it?**

---

## Task 1 — Data Exploration

- **Question**: What is this dataset?
- **Methods**: Load `srivatsan_2020_sciplex2` via pertpy; inspect AnnData structure (`.X`, `.obs`, `.var`, `.layers`).
- **Data**: A549 lung adenocarcinoma cells; 4 drugs (dexamethasone, nutlin-3a, BMS-345541, vorinostat/SAHA) + vehicle control; 8 doses (0–100 µM); 6 wells per drug-dose; ~24k cells × 58k genes. Raw UMI counts in `.X`.
- **Key note**: `srivatsan_2020_sciplex2` is a *subset* (A549 + 4 drugs), not the full 188-drug screen (`srivatsan_2020_sciplex3`).

## Task 2 — QC & Preprocessing

- **Question**: Which cells/genes are reliable?
- **Methods**: Compute QC metrics (counts, genes, % mitochondrial); data-driven filtering; `normalize_total` + `log1p`; HVG selection; raw counts kept in `layers["counts"]`.
- **Key findings**: The provider already filtered cells < 1000 UMIs. Mitochondrial % is elevated overall (median 7%) because this is a cancer cell line + sci-RNA-seq, so a template `mt < 5%` cutoff would remove 74% of cells — a data-driven `mt < 20%` was used. **The un-hashed control is sequenced ~3× shallower than treated cells** (a batch confound flagged for downstream).

## Task 3 — Drug- and Dose-dependent Cell-state Landscape

- **Question**: Where does the cell state move?
- **Methods**: PCA → neighborhood graph → UMAP → Leiden clustering; PCA-space centroids to quantify perturbation **magnitude** and **direction**; replicate reproducibility.
- **Key findings**: Perturbation magnitude ranks SAHA (7.5) > Nutlin (4.0) ≈ BMS > Dex (2.3). Nutlin and SAHA push cells in different directions (cosine ≈ 0.35). Drug signal exceeds replicate noise ~6×.

## Task 4 — Differential Expression & Pathway Interpretation

- **Question**: Why does the state move?
- **Methods**: **Replicate-aware pseudobulk** (aggregate raw counts per drug×dose×well); `pydeseq2` count-based DEG (6 vs 6 replicates); MSigDB Hallmark enrichment via `gseapy`; dose-dependent gene trajectories; replicate consistency.
- **Key findings**: Nutlin-3a top pathway = **p53 Pathway (adjusted p = 1e-46)** — consistent with MDM2 inhibition. Vorinostat = broad response (7,405 DEGs vs 2,824), top pathways are stress/UV/cholesterol, not p53-specific. Only ~130 shared upregulated genes → mostly drug-specific programs.

## Task 5 — Predictive Perturbation Modeling

- **Question**: Can we predict an unobserved drug-dose state?
- **Methods**: Held-out dose prediction (interpolation: hide 10 µM; extrapolation: hide 100 µM); baselines (control, nearest-dose, log-dose linear interpolation) + ridge regression in PCA space; evaluated PCA error, magnitude/direction recovery, gene correlation, DEG recovery (Jaccard), pathway recovery.
- **Key findings**: Local interpolation/nearest-dose (PCA error ≈ 1.0) beat global ridge (≈ 3.2) → locally smooth, globally nonlinear trajectories. All-gene correlation is ~1.0 for every model (unchanged genes dominate), but response-gene correlation separates them. **The model recovers the average state but not the drug-specific program** (ridge predicted generic stress pathways, not p53).

## Central narrative

```
Task 3  Where did the cell state move?   → magnitude + direction
Task 4  Why did it move?                 → genes + pathways
Task 5  Can we predict the move?         → held-out dose prediction
```

## Gaps toward a Virtual Cell

Held-out dose prediction is interpolation/extrapolation within one drug and one cell line. A true Virtual Cell requires generalizing to **unseen drugs**, **unseen cellular contexts**, **time**, **genetic background**, and **multimodal state** — and recovering drug-specific biological programs, not just average expression.
