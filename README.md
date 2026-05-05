# Loki2
<img src="docs/_images/Loki2.png" width="400" title="Loki" alt="Loki" align="right" vspace = "50">
Loki2 is a single-cell pathology foundation model that uses an encoder–decoder architecture to reliably translate nucleus morphology into molecular-level information. Loki2 is trained on UniSeg, a pan-tissue resource of 15 million cells with paired segmentation and transcriptomic measurements from multiple spatial technologies. This training strategy enables three capabilities from hematoxylin and eosin (H&E) images alone: universal cell type inference, in silico spatial transcriptomics by retrieving transcriptionally matched cells from reference atlases, and continuous morphological pseudotime inference of cell state transitions. By further aggregating single-cell features, Loki2 supports in silico protein staining and whole-slide clinical inference. These results show that routine histology contains far richer molecular and cellular information than previously recognized, and that Loki2 provides a general framework for accessing this information across technologies, tissues, disease contexts, and scales.

The pre-train weights and source code will be released on GitHub and Hugging Face after the manuscript is accepted.


## User Manual and Notebooks
User manual and notebook walkthroughs are available at [GitHub](https://guangyuwanglab2021.github.io/pathweb/).
This README provides a quick overview of how to set up and use Loki2.


## Demonstration Data 
Please find the demonstration data for the tutorial [here](https://drive.google.com/drive/folders/19qzd8jTZBQ57a3N5xbiZFSGZJNpIz8NQ?usp=sharing).


## Pretrained Weights
The pretrain weights will be released on GitHub and Hugging Face after the manuscript is accepted.


## Source Code
All source code for Loki2 is contained in the `./src/loki2` directory.
The source code will be released on GitHub and Hugging Face after the manuscript is accepted.


## Project Structure
Please organize your project folders as follows:
```
.
├── src/                 # Source code and conda environment file
├── model_ckpt/          # Downloaded model checkpoints (loki2_checkpoint.pth)
├── data/                # Input data (WSI, .h5ad, metadata)
├── notebooks/           # Local notebooks (copied from GitHub/tutorial materials)
└── outputs/             # Generated outputs
```


## Installation

1. **Navigate to the Loki2 source directory and create a conda environment**:
   ```bash
   cd ./src
   conda env create -f environment.yaml
   conda activate loki2_env
   ```
   
2. **Install Loki2**:
   ```bash
   pip install .
   ```
   

## Run the Model

### Whole Slide Images Cell Segmentation and Annotation
See [Notebook - Loki2 Cell Type Inference](https://guangyuwanglab2021.github.io/pathweb/notebooks/Loki2_cell_type_inference.html) for more details.

```bash
MODEL="../model_ckpt/loki2_checkpoint.pth"
OUTDIR="../outputs/cell_infer"
mkdir -p "$OUTDIR"

FILE="../data/cell_infer/colon_cancer_sample.tif"
WSI_PROPERTIES='{"slide_mpp": 0.25, "magnification": 40}'

echo "Processing ${FILE}"
python ../src/loki2/detect_cells.py \
  --model "$MODEL" \
  --outdir "$OUTDIR" \
  --geojson \
  --graph \
  process_wsi \
  --wsi_path "$FILE" \
  --wsi_properties "$WSI_PROPERTIES"
```


### Single-cell Morphology-to-Transcriptome Retrieval
See [Notebook - Loki2 Morphology-to-Transcriptome Retrieval](https://guangyuwanglab2021.github.io/pathweb/notebooks/Loki2_morph_retrieve.html) for more details.

1. Prepare Finetuning Data
```bash
conda activate loki_env

DATA_PATH="../data/morph_retrieve/P1CRC_VISIUMHD_LOKI2_mask.h5ad"
OUTPUT="../outputs/morph_retrieve/output/P1CRC_cell_trans_emb_raw.pt"

python ../src/loki2/encode_trans.py "$DATA_PATH" \
    --output "$OUTPUT" \
    --batch-size 1024 \
    --num-threads 32 \
    --device cuda

python ../src/loki2/cl/prepare_training.py \
    --dataset-name P1CRC \
    --trans-path "$OUTPUT" \
    --morph-path ../data/morph_retrieve/P1CRC_cell_morph_emb.pt \
    --output-dir ../outputs/morph_retrieve/output/P1CRC_train \
    --shard-size 10000
```

2. Finetuning
```bash
conda activate loki_env

DATASET="${DATASET:-P1CRC}"
RUN_NAME="${RUN_NAME:-${DATASET}_wds_vanilla}"
RUN_DIR="../outputs/morph_retrieve/output/runs/${RUN_NAME}"

TRAIN_DIR="../outputs/morph_retrieve/output/P1CRC_train/${DATASET}/train"
VAL_DIR="../outputs/morph_retrieve/output/P1CRC_train/${DATASET}/val"
TRAIN_META="${TRAIN_DIR}/manifest_train.csv"
VAL_META="${VAL_DIR}/manifest_val.csv"

if [[ ! -f "${TRAIN_META}" ]]; then
  echo "Missing train manifest: ${TRAIN_META}" >&2
  exit 1
fi

if [[ ! -f "${VAL_META}" ]]; then
  echo "Missing validation manifest: ${VAL_META}" >&2
  exit 1
fi

mapfile -t TRAIN_SHARDS < <(find "${TRAIN_DIR}" -maxdepth 1 -type f -name 'shard-*.tar' | sort)
mapfile -t VAL_SHARDS < <(find "${VAL_DIR}" -maxdepth 1 -type f -name 'shard-*.tar' | sort)

if [[ ${#TRAIN_SHARDS[@]} -eq 0 ]]; then
  echo "No training shards found in ${TRAIN_DIR}" >&2
  exit 1
fi

if [[ ${#VAL_SHARDS[@]} -eq 0 ]]; then
  echo "No validation shards found in ${VAL_DIR}" >&2
  exit 1
fi

TRAIN_SHARD_LIST=$(IFS=, ; echo "${TRAIN_SHARDS[*]}")
VAL_SHARD_LIST=$(IFS=, ; echo "${VAL_SHARDS[*]}")

mkdir -p "${RUN_DIR}"

python ../src/loki2/cl/train_projection_wds.py \
  --train-shards "${TRAIN_SHARD_LIST}" \
  --train-meta "${TRAIN_META}" \
  --val-shards "${VAL_SHARD_LIST}" \
  --val-meta "${VAL_META}" \
  --num-layers 1 \
  --batch-size 4096 \
  --epochs 20 \
  --lr 5e-4 \
  --device cuda \
  --amp \
  --save-every 1 \
  --output-dir "${RUN_DIR}" \
  --log-file train.log
```

3. Retrieve from scRNA Data
```bash
conda activate loki_env

sam="CRC_sc"
DATA_PATH="../data/morph_retrieve/CRC_sc.h5ad"
OUTPUT="../outputs/morph_retrieve/output/${sam}_trans.pt"

python ../src/loki2/encode_trans.py "$DATA_PATH" \
    --output "$OUTPUT" \
    --batch-size 1024 \
    --num-threads 32 \
    --device cuda

conda deactivate
conda activate loki2_env

CHECKPOINT_DIR="../outputs/morph_retrieve/output/runs/P1CRC_wds_vanilla"

# Set the epoch to use for projection (default: 20)
EPOCH=${EPOCH:-20}
CKPT_PATH="${CHECKPOINT_DIR}/projection_cl_epoch${EPOCH}.pt"

if [[ ! -f "${CKPT_PATH}" ]]; then
  echo "Checkpoint not found: ${CKPT_PATH}" >&2
  exit 1
fi

declare -A SAMPLE_MAP=(
  ["Cancer_P2"]="P2CRC"
)

for label in Cancer_P2; do
  dataset="${SAMPLE_MAP[$label]}"
  morph_path="../data/morph_retrieve/${SAMPLE_MAP[$label]}_cell_morph_emb.pt"
  output_dir="../outputs/morph_retrieve/output/data_projection/${dataset}"

  if [[ ! -f "${morph_path}" ]]; then
    echo "Skipping ${label}: missing morphology embeddings at ${morph_path}" >&2
    continue
  fi

  echo "Projecting ${label} (dataset: ${dataset})"
  mkdir -p "${output_dir}"

  python ../src/loki2/cl/project_raw_embeddings.py \
    --checkpoint "${CKPT_PATH}" \
    --morph-path "${morph_path}" \
    --modality morph \
    --batch-size 4096 \
    --normalized \
    --tag "${label}_epoch${EPOCH}" \
    --output-dir "${output_dir}"
done

TRANS_PATH="../outputs/morph_retrieve/output/CRC_sc_trans.pt"
OUTPUT_DIR="../outputs/morph_retrieve/output/data_projection/sc"

if [[ ! -f "${TRANS_PATH}" ]]; then
  echo "Transcription embeddings missing: ${TRANS_PATH}" >&2
  exit 1
fi

mkdir -p "${OUTPUT_DIR}"

python ../src/loki2/cl/project_raw_embeddings.py \
  --checkpoint "${CKPT_PATH}" \
  --trans-path "${TRANS_PATH}" \
  --modality trans \
  --batch-size 4096 \
  --normalized \
  --tag "epoch${EPOCH}" \
  --output-dir "${OUTPUT_DIR}"

declare -A SAMPLE_MAP=(
  ["Cancer_P2"]="P2CRC"
)

for label in "${!SAMPLE_MAP[@]}"; do
    dataset="${SAMPLE_MAP[$label]}"
    output_dir=${3:-"../outputs/morph_retrieve/output/result_centroid/retrieve_epoch${EPOCH}"}
    mkdir -p ${output_dir}
    echo "Processing sample: ${dataset}, output directory: ${output_dir}"
    python ../src/loki2/retrieve_from_sc.py ${dataset} ${label} ${EPOCH} ${output_dir}
done
```

### Run Notebooks for [Morphological Pseudotime Inference](https://guangyuwanglab2021.github.io/pathweb/notebooks/Loki2_morph_psdtime_CRC.html), [In silico Immunostaining](https://guangyuwanglab2021.github.io/pathweb/notebooks/Loki2_in_silico_immunostaining.html), and [Cell-level Multiple Instance Learning for Cancer Patient Staging](https://guangyuwanglab2021.github.io/pathweb/notebooks/Loki2_multiple_instance_learning.html)

1. Download/copy the tutorial notebooks from [GitHub](https://guangyuwanglab2021.github.io/pathweb/) into `./notebooks`.
2. Activate the Loki2 environment:
   ```bash
   conda activate loki2_env
   ```
3. Start Jupyter from repository root:
   ```bash
   jupyter notebook
   ```
4. Open notebooks from `./notebooks` and set the kernel to `loki2_env`.


## Python API
After installation, Loki2 modules are importable in Python scripts and notebooks:

```python
import loki2.preprocess
import loki2.plot
import loki2.retrieve
import loki2.psdtime
import loki2.immstain
import loki2.mil
```
