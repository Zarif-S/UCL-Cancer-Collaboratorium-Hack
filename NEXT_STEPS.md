# Next Steps: TC-AWARE Pipeline

## Current State

The hackathon delivered a **proof-of-concept for Step 3 (structural modelling) only**.

The notebook:
- Takes 6 known TCR-pMHC binding pairs from IEDB
- Enriches them with MHC heavy chain + β2-microglobulin sequences from UniProt
- Generates AlphaFold 3 JSON input files (and Boltz-2 YAML files)
- AF3 was run externally and returned ipTM=0.66, pTM=0.7 — confirming the 5-chain complex folds plausibly

Everything else in the proposed pipeline is unbuilt.

---

## Pipeline Status

| Step | Component | Status | Notes |
|---|---|---|---|
| 1 | Neoantigen identification from tumour genome | Not built | No variant calling, no NGS/WES pipeline |
| 2 | MHC binding prediction | Not built | No NetMHCpan, no custom transformer |
| 3 | TCR-pMHC structural modelling | PoC done | 6 IEDB complexes run through AF3 |
| 3 | Exclusion of exhausted T-cell targets | Not built | The core innovation — no implementation |
| 4 | Ranking & scoring pipeline | Not built | None of the 7 metrics implemented |
| — | Patient-specific input pipeline | Not built | No support for real patient data as input |

---

## Known Bugs

- **Boltz-2 cell** (`hack-1.ipynb`): reads from `AFserver_inputs/` but files are saved to `Human_AF3_inputs/`. Fix the directory path before running.

---

## What to Build Next (in order)

### 1. Neoantigen Identification (Step 1)
The entry point of the pipeline. Without this, there are no candidate peptides to score.

- **Tool:** pVACseq (wraps variant calling → peptide extraction → HLA typing in one workflow)
- **Input:** Paired tumour/normal WES or WGS BAM/VCF files
- **Test data:** Use a TCGA sample (GDC portal) — somatic MAF files are publicly available
- **Output:** List of candidate mutated 9-mer peptides specific to the tumour

### 2. MHC Binding Prediction (Step 2)
Filter the neoantigen candidates down to peptides that will actually be presented on the cell surface.

- **Tools:** NetMHCpan 4.1 or MHCflurry 2.0 (both free, well-benchmarked)
- **Input:** Candidate peptides from Step 1 + patient HLA type
- **Output:** Peptides with predicted IC50 binding affinity; keep those below ~500 nM
- **Note:** No need to train a custom transformer — existing tools are state of the art

### 3. Exclusion of Exhausted T-Cell Targets (Step 3 — core innovation)
This is the key differentiator of the TC-AWARE approach. Remove peptides the patient's immune system has already tried and failed to attack.

- **Input A:** MHC-binding peptides from Step 2
- **Input B:** Patient TCR repertoire sequences (blood sample; public test data available from Adaptive Biotechnologies immuneACCESS)
- **TCR-peptide binding predictor:** NetTCR-2.0, ERGO-II, or ImRex
- **Logic:** For each candidate peptide, predict which TCRs in the patient's repertoire bind it. Flag and remove peptides already targeted by known/expanded T-cell clones (indicating prior immune exposure and likely exhaustion).
- **Output:** Filtered list of peptides the patient's immune system has *not* yet engaged

### 4. Ranking & Scoring Pipeline (Step 4)
Score the remaining candidates across multiple axes and output a ranked vaccine target list.

The 7 metrics from the proposal:

| Metric | How to compute |
|---|---|
| MHC-peptide binding affinity | NetMHCpan IC50 (from Step 2) |
| TCR-peptide binding affinity | NetTCR / ERGO score (from Step 3) |
| Surface presentation likelihood | NetChop (proteasomal cleavage) + TAP transport score |
| Peptide concentration | Requires Ribo-seq or MS data — hardest to get |
| Conservation across cancer subtypes | Cross-reference against TCGA mutation frequency |
| Clonality (phylogenetic position) | Requires multi-region sequencing or ctDNA; use VAF as proxy |
| Cross-reactivity / safety | BLAST against human proteome; flag close matches |

