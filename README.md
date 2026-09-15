# NSCLC CCRT — RECIST Annotation & Response Calculation Pipeline

End-to-end workflow for generating radiologist-annotated RECIST 1.1 datasets for stage III NSCLC patients treated with concurrent chemoradiotherapy (CCRT), and converting those annotations into structured RECIST response and failure-analysis tables.

The pipeline has three stages across two notebooks:

| Stage | Notebook | What it does |
|---|---|---|
| **1. Dataset creation & upload** | `SR_workflow_tagging_V4.ipynb` | Build the patient/scan cohort from source tables, assemble DICOM series records, and push them to nference AI Studio for radiologist tagging |
| **2. Annotation export** | `SR_workflow_tagging_V4.ipynb` | Pull back the DICOM SR (Structured Report) annotations via the AI Studio API once radiologists finish marking, and flatten them to CSV |
| **3. RECIST & failure calculation** | `Recist_Calculations_V7.ipynb` | Merge SR measurements with the radiologist Grist sheet, compute SOD / NADIR / TL / NTL / NL / overall response, run failure-type (LRF vs DF) analysis, and export the final `TASK_DETAILS` and `RECIST_SUMMARY` CSVs |

The outputs of Stage 3 feed directly into the separate **RECIST QC package** (`recist_qc_modular_package`), which validates them before they're released.

---

## Prerequisites

- Access to an **nference workspace** with the `nferhub` SDK available
- Environment variables: `X_NFER_DATA`, `X_NFER_BASEURL`, `WORKSPACE_USER`, `WORKSPACE_USER_TOKEN`
- Python packages: `pandas`, `numpy`, `pydicom`, `requests`
- Read access to the project data mounts under `/data/<project_name>/shared/Nbs_<version>/`

Both notebooks start with a `GLOBAL CONSTANTS` cell — set `project_name`, `version_of_work`, and `version_of_data` there before running anything else. Everything downstream (folder paths, scratch table names, mapping-file paths) is derived from those three values.

```python
project_name    = "NferenceInternalWorkingProject"
version_of_work = "V8"
version_of_data = "V8"
```

---

# Stage 1 — Dataset Creation & AI Studio Upload

**Notebook:** `SR_workflow_tagging_V4.ipynb` (sections: *Importing CSVs* → *Uploading Image Data*)

### Inputs

| File / source | Purpose |
|---|---|
| `RADIOLOGY_DICOM.csv` | Study/series-level DICOM metadata (`PERSON_ID`, `STUDY_INSTANCE_UID`, `SERIES_INSTANCE_UID`, `STUDY_DATE`, `MODALITY_DESCRIPTION`, `HARMONIZED_STUDY_DESCRIPTION`) |
| `duke_cohort_*.csv` / `mayo_cohort_*.csv` | Clinical cohort: `person_id`, `study_index_date`, `crt_start_date`, `crt_end_date`, `overall_stage`, `t_value`, `n_value`, `m_value`, `priority` |
| `image_availability.csv` | De-identified images actually available in the workspace — used to confirm a series is retrievable |
| `TAGGING_DATASET_METADATA.csv` | Tracking sheet for which patients have already been pushed for tagging |

### Steps

1. **Connect clients** — `nferhub.client("cohort")`, `nferhub.client("sql_blaze")`, `nferhub.client("ai-studio")`.
2. **Prioritise & randomise the cohort** — shuffle with a fixed seed (`random_state=42`), then sort so `priority == True` patients come first. This gives a reproducible tagging queue.
3. **Filter radiology** — keep only `CT`, `PT`, `MR` modalities and studies from **2018 onwards**.
4. **Select a patient** and join their series against `image_availability` on `(nfer_pid, study_id, series_id, modality)` so only downloadable series are pushed.
5. **Categorise scan dates** — `categorize_scan_dates()` splits each patient's studies into:
   - **Baseline window:** `crt_start_date − 90 days ≤ STUDY_DATE < crt_start_date`
   - **Follow-up window:** `STUDY_DATE ≥ crt_end_date`
