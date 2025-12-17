# Genesis — Genotyping Pipeline Tutorial (IDAT → GTC → VCF → PLINK)

This repository provides a **beginner-friendly**, **reproducible** tutorial that demonstrates a typical SNP microarray genotyping flow:

**IDAT (raw intensities)** → **GTC (genotype calling)** → **VCF (standard variant format)** → **PLINK (bed/bim/fam)**

All steps are executed as **Bash commands inside a Jupyter notebook** (`%%bash`), so learners can clearly see **inputs → commands → outputs → validation**.

---

## What You’ll Learn

* What each file type means (`.idat`, `.gtc`, `.vcf.gz`, `.bed/.bim/.fam`)
* Why each stage exists (calling, normalization, sorting, indexing, conversion)
* How to inspect outputs to confirm each step worked
* How to keep a pipeline **reproducible** via a single configuration file (`params.sh`)

---

## Repository Contents

Key files:

* `genotyping_pipeline_bash_tutorial.ipynb` — main notebook (bash-only tutorial)
* `prs-genesis-env.yml` — Conda environment (bcftools + gtc2vcf plugin + plink)
* `data/` — reference files and sample data location (paths configured in `params.sh`)
* `tools/` — vendor tool bundle (`iaap-cli`) + wrapper
* `params.sh` — pipeline configuration (created/edited from the notebook)

---

## Requirements

Recommended OS:

* Ubuntu Linux or **WSL2** (Windows)

Required tools:

* `git`
* `conda` (Miniconda/Anaconda/Mamba)

The pipeline uses:

* `iaap-cli` (vendor tool for genotype calling; provided separately)
* `bcftools` (+ `gtc2vcf` plugin)
* `plink`

---

## Quick Start

### 1) Clone the repo

```bash
cd ~
git clone https://github.com/synmutanX/genesis.git
cd genesis
```

### 2) Create the Conda environment

```bash
conda env create -f prs-genesis-env.yml
conda activate prs-genesis-env
```

Verify:

```bash
bcftools --version | head -n 2
plink --version | head -n 2
```

### 3) Install JupyterLab

If JupyterLab is not installed yet:

```bash
conda install -c conda-forge -y jupyterlab
```

---

## Setting up `iaap-cli` (from `.tar.gz`)

> `iaap-cli` is not installed via Conda. You typically receive it as a `.tar.gz` bundle.

Assume your `.tar.gz` is placed under `tools/`.

### 1) Extract and bundle it into `tools/iaap-cli-bin`

```bash
cd ~/genesis/tools

tar -xzf iaap-cli-linux-x64-1.1.0-*.tar.gz

rm -rf iaap-cli-bin
mkdir -p iaap-cli-bin

# Copy EVERYTHING from the bundle (important: dll/json/etc.)
cp -a ./iaap-cli-linux-x64-1.1.0-sha.*/iaap-cli/* ./iaap-cli-bin/
chmod +x ./iaap-cli-bin/iaap-cli
```

### 2) Create a wrapper to avoid ICU/.NET globalization issues

On some Ubuntu/WSL setups, `iaap-cli` can fail with an ICU/globalization error.
Use this wrapper so learners don’t need to export env vars manually each time:

```bash
cat > ./iaap-cli-bin/iaap-cli.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
export DOTNET_SYSTEM_GLOBALIZATION_INVARIANT=1
DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
exec "$DIR/iaap-cli" "$@"
EOF

chmod +x ./iaap-cli-bin/iaap-cli.sh
```

Test:

```bash
./iaap-cli-bin/iaap-cli.sh --help
```

### 3) Add it to PATH

```bash
echo 'export PATH="$HOME/genesis/tools/iaap-cli-bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

Test:

```bash
iaap-cli.sh --help
```

> If you use zsh, replace `~/.bashrc` with `~/.zshrc`.

---

## Required Input Data

Configure all paths in `params.sh` (the notebook helps generate this file).

Minimum inputs:

### Chip reference files

* `*.bpm` — manifest (binary)
* `*.csv` — manifest (CSV)
* `*.egt` — cluster file

### Reference genome

* `*.fasta` — reference genome (e.g., GRCh38)

If you see FASTA index errors, install samtools and index the FASTA:

```bash
conda install -c bioconda -y samtools
samtools faidx data/data-reference/human_v38.fasta
```

### IDAT sample folder

A folder containing paired IDAT files:

* `*_Red.idat`
* `*_Grn.idat`

### Mapping files

* `asa2rsid.map` — chip variant ID → rsID mapping
* `chr.map` — chromosome rename mapping (e.g., `chr1` → `1`)

(Optional for sex-check)

* population frequency file under `${POP_DIR}/${POP}/...` (used in the notebook if enabled)

---

## Running the Tutorial Notebook

From the repo root:

```bash
cd ~/genesis
conda activate prs-genesis-env
jupyter lab
```

In JupyterLab:

1. Open `genotyping_pipeline_bash_tutorial.ipynb`
2. Run cells **top-to-bottom** (Step 0 → Step 7)

> Even though the kernel may show Python, the tutorial cells run **Bash** via `%%bash`.

---

## Outputs You Should Expect

After a successful run, outputs are written under `output/`:

### Step 1 — IDAT → GTC

* `output/gtc/*.gtc`

### Step 2 — GTC → VCF

* `output/vcf/38/out_vcf.vcf.gz`
* `output/vcf/38/out_vcf.vcf.csi`
* `output/vcf/38/out_vcf.tsv`

### Step 3 — VCF → PLINK

* `output/plink/38/out_plink.bed`
* `output/plink/38/out_plink.bim`
* `output/plink/38/out_plink.fam`

### Additional exports

* `output/allel/38/variant.list`
* `output/vcf/38/out23.txt`

---

## Troubleshooting

### `iaap-cli` not found

* Confirm wrapper exists:

  ```bash
  ls -lh ~/genesis/tools/iaap-cli-bin/iaap-cli.sh
  ```
* Confirm PATH contains `iaap-cli-bin`:

  ```bash
  echo $PATH | tr ':' '\n' | grep iaap-cli-bin
  ```
* Test:

  ```bash
  iaap-cli.sh --help
  ```

### ICU/.NET globalization error

Use the wrapper `iaap-cli.sh`.
If running directly:

```bash
export DOTNET_SYSTEM_GLOBALIZATION_INVARIANT=1
./iaap-cli --help
```

### `bcftools +gtc2vcf` plugin missing

Check:

```bash
bcftools plugin -l | grep gtc2vcf
```

Make sure `bcftools-gtc2vcf-plugin` is installed in the conda env.

### FASTA indexing errors

```bash
samtools faidx path/to/reference.fasta
```

---

## Reproducibility Notes (for Classes)

To make results consistent across students:

* Use the **same OS environment** (Ubuntu/WSL recommended)
* Use the **same conda environment** (`prs-genesis-env.yml`)
* Use the **same input files** (chip bundle + reference FASTA + mappings)
* Run notebook **in order**
* Keep outputs in the default `output/` structure

---

## Licensing & Data

* `iaap-cli` is a vendor tool with its own license/distribution rules.
* Any chip/reference/sample data may have restrictions — only share if permitted.

---