- **Output:** Top-k peptides with per-metric scores and composite rank

### 5. End-to-End Patient Pipeline
Wire steps 1–4 into a single workflow that accepts:
- Tumour VCF/MAF (from NGS)
- Patient HLA type (from genotyping)
- Patient TCR repertoire (from blood sequencing)

And outputs a ranked, annotated list of personalised vaccine candidates.

---

## Data Availability

There are two distinct categories of data this pipeline needs: **development/training data** (used to build and validate the system) and **patient-specific clinical data** (the actual per-patient inputs in a real deployment). These have very different accessibility profiles.

### Development & Training Data

| Data | Source | Availability | Notes |
|---|---|---|---|
| TCR-pMHC binding pairs | IEDB | Open access | Current dataset uses 6 entries — relax query filters to get more |
| HLA-peptide binding affinities | IEDB | Open access | Large curated dataset |
| MHC heavy chain sequences | UniProt REST API | Open access | Already used in the PoC |
| Tumour somatic mutations (research cohorts) | TCGA via GDC portal | Controlled access | Processed MAF files are open; raw WES/WGS requires dbGaP application |
| TCR repertoire data (research cohorts) | Adaptive Biotechnologies immuneACCESS | Mixed | Some datasets open; others require registration or MTA |
| HLA population frequencies | Allele Frequency Net Database | Open access | |
| TCR-pMHC crystal structures | RCSB PDB | Open access | For validating AF3 outputs |
| Ribo-seq / MS peptidome (cell lines) | PRIDE, GEO, published studies | Open access | For surface presentation / peptide concentration metrics |

### Patient-Specific Clinical Data

This is what the pipeline would consume in a real clinical deployment. **None of this is publicly available** — it is protected personal health data governed by GDPR (UK/EU) and equivalent regulations.

| Data | How it's obtained | Availability | Barrier |
|---|---|---|---|
| Patient tumour genome | Tumour biopsy + NGS/WES sequencing | Not public | Requires patient consent, NHS/hospital ethics approval, clinical governance |
| Patient germline genome | Blood or saliva sample + sequencing | Not public | Same as above — needed to distinguish somatic from germline variants |
| Patient HLA genotype | Derived from germline sequencing or dedicated typing | Not public | Clinical data, tied to patient identity |
| Patient TCR repertoire | Blood draw + bulk/single-cell TCR-seq | Not public | Clinical sample, consent required; also expensive (~£500–2000 per patient) |

### Workarounds for Development Without Patient Data

Since real patient data is inaccessible for development purposes, these are practical alternatives:

| Need | Alternative |
|---|---|
| Tumour genome | TCGA open-access MAF files (de-identified, consented research cohorts) |
| TCR repertoire | Public immuneACCESS datasets from Adaptive Biotechnologies (healthy donors and some cancer patients) |
| HLA type | Infer from TCGA WES data using tools like OptiType, or use published HLA-typed cohorts |
| Synthetic patient | Simulate a patient profile using published neoantigen case studies (e.g. Parkhurst et al. 2019, Nature Medicine) |

---

## Startup Blockers

Beyond data access, there are commercial and regulatory constraints that would affect building this as a startup.

### Tool Licensing (Commercial Use Restrictions)

| Tool | Used for | Academic | Commercial | Alternative |
|---|---|---|---|---|
| AlphaFold 3 | TCR-pMHC structural prediction | Free (research only) | Requires Google DeepMind license | **Boltz-2** (MIT licensed) — already in the notebook |
| NetMHCpan | MHC binding prediction | Free | Requires DTU commercial license | **MHCflurry 2.0** (MIT licensed) |
| NetTCR | TCR-peptide binding prediction | Free | Requires DTU commercial license | **ERGO-II** (open source) |
| NetChop | Proteasomal cleavage prediction | Free | Requires DTU commercial license | Limited alternatives — may need to license |
| pVACseq | Neoantigen identification pipeline | Free (BSD) | Free (BSD) | No issue |

