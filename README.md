# Control Performance Monitoring — GIMSCOP SISO

A reproducible Python pipeline for **Control Performance Monitoring (CPM)** of industrial SISO control loops, applied to the publicly available [GIMSCOP SISO dataset](https://www.ufrgs.br/gimscop/) published by UFRGS (Brazil).

The project comprises two notebooks:
- **`CPM_Use_Case_GIMSCOP.ipynb`** — deterministic KPI computation pipeline
- **`CPM_AI_Interpretation.ipynb`** — AI-assisted interpretation layer (Mistral API)

---

## What this project does

Most industrial plants collect historian data from hundreds of control loops 24/7. The challenge is converting that data into actionable diagnostic information.

This pipeline selects **11 contemporaneous loops** from the GIMSCOP dataset — all sharing the same 60-hour operating window (6–8 October 2017) — and computes a comprehensive set of model-free KPIs across **7 non-overlapping 8-hour windows per loop** (77 observations total). The temporal windowing enables shift-level monitoring and reveals degradation trends invisible in single-window analyses.

A second notebook sends the pre-computed KPI summaries to a language model (Mistral), which returns structured plain-language diagnoses prioritised for engineering and maintenance review.

---

## Why SISO-SEL and not SISO-SAMP

The GIMSCOP dataset is distributed in three variants:

| Variant | Description |
|---|---|
| `SISO-RAW.h5` | Raw historian — real timestamps, exception logging, signals not aligned |
| `SISO-SEL.h5` | Curated fragments from RAW — same structure, real timestamps |
| `SISO-SAMP.h5` | SEL resampled to constant grid — flat layout, integer sample index |

**SISO-SAMP is convenient but unsuitable for comparative analysis**: its fragments are 1–3 hours long and come from different calendar periods for different loops. Comparing KPIs across loops that were recorded at different times is not valid — plant conditions, feed quality and utility loads may differ.

**SISO-SEL** is used instead. Eleven loops have fragments covering the full 60-hour window from 6 to 8 October 2017. All 11 loops are contemporaneous, making cross-loop KPI comparisons and temporal monitoring methodologically sound.

---

## KPI families

### Oscillation
| KPI | Method |
|---|---|
| OI_zc | Zero-crossing rate — Hägglund (1995) |
| OI_reg | ACF regularity index — Thornhill, Huang & Zhang (2003) |
| T_osc | Dominant spectral period (FFT peak) |
| P_osc | Spectral power fraction at dominant frequency |

> **Note on OI_zc:** OI_zc ≈ 2·Δt/T. At Δt = 2.059 s, the threshold of 0.30 corresponds to T ≈ 14 s. For slow process oscillations (T > 30 min), OI_zc will be near zero even when the oscillation is real. Always combine with OI_reg and P_osc.

### Valve activity
TV_norm, TV_abs, Direction Change Rate, empirical OP bound fraction

### Tracking error *(SP coverage ≥ 5%)*
MAE, RMSE, Bias, IAE_norm

### Positioner tracking *(MV available)*
| KPI | Interpretation |
|---|---|
| op_mv_bias | Systematic OP–MV offset — possible calibration mismatch |
| op_mv_corr | Dynamic tracking quality |
| tv_mv_op_ratio | MV travel / OP travel — attenuated response if < 0.60; MV ≠ valve position if > 3.0 |

---

## Key findings

- **FIC28**: persistent regular oscillation (OI_reg up to 18 in windows 3–6) with attenuated valve response (TV_ratio = 0.56). Cascade slave operation confirmed — oscillation source is likely the upstream master loop.
- **LIC42**: growing positive bias (+0.14 → +0.26 over 56 hours), consistent with a progressively destabilising over-tuned level controller. MV signal is not valve position (TV_ratio = 12.2).
- **FIC12, PIC47**: systematic OP–MV offset > 0.11, consistent with positioner calibration mismatch.
- **FIC38**: TV_ratio = 0.39 — valve travels only 39% of what the controller commands.
- **LIC43, TIC52**: near-perfect OP–MV consistency, stable tracking — healthy loops.

---

## Project structure

```
CPM_Final/
├── CPM_Use_Case_GIMSCOP.ipynb    # KPI pipeline
├── CPM_AI_Interpretation.ipynb   # AI interpretation layer
├── .env                          # API key (never in git)
├── .gitignore
├── requirements.txt
├── outputs/
│   ├── figures/                  # generated plots (excluded from git)
│   ├── cpm_kpi_windows.csv
│   ├── cpm_kpi_windows_flagged.csv
│   ├── cpm_loop_summary.csv
│   └── cpm_ai_diagnosis.csv
└── data_gimscop/                 # downloaded automatically (excluded from git)
```

---

## How to run

### 1. Install dependencies

```bash
pip install -r requirements.txt
```

### 2. Run the CPM pipeline

Open and run all cells in **`CPM_Use_Case_GIMSCOP.ipynb`**.

The notebook downloads the GIMSCOP archive automatically (~204 MB from Google Drive) on first run. All outputs are saved to `outputs/`.

### 3. Run the AI interpretation *(optional)*

Create a `.env` file in the project root:

```
MISTRAL_API_KEY=your_key_here
```

Get a free API key at [console.mistral.ai](https://console.mistral.ai).

Open and run all cells in **`CPM_AI_Interpretation.ipynb`**.

> The AI notebook uses Mistral's API via the OpenAI-compatible endpoint. The `mistralai` Python package is not required.

---

## Dependencies

```
numpy / pandas / matplotlib / scipy   — data processing and visualisation
h5py / tables                          — HDF5 file reading (GIMSCOP format)
gdown                                  — automatic dataset download
openai                                 — Mistral API client (OpenAI-compatible)
python-dotenv                          — secure API key loading
```

---

## Dataset

**GIMSCOP SISO** — published by the GIMSCOP group, Federal University of Rio Grande do Sul (UFRGS), Brazil.

> Dambros, J. W. V., Trierweiler, J. O., & Farenzena, M. *Oscillation Detection in Process Industries – Part I: Review of the Detection Methods.* Journal of Process Control.

Dataset available at: [https://www.ufrgs.br/gimscop/repository/sisoviewer/datasets/](https://www.ufrgs.br/gimscop/repository/sisoviewer/datasets/)

---

## Limitations

1. **No process topology**: multi-loop interaction analysis (CCF, RGA) is not possible without the process interconnection structure, which is not documented in the dataset.
2. **No controller MODE signal**: KPIs computed during manual operation inflate error metrics. Manual-mode periods cannot be excluded.
3. **SP rescaling**: FIC29 and FIC38 had SP in engineering units at the historian source. Min-max rescaling was applied — error-based KPIs for these loops are approximate.
4. **Positioner diagnostics are screening indicators**: tv_mv_op_ratio and op_mv_bias flag loops for further investigation. Confirming stiction, calibration faults or backlash requires valve signature tests, maintenance records and plant context.
5. **AI diagnosis is an interpretation layer**: the language model interprets pre-computed KPIs. It does not have access to raw data and should not be treated as a root-cause engine. All diagnoses require engineering verification.

---

## References

- Hägglund, T. (1995). A control-loop performance monitor. *Control Engineering Practice*, 3(11).
- Thornhill, N. F., Huang, B., & Zhang, H. (2003). Detection of multiple oscillations in control loops. *Journal of Process Control*, 13(1).
- Jelali, M. (2013). *Control Performance Management in Industrial Automation*. Springer.
- Desborough, L., & Miller, R. (2002). Increasing customer value of industrial control performance monitoring. *AIChE Symposium Series*, 98(326).