6. **Generate the dataset description** — `generate_dataset_description()` builds the free-text block the radiologist sees in AI Studio: study index date, stage, T/N/M, CCRT start/end, and the lists of baseline and follow-up scan dates.
7. **Build the image payload** — one record per series with `modality`, `patient_id`, `study_uid`, `series_uid`, `instance_uid`; all values cast to `str` (AI Studio requires string values).
8. **Create the dataset** on the existing experiment and assign taggers:

```python
dataset = experiment.create_dataset(
    name="dataset_<n>_<PERSON_ID>",
    description=dynamic_description,
    json_data={"images": records3},
)
dataset.add_taggers(taggers=user_list)
```

The experiment itself (`NSCLC_CCRT_RECIST_Tagging`, type `DicomSeries` / `Segmentation`, label `tumor`) is created once — that cell is commented out by default so it isn't recreated on every run.

> **Naming convention:** datasets are named `dataset_<sequence>_<PERSON_ID>`. Stage 3 parses the `PERSON_ID` back out of this name, so keep the format.

---

# Stage 2 — Exporting Annotated Data

**Notebook:** `SR_workflow_tagging_V4.ipynb` (section: *Exporting annotated data*)

Once radiologists have marked lesions in AI Studio, annotations come back as **DICOM SR** files.

### Download

```python
experiment_name = "NSCLC_CCRT_RECIST_Tagging"
dataset_name    = "dataset_6_93207188"
tagger_name     = "<tagger>@nfer-workspaces.com"

annotations = download_SR_for_dataset(experiment_name, dataset_name, tagger_name)
```

- `download_SR_for_dataset()` resolves the experiment → dataset → `export_dataset(tagger=[...], transform=False)`, then pulls each sample's `annotation_url`.
- `download_file()` fetches the blob from `X_NFER_BASEURL` using basic auth (`WORKSPACE_USER` / `WORKSPACE_USER_TOKEN`) and writes it to the system temp directory.
- Returns a list of `(SR_file_path, tagger_name)` tuples.

### Flatten SR → CSV

`create_csv_for_annotations()` reads each SR with `pydicom` and walks the Content Sequence (`0x0040A730`) recursively to pull out:

| Extracted field | DICOM source |
|---|---|
| `PixelCoordinates` | `SCOORD` → Graphic Data (`0x00700022`) |
| `PhysicalCoordinates` | `SCOORD3D` → Graphic Data |
| `InstanceUID` | `ReferencedSOPInstanceUID`, found by recursive traversal (not a fixed index) |
| `StudyUID` / `SeriesUID` | Referenced Study/Series sequences (`0x0040A375` → `0x00081115` → `0x00081199`) |
| `Label` | Lesion ID assigned by the radiologist (`TL1`, `NTL2`, `NL1`, …) |
| `Length` / `Width` / `Unit` | Numeric measurement values in the measurement group |
| `StudyDate`, `Modality`, `descriptions` | Top-level SR attributes (`descriptions` = last 2 tokens of `SeriesDescription`, i.e. the timepoint label) |

Post-processing: measurement blocks with no pixel coordinates or no referenced instance are skipped; duplicates are dropped on `(InstanceUID, PixelCoordinates, descriptions)`; `Length`/`Width` are floor-truncated to 4 decimals; pixel coordinates are rounded to integers; `PatientID` is renamed to `PERSON_ID`. Output is one CSV per patient.

> These per-patient CSVs are what Stage 3 reads from `raw_download/<version>/dicom_api/`.

---

# Stage 3 — RECIST & Failure Calculations

**Notebook:** `Recist_Calculations_V7.ipynb`

This is where the annotated geometry becomes clinical response data. Two formulas carry the notebook: **`calculate_master_recist_v9_8()`** (RECIST response) and **`analyze_failure_details_v16()`** (failure pattern).

## 3.1 Assembling the master table