### Clinical & Regulatory Blockers

These cannot be fully worked around — they are hard requirements for any clinical deployment.

| Blocker | Workaround? | Notes |
|---|---|---|
| Real patient data for development | Yes — use TCGA + public cohorts | Sufficient to build and validate computationally |
| Real patient data for clinical claims | No | Requires ethics approval (IRB/NHS REC) and a hospital partnership |
| Regulatory approval (SaMD) | No — but manageable | Output informs clinical decisions → classified as Software as a Medical Device under MHRA (UK), MDR (EU), or FDA. Plan for 2–3 years minimum |
| Clinical validation study | No | Must run a study on real patients before any regulatory submission or clinical claim |
| Vaccine manufacturing (GMP) | Out of scope | Partner with a CDMO — not a software problem |

### Bottom Line for a Startup

Nothing blocks you from **building the full system**. The development path using public data (TCGA, IEDB, immuneACCESS) is clear, and open-source alternatives exist for every commercially-restricted tool.

The hard wall is at the point of **clinical use**:
1. You need a hospital/oncology centre partner to run a validation study with real patients
2. You need ethics approval before accessing any patient data
3. You need SaMD regulatory clearance before clinical deployment

The practical startup strategy is: build a compelling computational demo on public data → use that to secure an academic hospital partnership → run clinical validation through them → use that data for regulatory submission.

---

## Funding Roadmap

VC funding is not the first step — it comes after clinical validation data exists. The realistic path is **grants first, hospital partnership second, VC third**.

### Phase-by-Phase Plan

| Phase | Timeline | What to do | Funding |
|---|---|---|---|
| 1 | Now – 6 months | Build full computational pipeline on public data | Self-funded / UCL resources / hackathon prize |
| 2 | 6 – 12 months | Approach UCL/UCLH oncology group for research partnership (MOU, not commercial) | Apply for Innovate UK or Wellcome grant (£50–250k) |
| 3 | 1 – 2 years | Run validation study through hospital partnership, generate clinical data | NIHR i4i or Wellcome (~£500k–1M) |
| 4 | 2 – 3 years | Regulatory pre-submission (SaMD), seed fundraise | Seed VC round (£1–5M) — now you have clinical data to show |

### Why the Hospital Partnership Doesn't Cost Money (Initially)

NHS hospitals and academic medical centres actively want research collaborations — it generates publications, grant co-applications, and prestige. The initial agreement is a **research MOU**, not a commercial contract. You provide the software; they provide clinical access and expertise. No money changes hands at this stage.

### Early-Stage Grant Sources (Non-Dilutive)

These fund pre-clinical and early translational work without taking equity:

| Funder | Grant | Amount | Notes |
|---|---|---|---|
| Innovate UK | Health & Life Sciences grants | £50k–£500k | Funds health tech at early stage |
| NIHR | i4i (Invention for Innovation) | £500k–£1M | Specifically for NHS-applicable health tech |
| Wellcome Trust | Leap / Innovator Awards | £100k–£500k | Funds ambitious biomedical ideas |
| UKRI | Future Leaders Fellowship | £400k–£1.5M | For early-career researchers on the team |

### The UCL Shortcut

Given the project originated at the UCL CCC Hackathon, there are direct routes in:

- **UCL Technology Fund** — translational funding + IP support for UCL spinouts
- **UCL Business (UCLB)** — helps commercialise UCL research, provides introductions to clinical collaborators at UCLH
- **UCL Hatchery / Entrepreneur in Residence programme** — early-stage startup support

UCLH (University College London Hospitals) is one of the leading cancer centres in the UK and is directly linked to UCL — the oncology groups there are the natural first clinical partner.
