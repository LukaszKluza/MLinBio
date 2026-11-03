# Protein Function Prediction (PFP)

> Create an end-to-end framework that predicts protein function — in the form of **Gene Ontology (GO) term probabilities** — from an **amino acid protein sequence**.

## Authors

**Team 2**

- Szymon Janik
- Łukasz Kluza
- Mateusz Sacha

---

## Meeting 1 — Train / Eval / Test Split Preparation

> NOTE: Use a **predefined random seed**, **last access date**, **version** etc. wherever possible for maximum reproducibility.

### Organisms

| # | Organism |
|---|----------|
| 1 | *Candida* |
| 2 | *Candida parapsilosis* |
| 3 | *Baker's yeast* (*Saccharomyces cerevisiae*) |

### 1. Download GO annotations (GAF)

Download the **UniProt-centric GAF archives** for our organisms from:
<https://geneontology.org/docs/download-go-annotations/downloads/>

### 2. Download UniProt ID → sequence mappings

For each model organism, using <https://www.uniprot.org/>:

1. Type the model organism name (e.g. *rat*) in the search window.
2. Keep only **reviewed** proteins (**Swiss-Prot**) — left navigation panel.
3. Customize columns to: **Entry**, **Gene Ontology IDs**, **Sequence**.
4. Download the results as **TSV** files.

### 3. Merge into one master file

Merge the outputs of steps 1–2 into a single master file:

1. Add a column with **taxonomic information** (*Human*, *Candida*, …).
2. Keep only **high-quality evidence codes**:
   | Category | Codes |
   |----------|-------|
   | Experimental | `EXP`, `IDA`, `IPI`, `IMP`, `IGI`, `IEP` |
   | High-throughput | `HTP`, `HDA`, `HMP`, `HGI`, `HEP` |
   | Author / Curator statement | `TAS`, `IC` |

3. **Compare GO IDs per protein** from the GAF vs. from UniProt.
   - Are there any differences?
   - Keep the **UniProt** annotations as the **ground truth**.
4. Create a **FASTA** file with the resulting well-annotated sequences.

### 4. Train / eval / test split

1. Cluster the FASTA file with MMseqs2:

   ```bash
   mmseqs easy-cluster input.fasta clusterRes tmp \
       --min-seq-id 0.5 \
       -c 0.8 \
       --cov-mode 0
   ```

2. Assign **whole clusters** to **train**, **eval** and **test** sets at a ratio of **8 : 1 : 1**.

---

## Meeting 2 — End-to-end Skeleton

> NOTE: Keep the implementation as **modular and customizable** as possible. Once we integrate a real architecture, the initial logic may change substantially. The goal of this meeting is to implement a **working end-to-end pipeline**, not a meaningful prediction model.

Create a working **predictor template** that:

1. Receives a **multi-protein FASTA file** as input, parsed using **scikit-bio** (`skbio.io.read()`).
2. Outputs **three prediction vectors per protein**, one per Gene Ontology (**BP**, **MF**, **CC**).
   - Each vector must have the same length as the corresponding **GO-term vocabulary** prepared in Meeting 2 (these sizes may be hardcoded for now).
3. Saves predictions to a **CSV** file with the format: `ProteinID,GOterm,Score`
   - Each row represents the prediction score for one **protein–GO-term pair**.
4. Implements a **dummy PyTorch model** for each ontology that accepts batches of encoded sequences and outputs **random prediction scores**.

### Model input / output

| Tensor | Shape | Description |
|--------|-------|-------------|
| Input | `(B, N, D)` | Batch of encoded sequences |
| Output | `(B, P)` | Prediction scores per GO term |

Where:

| Symbol | Meaning | Example |
|--------|---------|---------|
| `B` | Batch size | 32 |
| `N` | Amino-acid vocabulary size (20 standard amino acids, `X` for unknown / non-standard residues, `PAD` for padding) | 22 |
| `D` | Maximum sequence length | 1024 |
| `P` | Number of GO terms for the corresponding ontology | — |

- Shorter sequences are **padded**; longer sequences are **truncated**.
- Implement a **mask** identifying **valid residues** versus **padding** positions. The dummy model may ignore sequence content, but the interface should support masking.