1. **Load all downloaded SR CSVs** recursively from `raw_download/<V*>/dicom_api/**/*.csv`, tagging each row with its `source_file` and `version`, then concatenate and de-duplicate.
2. **Map clinic numbers → NFER_PID** by querying `DIM_PATIENT` (`PATIENT_CLINIC_NUMBER` → `NFER_PID`). Rows that don't map are dropped.
3. **Normalise timepoint labels** — regex-based cleanup of typo'd radiologist entries:
   - `^BAS[ELIN]*\d*` → `BASELINE` (catches `BASELEINE`, `BASELINE1`, …)
   - `^F[O0LWIU\s\[\]]*P*\s*\[?(\d+)` → `FOLLOWUP<n>` (catches `FLLOWUP 2`, `F/U[3]`, …)
   - Rows labelled `ERROR`, `IGNORE`, `TEST` are removed.
4. **Add the Z coordinate** — SR pixel coordinates are 2-D `[x1, y1, x2, y2, …]`. Joining against `FACT_SYN_DICOM_RADIOLOGY` on `(PERSON_ID, StudyUID, SeriesUID, InstanceUID)` gives `INSTANCE_NUMBER` (slice number), which is appended as Z to produce `[[x, y, z], …]`.
5. **Merge the Grist sheet** — the radiologists' tabular annotation export (`DR.* - RECIST 1.1 ANNOTATION-Table1.csv`) carries the clinical judgement fields that aren't in the SR: `SERIES_TYPE`, `LESION_ID`, `LESION_DESCRIPTION`, `LOCATION`, `LOCATION-OTHER`, `LN STATION`, `LATERALITY`, `ORIENTATION`, `TL_RESPONSE_CAT`, `NTL_RESPONSE`, `NL_RESPONSE`, `NL_TUMOR_STATUS`, `FAILURE_TYPE`, `REMARKS`.
   - `PERSON_ID` is recovered from the `dataset_<n>_<PERSON_ID>` header rows via forward-fill.
   - Merged onto the SR data with an **outer join** on `(PERSON_ID, SERIES_TYPE, LESION_ID)`.
   - `StudyDate`/`StudyUID`/`SeriesUID`/`Tagger` are ffill/bfill'd within `(PERSON_ID, SERIES_TYPE)`; lesion characteristics (`LATERALITY`, `LOCATION`, `LN STATION`, `LESION_DESCRIPTION`) are forward-filled within `(PERSON_ID, LESION_ID)` so a lesion keeps its description across follow-ups.
   - Lesion IDs are validated against `^(TL|NTL|NL)\d*$`.

## 3.2 `TASK_DETAILS` output

`process_radiology_tasks()` derives the lesion-level deliverable:

- **Diameter selection:** if the lesion description contains "node" **and** `Width` exists → use `Width` (short axis); otherwise use `Length` (longest diameter).
- **NE reason extraction:** any response field formatted `NE-<reason>` has its reason split out, checked in priority order TL → NTL → NL, and written to `NE_Reason`. Any row with a real NE reason gets `Diameter = "NE"`.
- **Other location:** where `Location` is `Other`/blank, `extract_other_location()` keyword-searches `Comments` + `LOCATION-OTHER` against an anatomical vocabulary (`RUL`, `LUL`, `Lobe`, `Lung`, `Hilar`, `Mediastinal`, `Liver`, `Adrenal`, `Bone`, `Pleura`, `Node`, `Mass`, `Nodule`).
- **Lobe mapping:** detailed segment labels (`RUL - Apical`, `LLL - Posterior basal`, …) are collapsed to the five lobes.
- **Lesion type:** inferred from the ID prefix — `^TL` → TL, `^NL` → NL, otherwise NTL.

Exported as **`TASK_DETAILS_V13.csv`**.

## 3.3 RECIST response formula — `calculate_master_recist_v9_8()`

### Step A — per-lesion `VAL_FOR_SOD`

The value each lesion contributes to the Sum of Diameters.

**Lymph nodes** (description contains `NODE`/`LYMPH`, or an LN station is populated, or location is subcarinal/paratracheal) use the **short-axis diameter** (`Width`, falling back to `Length`):

| Node condition | Contribution |
|---|---|
| SAD = 0 (not seen) | `0.0` |
| SAD > 10 mm (pathological) | `SAD` |
| SAD ≤ 10 mm, clearly visible | `SAD` as measured |
| SAD < 5 mm, faint/obscured | floored to `5.0` mm |
| SAD 5–10 mm, faint/obscured | `SAD` as measured |

