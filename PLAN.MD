### Meeting 2: Train/eval/test split preparation

> **Note:** Use predefined random seeds, last access date information, version numbers, etc., wherever possible for maximum reproducibility.

**Protein function prediction**

1. **Download GAF UniProt-centric archives** from [Gene Ontology Downloads](https://geneontology.org/docs/download-go-annotations/downloads/):
   * *Candida*
   * *Candida parapsilosis*
   * *Baker's yeast*
2. **Download UniProt ID-to-sequences mappings** for the above model organisms using [UniProt](https://www.uniprot.org/):
   * Type the model organism name (e.g., `rat`) in the search window.
   * Filter by **Reviewed (Swiss-Prot)** in the left navigation panel.
   * Customize columns to include: `Entry`, `Gene Ontology IDs`, and `Sequence`.
   * Download the outcomes as **TSV** files.
3. **Merge the resulting files (steps 1–2)** into one master file:
   * Add a column with taxonomic information (e.g., *Human*, *Candida*, etc.).
   * Compare GO IDs per protein coming from GAF vs. UniProt to check for differences, and keep the **UniProt** annotations as ground truth.
   * Filter for high-quality Evidence Codes only: `'EXP'`, `'IDA'`, `'IPI'`, `'IMP'`, `'IGI'`, `'IEP'`, `'HTP'`, `'HDA'`, `'HMP'`, `'HGI'`, `'HEP'`, `'TAS'`, `'IC'`.
   * Create a **FASTA** file with the resulting well-annotated sequences.
4. **Create the train/eval/test split:**
   * Run `mmseqs easy-cluster` on the generated FASTA file with parameters: `min-seq-id = 0.5`, `coverage = 0.8`, and `cov-mode = 0`.
   * Assign whole clusters to `train`, `eval`, and `test` sets at an **8:1:1** ratio *(note: this is not straightforward and requires an iterative procedure)*.

- [ ] GAF archives and UniProt TSV files downloaded (with versions/access dates logged)
- [ ] Master dataset merged, filtered by high-quality evidence codes, and exported to FASTA
- [ ] Clustering completed via `mmseqs easy-cluster` and 8:1:1 cluster split generated

---

### Meeting 3: End-to-end skeleton

> **Note:** Keep the implementation as modular and customizable as possible. Once a real architecture is integrated, the initial logic may change substantially. The goal is to build a working end-to-end pipeline, not a meaningful prediction model.

**Protein function prediction**

1. **Create a working predictor template** that:
   * Receives a multi-protein FASTA file as input, parsed using `scikit-bio` (`skbio.io.read()`).
   * Outputs three prediction vectors per protein—one per Gene Ontology sub-ontology (**BP**, **MF**, **CC**)—matching the length of the corresponding GO-term vocabulary from Meeting 2 *(sizes may be hardcoded for now)*.
   * Saves predictions to a CSV file with the format `ProteinID,GOterm,Score`, where each row represents the prediction score for a single protein–GO-term pair.
2. **Implement a dummy PyTorch model** for each ontology that accepts batches of encoded sequences and outputs random prediction scores:
   * **Input shape:** `(B, N, D)`, where:
     * `B` – batch size (e.g., `32`)
     * `N` – amino-acid vocabulary size (e.g., `22`: 20 standard amino acids, `X` for unknown/non-standard residues, and `PAD` for padding)
     * `D` – maximum sequence length (e.g., `1024`; shorter sequences are padded, longer sequences are truncated)
   * **Masking:** Implement a mask identifying valid residues versus padding positions (the dummy model may ignore sequence content, but the interface must support masking).
   * **Output shape:** `(B, P)`, where `P` is the number of GO terms for the corresponding ontology.

- [ ] FASTA input parser implemented with `skbio.io.read()` and sequence encoding/padding/masking pipeline ready
- [ ] Modular dummy PyTorch models implemented for BP, MF, and CC with `(B, N, D)` input and `(B, P)` output
- [ ] End-to-end inference exporting predictions in `ProteinID,GOterm,Score` CSV format

---

### Meeting 4: Evaluation framework

> **Note:** The goal is to implement reliable, community-accepted evaluation metrics. Some metrics are time-consuming, which is acceptable since they run during testing only. Random predictions suffice to verify the pipeline, though testing with real predictions is also recommended.

**Protein function prediction**

1. **Install and configure** `CAFA-evaluator`.
2. **Prepare the required input files:**
   * GO ontology (`OBO` file)
   * Ground-truth protein annotations generated in Meeting 2
   * Predictions generated in Meeting 3
3. **Run the evaluation** separately for **BP**, **MF**, and **CC**.
4. **Extract and save the evaluation results**, including:
   * Precision-recall curves
   * $F_{\max}$
   * $S_{\min}$

- [x] `CAFA-evaluator` installed and configured
- [x] Input files prepared (OBO ontology, Meeting 2 ground truth, Meeting 3 predictions)
- [x] Evaluation executed across BP, MF, and CC, with Precision-Recall curves, $F_{\max}$, and $S_{\min}$ saved