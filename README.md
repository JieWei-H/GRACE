# GRACE: GRaph Autoregressive Complex gEneration

GRACE reformulates protein complex prediction from protein-protein interaction (PPI) networks as a **graph-conditioned autoregressive generation** problem. It fuses ESM-2 sequence semantics with PPI topology via a graph-structure embedding module, and decodes complex-member sequences under functional calibration provided by a topology-aware similarity bias.

On the human PPI network, GRACE outperforms nine representative methods across Precision, Recall, F-measure, Global MMR, and Mean MMR.

---

## Repository Structure

The codebase is organized following the Methods sections of the manuscript:

| Directory                   | Corresponding manuscript section      | Contents                                                                                                               |
| --------------------------- | ------------------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| `01_data_preparation/`      | Data sources and network construction | Base subgraph generation (`main/`: pipeline version; `legacy/`: early version), train/test splitting, PPI data loading |
| `02_graph_embedding/`       | Graph-structure embedding             | GAT, attention pooling, contrastive training (`Train-Cosine.py`), tokenizer                                            |
| `03_corpus_construction/`   | Training corpus construction          | BFS-serialized pretraining and SFT corpus builders (`GraphTransform_BFS_Pretrain.py`, `GraphTransform_BFS_SFT-1.py`)   |
| `04_pretraining/`           | Autoregressive pre-training           | `Pretrain_Cosine.py`, LM backbone (`ModelLM.py`), dataset loader                                                       |
| `05_fine_tuning/`           | Supervised fine-tuning                | `SFT_Cosine.py`, `SFT_None.py`                                                                                         |
| `06_functional_annotation/` | Functional annotation integration     | Label propagation (`Annotation.py`), similarity matrix construction                                                    |
| `07_inference_evaluation/`  | Inference and evaluation              | `Eval_Model-100.py` (main inference), `Eval.py`, `Eval-Score.py` (Precision/Recall/F/MMR)                              |
| `08_enrichment/`            | Functional enrichment                 | Enrichment analysis scripts (GO/KEGG, etc.)                                                                            |
| `09_heldout_evaluation/`    | Held-out evaluation                   | Independent held-out pipeline (its own data split under `dataset/`)                                                    |
| `data/`                     | —                                     | Inference/evaluation input files: PPI graph, complex gold standard, token mapping, held-out splits (see below)         |
| `artifacts/`                | —                                     | Pre-trained node embeddings and functional similarity matrices (see below)                                             |



---

## Requirements

- Python 3.10
- PyTorch ≥ 2.0 (CUDA 11.8+)
- transformers ≥ 4.35
- numpy, pandas, scipy, scikit-learn, networkx, tqdm, matplotlib

Optional (used by feature modules): `dgl`, `cuml`, `cupy`, `numba`, `wandb`, `gseapy`/`gprofiler` (enrichment).

See `requirements.txt`.

---

## Data

External data are **not** redistributed in this repository (licensing constraints). They can be downloaded from the original sources:

| Data                     | Source                                                        | Use                                                   |
| ------------------------ | ------------------------------------------------------------- | ----------------------------------------------------- |
| Human PPI network        | STRING v12.0 (<https://string-db.org>)                        | Network construction                                  |
| Complex annotations      | CORUM v4.0 (<https://mips.helmholtz-muenchen.de/corum>)       | Training/held-out complexes, evaluation gold standard |
| Gene Ontology            | GO (<https://geneontology.org>)                               | Functional annotation                                 |
| KEGG pathways            | KEGG (<https://www.kegg.jp>)                                  | Functional annotation                                 |
| Subcellular localization | UniProt (<https://www.uniprot.org>)                           | Functional annotation                                 |
| ESM-2 features           | ESM-2 (<https://huggingface.co/facebook/esm2_t33_650M_UR50D>) | Sequence embeddings                                   |

Pre-trained node embeddings and functional similarity matrices used for inference are provided in `artifacts/`:

- `node_embeddings_Cosine.pt` — graph-structure node embeddings (GE module output)
- `integrated_similarity_All.pt` — topology-aware functional similarity matrix B (full)
- `integrated_similarity_complex.pt` — complex-level similarity matrix

Input files required to run inference and evaluation are provided in `data/`:

- `graph.pkl` — global PPI graph (loaded by `Eval_Model-100.py`)
- `complex_graphs.pkl` — known complexes (evaluation gold standard, loaded by `Eval.py` / `Eval-Score.py`)
- `token_id_gene.pkl` — token ↔ gene mapping (loaded by evaluation scripts)
- `gene_names.pt` — vocabulary
- `hub_nodes_ids.pkl` — hub node ids used as inference seeds
- `Complex_Train.pkl` / `Complex_Test.pkl` — held-out training/test split (loaded by the held-out evaluation pipeline)

Model checkpoints and training corpora are not included (large, regenerable via the scripts in `03_corpus_construction/` and `04_pretraining/`). Contact the corresponding author for access.

---

## Pipeline

```bash
# 0. Prepare base subgraphs and splits (01_data_preparation/main/)
python 1.GraphModule.py
python 2.ALL_Complex_Generate_multi.py        # ~20M noisy subgraphs
python 3.Complex_Generate_multi.py

# 1. Build BFS corpus (03_corpus_construction/)
python GraphTransform_BFS_Pretrain.py         # -> pretrain corpus
python GraphTransform_BFS_SFT-1.py            # -> SFT corpus

# 2. Pre-training (04_pretraining/)
python Pretrain_Cosine.py --epochs 5 --batch_size 64 --learning_rate 5e-4 \
    --max_seq_len 256 --hidden_size 512 --num_hidden_layers 16 \
    --use_external_embedding True --external_embedding_path ../artifacts/node_embeddings_Cosine.pt \
    --data_path ../pretrain_combined.json --out_dir ../out_pretrain

# 3. Supervised fine-tuning (05_fine_tuning/)
python SFT_Cosine.py --epochs 2 --batch_size 64 --learning_rate 5e-6 \
    --max_seq_len 128 --data_path ../sft_0.json --out_dir ../out_sft

# 4. Inference and evaluation (07_inference_evaluation/)
python Eval_Model-100.py --output_file generation_results.pkl
python Eval.py
python Eval-Score.py                            # Precision / Recall / F-measure / Global MMR / Mean MMR

# 5. (Optional) Held-out zero-shot evaluation (09_heldout_evaluation/)
python Eval_Model.py && python Eval.py
```

---

## License

MIT License. See `LICENSE`.

If you use GRACE in your work, please cite:

> *(Citation placeholder — to be updated upon publication.)*