**Non-nodal lesions** use the **longest diameter** (`Length`, falling back to `Width`), except that a `TL_RESPONSE_CAT` of *Complete Response* forces `0.0`.

### Step B — SOD per timepoint

`SOD = Σ VAL_FOR_SOD` over all TL rows in that `(PERSON_ID, SERIES_TYPE)` group.
If **any** target lesion is flagged NE, `SOD` is locked to the string `"NE"` and the timepoint is marked `IS_ALL_NE`.

### Step C — NADIR

Running minimum of non-NE, non-zero SODs since baseline, with two deliberate exceptions:

- **NE timepoints inherit the last valid nadir** rather than resetting it.
- **After a complete response** (SOD = 0), the nadir reverts to the last valid non-zero value on the next measurable timepoint, so post-CR regrowth is measured against a meaningful reference rather than zero.

### Step D — TL response

Evaluated in this precedence order:

1. No target lesions → `NA`
2. Any TL flagged NE → `NE`
3. All TLs marked *Complete Response* → `CR` (even if SOD > 0, e.g. a residual nodal short axis)
4. All non-nodal TLs at 0 **and** all nodal TLs < 10 mm → `CR`
5. `(SOD − NADIR) / NADIR ≥ 20%` **and** `(SOD − NADIR) ≥ 5 mm` → `PD`
6. `(SOD − BASELINE_SOD) / BASELINE_SOD ≤ −30%` → `PR`
7. Otherwise → `SD`

Plus an override: if the previous timepoint was a CR and the current SOD > 0, the response becomes `PD` (reappearance of disease).

### Step E — NTL and NL responses

- **NTL:** `PD` if any NTL is PD → else `NE` if any NTL is NE → else `CR` if all are CR → else `NN` (non-CR/non-PD). Always `NA` at baseline.
- **NL:** `YES` if any new lesion is confirmed → `NE` if equivocal → else `NO`.

### Step F — Overall timepoint response

| Condition | Overall |
|---|---|
| NL = YES, or NTL = PD, or TL = PD | `PD` |
| TL = PR, NTL = CR, NL = NO | `PR` |
| TL = CR, NTL = CR | `CR` |
| TL ∈ {CR, PR}, NTL ∈ {NN, NE} | `PR` |
| TL = SD, NTL ∈ {NN, NE, CR} | `SD` |
| TL = NE | `NE` |

Baseline rows return `BASELINE` for TL response and `NA` overall.

## 3.4 Failure analysis formula — `analyze_failure_details_v16()`

Classifies each progression event as **LRF** (loco-regional failure) or **DF** (distant failure), which is the endpoint of interest for stage III CCRT.

- **New lesions** (`NL_OVERALL_RESPONSE == YES`) and **progressing non-target lesions** (`NTL_OVERALL_RESPONSE == PD`) are classified using the radiologist's `FAILURE_TYPE` column: `BOTH` sets LRF and DF, `DF` sets DF only, `LRF` sets LRF only.
- **Target lesions** are evaluated at the **lesion level**, not just in aggregate. The function maintains a per-lesion nadir (seeded from baseline) and flags a lesion as failing when:

  ```
  (current − lesion_nadir) / lesion_nadir ≥ 0.20
  ```

  Every individually-progressing TL is added to the LRF list. The per-lesion nadir is updated *after* the check, and non-positive values (NE timepoints, disappeared lesions) are ignored so they can't corrupt the running minimum.
- **Point of failure location** is assembled from the `LOCATION` of every failing lesion, appending `LOCATION-OTHER` when present (e.g. `Other*-Right lower chest wall`). It falls back to the lesion's location at any timepoint if it isn't recorded at the current one.
- Column lookup is normalisation-based (`_norm_key`), so `LESION ID`, `Lesion_Id`, and `LESION_ID` all resolve — the function tolerates inconsistent Grist sheet headers.

Adds `LRF`, `LRF_Lesions_List`, `DF`, `DF_Lesions_List`, `Anatomical_location_of_the_PoF` to the summary. Baseline rows are always `no` / `NA`.

