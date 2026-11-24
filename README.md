Research project using the CT-RATE chest CT dataset.

## Container

Built from the lab's recon-dev image. To start:

'''bash
docker compose -f docker-compose-ctrate.yml up -d
docker exec -it aoibhe-ct-rate bash
'''

## Data

CT-RATE lives at '/data/ct-rate' on AXIS03, mounted into the
container at '/workspace/data'. Not tracked in git.

Note: the original NIfTI files have an identity affine with no
spacing or orientation information. This is fixed in the 'valid_fixed' /
'train_fixed' folders from the Hugging Face repo. However, affine doesn't
matter for CT-CLIP usage; the non-fixed data is fine. 

## Layout

- 'docker-compose-ctrate.yml' — container definition


## Downloading the CT-RATE data

CT-RATE is hosted on Hugging Face at [ibrahimhamamci/CT-RATE](https://huggingface.co/datasets/ibrahimhamamci/CT-RATE). We only use the **validation split** and only download what we need: the three CSV files, the CT-CLIP weights, and a chosen set of CT volumes.

All commands below are run **inside the container**, where the data drive `/data/ct-rate` on AXIS03 is mounted as `/workspace/data`.

### 1. One-time setup: account, access and token

1. Create a free account at [huggingface.co](https://huggingface.co).
2. Open the [CT-RATE dataset page](https://huggingface.co/datasets/ibrahimhamamci/CT-RATE) while logged in, fill in the short form (name, institution, email) and agree to the terms. The dataset is gated, so downloads fail until this is approved.
3. Create an access token: profile picture → **Settings → Access Tokens → Create new token**, type **Read**. The name doesn't matter.
4. Log in from the container and paste the token when asked:

   ```bash
   hf auth login
   ```

   (Older versions of `huggingface_hub` use `huggingface-cli login` instead.) The token is stored in your home folder inside the container. **Never put the token in a script or commit it to this repo.**

5. Turn off Hugging Face's newer "Xet" download backend, which fails on AXIS03 with a `CAS Client Error`. Add this to `~/.bashrc` so it's always set:

   ```bash
   echo 'export HF_HUB_DISABLE_XET=1' >> ~/.bashrc
   source ~/.bashrc
   ```

### 2. Download the CSV files

These are small and describe every validation volume: scan metadata (spacing, HU rescale, scanner, age and sex), the 18 abnormality labels, and the radiology reports.

```bash
hf download ibrahimhamamci/CT-RATE --repo-type dataset --local-dir /workspace/data \
    dataset/metadata/validation_metadata.csv \
    dataset/multi_abnormality_labels/valid_predicted_labels.csv \
    dataset/radiology_text_reports/validation_reports.csv

# move them up one level, to where our scripts look for them
mv /workspace/data/dataset/metadata \
   /workspace/data/dataset/multi_abnormality_labels \
   /workspace/data/dataset/radiology_text_reports  /workspace/data/
```

### 3. Download the CT-CLIP weights

```bash
hf download ibrahimhamamci/CT-RATE --repo-type dataset --local-dir /workspace/data \
    models/CT-CLIP-Related/CT-CLIP_v2.pt
```

### 4. Download CT volumes with `select_cohort.py`

`scripts/select_cohort.py` uses the metadata and label CSVs to choose volumes, writes the list to a manifest (a CSV of the chosen volumes), and downloads only those files. There are two ways to use it: **(A) the whole validation set**, or **(B) a selected cohort**.

Some background: the validation split has **1,304 patients, 1,564 scans and 3,039 volumes**. Most scans were saved as more than one reconstruction (`_1`, `_2`, ...) of the same scan. The CT-RATE paper reports its validation results per scan (1,564), so we download **one reconstruction per scan**, choosing `_1` when it exists.

Every command below is run from the scripts folder:

```bash
cd /workspace/src/scripts
```

Without `--download`, any of these commands only prints what it would pick and writes the manifest, so you can check before downloading. If a download stops partway, re-run the same command; files already downloaded are skipped.

#### A. All validation scans (1,564)

```bash
python select_cohort.py --all \
    --metadata /workspace/data/metadata/validation_metadata.csv \
    --labels   /workspace/data/multi_abnormality_labels/valid_predicted_labels.csv \
    -o cohort_all.csv --download --out-dir /workspace/data
```

This is a large download that runs for a long time, so:

- **Check free space first** with `df -h /workspace/data`. The script prints each file's size as it goes; after the first few, multiply the typical size by 1,564 to estimate the total.
- **Run it in the background so it survives a disconnect.** If VS Code or your SSH session drops, a command running in the terminal is killed. Instead start it like this, then follow the log:

  ```bash
  nohup python select_cohort.py --all \
      --metadata /workspace/data/metadata/validation_metadata.csv \
      --labels   /workspace/data/multi_abnormality_labels/valid_predicted_labels.csv \
      -o cohort_all.csv --download --out-dir /workspace/data \
      > download_all.log 2>&1 &

  tail -f download_all.log      # Ctrl+C stops watching; the download keeps going
  ```

  At the end the log says `downloaded N/1564` and lists any failures. Re-run the same command to retry them.

#### B. A selected cohort

```bash
python select_cohort.py \
    --metadata /workspace/data/metadata/validation_metadata.csv \
    --labels   /workspace/data/multi_abnormality_labels/valid_predicted_labels.csv \
    -o cohort.csv --download --out-dir /workspace/data
```

With no other options this picks the **default 10-volume cohort**. For each of lung nodule, emphysema, bronchiectasis and pulmonary fibrotic sequela, it takes one male and one female scan where that is the *only* positive finding. It then adds one healthy male and one healthy female scan (no positive findings).

To choose differently, add one of these:

| Option | What it selects |
|---|---|
| `--criteria my_cohort.json` | your own criteria. Run `python select_cohort.py --write-example-criteria my_cohort.json` first to get a template (the default cohort) to edit. Criteria can require or exclude findings, cap the number of findings, and filter by sex, age or scanner manufacturer |
| `--random N --seed S` | N randomly chosen patients, every scan of theirs, one reconstruction each. The same seed always gives the same patients |

**Note:** the repo also has a `valid_fixed` folder. We use `valid` (the default), because CT-CLIP reads voxel spacing from the metadata CSV, which is what its code expects.

### Resulting layout

```
/data/ct-rate/                          (= /workspace/data in the container)
├── metadata/validation_metadata.csv
├── multi_abnormality_labels/valid_predicted_labels.csv
├── radiology_text_reports/validation_reports.csv
├── models/CT-CLIP-Related/CT-CLIP_v2.pt
└── dataset/valid/
    └── valid_74/                       patient
        └── valid_74_a/                 scan
            └── valid_74_a_1.nii.gz     volume (reconstruction 1)
```

`scripts/run_zero_shot_ctrate.py` reads from exactly these paths.