Exported as **`RECIST_SUMMARY_V13.csv`**.

## 3.5 Supporting outputs

- **`final_mayo_cohort.csv`** — cohort metadata joined to the baseline study date. `BASELINE_REFERENCE_DATE` is populated only when the baseline scan is within **28 days** of `crt_start_date`; where multiple baselines exist, the closest to CCRT start is kept. `crt_start_date`/`crt_end_date` are upper-cased to `CRT_START_DATE`/`CRT_END_DATE`.
- **`image_manifest.csv`** — distinct series count per `(PERSON_ID, StudyDate)`, used by the QC pipeline's zero-images check.

---

## Output Files

| File | Grain | Consumed by |
|---|---|---|
| `TASK_DETAILS_V13.csv` | One row per lesion per timepoint | RECIST QC pipeline |
| `RECIST_SUMMARY_V13.csv` | One row per patient per timepoint | RECIST QC pipeline |
| `final_mayo_cohort.csv` | One row per patient | QC timing checks (`03_BASELINE_ELIGIBILITY`, `08_CRT_TIMING`, `05_LONGITUDINAL_CONSISTENCY`) |
| `image_manifest.csv` | One row per patient per study date | QC check `01_SCHEMA_AND_INPUTS` |

Run the QC package against these before release:

```bash
python -m recist_qc_modular_package.cli --input-dir <output_csv_dir> --output-dir qc_outputs
```

---

## Full Pipeline Order

```
Cohort tables + RADIOLOGY_DICOM + image_availability
        │
        ▼  [Stage 1] SR_workflow_tagging — sample, window, describe, upload
   AI Studio experiment "NSCLC_CCRT_RECIST_Tagging"
        │
        ▼  radiologists annotate (segmentation + RECIST Grist sheet)
        │
        ▼  [Stage 2] SR_workflow_tagging — export_dataset → DICOM SR → per-patient CSVs
   raw_download/<version>/dicom_api/*.csv
        │
        ▼  [Stage 3] Recist_Calculations — merge, normalise, add Z, join Grist
   master_df_final
        │
        ├─► calculate_master_recist_v9_8()  → SOD / NADIR / TL / NTL / NL / OVERALL
        ├─► analyze_failure_details_v16()   → LRF / DF / PoF location
        │
        ▼
   TASK_DETAILS_V13.csv + RECIST_SUMMARY_V13.csv + final_mayo_cohort.csv + image_manifest.csv
        │
        ▼
   RECIST QC pipeline → passed_patients_final.csv / failed_patients_final.csv
```

---

## Notes & Caveats

- **Hard-coded paths.** Both notebooks read and write absolute `/data/...` paths tied to `Nbs_V8`. Update the `GLOBAL CONSTANTS` cell and the CSV load paths when moving to a new version.
- **Single-patient sampling.** Stage 1 currently filters to one hard-coded `PERSON_ID` (`sampled_df = df[df['PERSON_ID'] == '...']`) for dataset creation. Loop over the randomised queue to batch-process.
- **`create_csv_for_annotations()` is commented out** in the shipped notebook — uncomment the cell before running the SR-to-CSV step.
- **Version drift in function names.** The notebook defines `analyze_failure_details_v16()` but the call site invokes `analyze_failure_details_v12()`. Confirm which version you intend to run before executing.
- **Dataset name is the ID carrier.** Stage 3 parses `PERSON_ID` out of `dataset_<n>_<PERSON_ID>`; a renamed dataset breaks the join back to the Grist sheet.
- **Duplicate `prepare_image_manifest`** exists in the QC package's `normalize.py` — the second definition wins. Unrelated to these notebooks but relevant when debugging the handoff.
- **Grist sheet quality drives everything.** Timepoint labels, lesion IDs, and `FAILURE_TYPE` are free-text radiologist entries. The regex normalisation handles known typos; genuinely new variants will silently drop rows via the `^(TL|NTL|NL)\d*$` filter. Check `descriptions.value_counts()` and `LESION_ID.value_counts()` after normalisation on every new batch.
