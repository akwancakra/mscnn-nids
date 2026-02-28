# Dokumentasi Hasil Full Run Fase 2

Sumber hasil: `mscnn-nids/histories-read-only/notebook-full-evaluating-final-result.ipynb`  
Tanggal run (dari notebook): `2026-02-28 09:53`

## 1. Ringkasan Pipeline
- Preprocessing, feature selection, scaling, latent extraction, windowing: selesai.
- Training Stage 1/2 pada run ini menggunakan checkpoint (cached weights).
- Evaluasi dilakukan pada:
  - CIC-IDS-2017 (all labels)
  - CSE-CIC-IDS-2018 (all labels)

## 2. Arsitektur Model
- Stage 1: MSCNN-AE (kernel 3/5/7) -> bottleneck 8 dim
- Stage 2: BiLSTM-AE -> bottleneck 4 dim
- Compression ratio:
  - Stage 1: `50 / 8 = 6.2x`
  - Stage 2: `(10 x 8) / 4 = 20.0x`
- Parameter:
  - Stage 1: `33,193`
  - Stage 2: `40,844`

## 3. Data dan Feature
- CIC-2017 all labels: `2,830,743` rows
  - Benign: `2,273,097`
  - Attack: `557,646`
- Feature final setelah seleksi: `50` fitur (dari awal 77)
- Domain shift summary (CIC benign vs CSE benign):
  - Low shift: `6`
  - Medium shift: `14`
  - High shift: `30`

## 4. Thresholding
- Kandidat threshold dievaluasi dari beberapa metode (Z-score, Gaussian, IQR, MAD, Percentile).
- Threshold terpilih: **`zscore_k2.0 = 0.200325`**
- FPR validasi benign untuk threshold terpilih: **`0.0422`**

## 5. Hasil Evaluasi Utama

| Metric | CIC-2017 | CSE-2018 | Gap (CSE - CIC) |
|---|---:|---:|---:|
| ROC-AUC | 0.6644 | 0.2658 | -0.3987 |
| PR-AUC | 0.5016 | 0.2241 | -0.2775 |
| F1-Score | 0.3836 | 0.1598 | -0.2238 |
| Recall | 0.2697 | 0.1850 | -0.0847 |
| Precision | 0.6640 | 0.1407 | -0.5233 |
| FPR | 0.0335 | 0.5543 | +0.5208 |

Interpretasi singkat:
- Di CIC-2017, false positive relatif rendah (`FPR 3.35%`) tapi recall masih rendah.
- Di CSE-2018, performa turun tajam dan FPR sangat tinggi (`55.43%`), menandakan generalization gagal.

## 6. Per-Attack Detection Highlights

### CIC-2017
- Tinggi: `DDoS (DR 0.8313)`
- Sangat rendah: `PortScan (0.0006)`, `DoS Hulk (0.1590)`, `FTP-Patator (0.1337)`, `Bot (0.0127)`, beberapa web attacks mendekati `0`.

### CSE-2018
- Tinggi: `DoS attacks-GoldenEye (0.9480)`, `DoS attacks-Slowloris (0.8016)`
- Rendah/bermasalah: `DoS attacks-Hulk (0.1196)`, `FTP-BruteForce (0.0462)`, `DoS attacks-SlowHTTPTest (0.0000)`
- Terdapat label anomali: `Label (n=1)` pada output.

## 7. Catatan Kritis Kualitas Evaluasi
- Pada evaluasi CSE terdapat banyak warning fitur tidak ditemukan dan diisi `0.0`.
- Ini mengindikasikan mismatch skema nama kolom antar dataset, dan sangat mungkin berkontribusi besar terhadap drop performa CSE.

## 8. Verdict
**CIC-SPECIFIC / overfit terhadap distribusi CIC-2017.**  
Model belum memenuhi target generalisasi ke CSE-2018.

## 9. Rekomendasi Prioritas
1. Perbaiki normalisasi/mapping nama fitur CIC <-> CSE secara konsisten sebelum re-evaluasi.
2. Bersihkan data evaluasi CSE dari baris invalid/header leakage (indikasi label `Label`).
3. Re-run evaluasi setelah preprocessing lintas-domain konsisten (tanpa retrain dulu).
4. Setelah itu baru lanjut tuning threshold/weight Stage1-Stage2 atau teknik domain adaptation.

## 10. Stabilization Pass (Tanpa Retrain)

Perubahan pipeline evaluasi yang sudah ditambahkan:
- Utilitas mapping kolom bersama:
  - `normalize_colname`
  - `build_column_lookup`
  - `resolve_feature_column`
  - `summarize_feature_mapping`
- Konfigurasi validasi schema:
  - `STRICT_FEATURE_MATCH = True`
  - `MIN_FEATURE_MATCH_RATIO = 0.95`
  - `DROP_HEADER_LEAK_ROWS = True`
  - `FEATURE_ALIAS_MAP = {...}` (siap diisi alias fitur CIC<->CSE)
- Sanitasi header leakage saat load CSE per-chunk.
- Metadata coverage mapping per file CSE.
- Pre-scoring validation gate sebelum Section 13.2.

Checklist verifikasi setelah re-run 13.1 -> 13.2:
- [ ] Tidak ada label literal `Label` di distribusi kelas CSE.
- [ ] `max_missing_features == 0` (atau sesuai threshold yang diset).
- [ ] `min_match_ratio >= 95%`.
- [ ] 13.2 berjalan tanpa warning mismatch schema masif.

### Before vs After (diisi setelah re-run stabilisasi)

| Metric | Before (full run awal) | After stabilization |
|---|---:|---:|
| CSE ROC-AUC | 0.2658 | TBD |
| CSE PR-AUC | 0.2241 | TBD |
| CSE F1 | 0.1598 | TBD |
| CSE FPR | 0.5543 | TBD |
| CSE min feature match ratio | N/A (banyak missing warning) | TBD |
| CSE label leakage (`Label` class) | Present (`n=1`) | TBD |

## 11. Phase 3 Implementation (Generalization Boost, Budget Tinggi)

Perubahan sudah diimplementasikan pada notebook utama:
- File: `mscnn-nids/mscnn_bilstm_ae_nids.ipynb`
- Section baru: `16. PHASE 3 - GENERALIZATION EXPERIMENTS (A/B/C/D)`
- Default aman: `RERUN_PHASE3_EXPERIMENTS = False` (tidak auto-train sampai flag diaktifkan)

Config baru yang ditambahkan:
- `ENABLE_DOMAIN_ADAPTATION = True`
- `CSE_BENIGN_ADAPT_MAX_ROWS = 300000`
- `ADAPT_MIX_RATIO_CIC = 0.7`
- `ADAPT_MIX_RATIO_CSE = 0.3`
- `S1_FINETUNE_EPOCHS = 20`
- `S2_FINETUNE_EPOCHS = 20`
- `FINETUNE_LR_SCALE = 0.1`
- `RECALIBRATE_THRESHOLD_ON_MIXED_VAL = True`
- `EXPERIMENT_TAG = "phase3_gen_v1"`

Artifact output yang disiapkan:
- `results/phase3_ablation_summary.csv`
- `artifacts/phase3_best_config.json`
- `results/phase3_report.md`

Track eksperimen yang tersedia:
- `A_baseline_clean`: retrain bersih CIC benign.
- `B_cse_finetune`: fine-tune dari Track A memakai CSE benign.
- `C_mixed_train`: train campuran CIC/CSE benign.
- `D_score_calibrated`: grid kalibrasi bobot score `(0.7/0.3, 0.5/0.5, 0.3/0.7)` di model base terbaik.

Hardening yang ikut ditambahkan:
- Guard ketika data benign CSE tidak cukup untuk split adaptasi.
- Guard ketika bundle model base untuk Track D tidak tersedia.
- Perbaikan recalibration Track B agar menggunakan `val_session_ids` (bukan train ids) pada mixed validation.

Cara menjalankan Phase 3:
1. Jalankan notebook berurutan sampai selesai section evaluasi CIC+CSE.
2. Set `RERUN_PHASE3_EXPERIMENTS = True`.
3. Jalankan section 16.1.
4. Cek artifact `phase3_ablation_summary.csv` untuk pemilihan konfigurasi terbaik.

## 12. Implementasi Final v3_sessionfix+ (Sudah Dipasang di Notebook Utama)

File notebook yang diubah:
- `mscnn-nids/mscnn_bilstm_ae_nids.ipynb`

Perubahan inti yang sudah aktif:
- Versioning pipeline baru:
  - `PIPELINE_VERSION = "v3_sessionfix"`
  - `PREPROCESSED_DIR = .../preprocessed/v3_sessionfix`
  - `ARTIFACTS_DIR = .../artifacts/v3_sessionfix`
- Sessionization ketat:
  - `SESSION_KEY_MODE = "5tuple"`
  - `STRICT_SESSION_QUALITY = True`
  - `MIN_UNIQUE_SESSIONS = 10000`
  - `MIN_TIMESTAMP_VALID_RATIO = 0.90`
  - `MAX_SINGLE_SESSION_SHARE = 0.05`
  - `ALLOW_ROW_SPLIT_FALLBACK = False`
- Verifikasi timestamp robust:
  - resolver kolom canonical dengan normalisasi `strip().lower()`
  - deteksi aman untuk kolom seperti `" Timestamp"`
  - fallback deterministik `source_file + row_idx` bila timestamp invalid
- Latent health gate sebelum windowing:
  - hard fail jika ada `NaN/Inf`
  - hard fail jika `latent_std_max > LATENT_STD_MAX` (default `100.0`)
- Evaluasi CSE dengan score-direction diagnostic:
  - jika `ROC-AUC < 0.5`, hitung juga AUC untuk `1 - score`
  - simpan diagnosis, default tidak auto-invert (`AUTO_APPLY_SCORE_INVERSION=False`)

Artifact baru yang dihasilkan:
- `artifacts/v3_sessionfix/session_quality_report.json`
- `artifacts/v3_sessionfix/session_split_report.json`
- `artifacts/v3_sessionfix/latent_health_report.json`
- `artifacts/v3_sessionfix/windowing_quality_report.json`
- `artifacts/v3_sessionfix/score_direction_report.json`

### Gate Readiness (wajib lulus sebelum lanjut Phase 3 tuning)
- [ ] `unique_sessions >= 10000`
- [ ] `timestamp_valid_ratio >= 0.90`
- [ ] `largest_session_share <= 0.05`
- [ ] split mode bukan row fallback
- [ ] latent health pass (tanpa `NaN/Inf`, `latent_std_max <= 100`)
- [ ] CSE mapping coverage tetap 100% dan tanpa header leakage `Label`

## 13. Part 4 Implemented: `v4_scoringfix` (Score Direction + Scoring Redesign)

Tujuan Part 4:
- Tanpa retrain model.
- Reuse cache/checkpoint `v3_sessionfix` sebagai input.
- Redesign anomaly score agar direction issue teratasi dan evaluasi stabil.

Perubahan utama yang sudah diimplementasikan di notebook:
- Input/output namespace dipisah:
  - input: `SOURCE_PIPELINE_VERSION = "v3_sessionfix"`
  - output: `SCORE_PIPELINE_VERSION = "v4_scoringfix"`
- Section baru `10.5`:
  - `fit_stage_score_stats`
  - `normalize_stage_errors`
  - `combine_anomaly_score`
  - `direction_diagnostic`
  - `run_weight_grid_search`
- Scoring v4:
  - per-stage z-normalization dari benign validation
  - Stage 2 contribution di-invert saat `INVERT_STAGE2_SCORE=True`
  - mode: `combined_normalized | stage1_only | stage2_only`
- Weight tuning:
  - grid search `w1 in [0.3, 0.5, 0.7, 0.9, 1.0]`
  - tuning pada CIC calibration split 10% (stratified)
  - evaluasi final CIC di holdout 90%
- Threshold dihitung ulang dari `combined_val` v4.
- Diagnostic direction wajib dicetak di CIC dan CSE (`auc_raw` vs `auc_inverted`).

Artifact baru Part 4:
- `artifacts/v4_scoringfix/score_calibrator.json`
- `artifacts/v4_scoringfix/weight_grid_search_results.csv`
- `artifacts/v4_scoringfix/threshold_results_v4.json`
- `artifacts/v4_scoringfix/best_threshold_v4.json`
- `artifacts/v4_scoringfix/cic_calibration_metrics.json`
- `artifacts/v4_scoringfix/cic_holdout_metrics.json`
- `artifacts/v4_scoringfix/cse_metrics.json`
- `artifacts/v4_scoringfix/score_direction_report_v4.json`
- `results/v4_scoringfix/part4_summary.md`

## 14. Part 5 Implemented: `v5_domainfix` (Domain Shift + Target Threshold Calibration)

Tujuan implementasi Part 5:
- Tidak retrain model.
- Reuse cache/model `v3_sessionfix` + scorer baseline `v4_scoringfix`.
- Mitigasi domain shift + threshold calibration target domain.

Perubahan utama:
- Config pipeline baru:
  - `PIPELINE_VERSION = "v5_domainfix"`
  - `SOURCE_PREPROCESS_VERSION = "v3_sessionfix"`
  - `SOURCE_SCORING_VERSION = "v4_scoringfix"`
- Strategy domain shift (non-retrain):
  - `mask_high_shift` dengan anchor **CIC benign median** (locked).
  - `clip_high_shift` dengan bounds **CIC p1-p99**.
- Score sign untuk CSE dikunci dari report direction v4 (`auto_from_part4`), tidak jadi variable search.
- Matrix evaluasi locked 4 kombinasi:
  - `mask_high_shift + cic_benign_threshold`
  - `mask_high_shift + cse_benign_threshold`
  - `clip_high_shift + cic_benign_threshold`
  - `clip_high_shift + cse_benign_threshold`

Artifact Part 5:
- `artifacts/v5_domainfix/v5_run_manifest.json`
- `artifacts/v5_domainfix/dos_hulk_deep_dive.json`
- `artifacts/v5_domainfix/dos_hulk_residual_features.csv`
- `results/v5_domainfix/dos_hulk_error_distributions.png`
- `artifacts/v5_domainfix/high_shift_feature_profile.json`
- `artifacts/v5_domainfix/strategy_inputs_preview.csv`
- `artifacts/v5_domainfix/score_profile_v5.json`
- `artifacts/v5_domainfix/thresholds_cic_benign_v5.json`
- `artifacts/v5_domainfix/thresholds_cse_benign_v5.json`
- `artifacts/v5_domainfix/threshold_transfer_drift_report.json`
- `artifacts/v5_domainfix/part5_eval_matrix.csv`
- `artifacts/v5_domainfix/part5_best_config.json`
- `artifacts/v5_domainfix/part5_gate_report.json`
- `results/v5_domainfix/part5_summary.md`

## 15. Capaian Aktual Full Run Part 5 (Validated)

Sumber validasi:
- `mscnn-nids/histories-read-only/notebook-huge-fix-part-5.ipynb`
- Output `[Part5] Completed` pada section eksekusi Part 5

Best configuration yang terpilih:
- `clip_high_shift + cse_benign_threshold`
- `CSE AUC raw/inverted: 0.6223 / 0.3777`
- `CSE FPR: 0.0003`
- `DoS Hulk DR: 0.0062`
- `Outcome: advance_to_part6_or_phase3`

### 15.1 Delta Utama vs Baseline Lama (Part 4-style baseline cell)

| Metric | Baseline Lama | Part 5 Best | Delta |
|---|---:|---:|---:|
| CSE ROC-AUC (raw) | 0.2770 | 0.6223 | +0.3453 |
| CSE FPR | 0.6586 | 0.0003 | -0.6583 |
| CIC ROC-AUC (raw) | 0.8633 | 0.8633 | +0.0000 |
| Direction issue (CSE) | True | False | Improved |

Interpretasi:
- Perbaikan CSE signifikan, membuktikan masalah utama ada pada domain calibration/scoring transfer.
- Guard CIC tetap terjaga (tidak ada penurunan ROC-AUC pada hasil best).
- Trade-off belum selesai untuk DoS Hulk (DR masih sangat rendah).

### 15.2 Status Gate Part 5 (berdasarkan best config)

- `G1 (CSE AUC inverted >= 0.72)`: `False`
- `G2 (CSE AUC raw > 0.60)`: `True`
- `G3 (CSE FPR < 0.20)`: `True`
- `G4 (DoS Hulk DR > 0.15)`: `False`
- `G5 (CIC AUC >= 0.8333)`: `True`

Keputusan eksekusi:
- Outcome final tetap `advance_to_part6_or_phase3` karena policy Part 5 memprioritaskan kelulusan `G2`.

### 15.3 Catatan Inkonsistensi Notebook (Wajib Diketahui)

Pada notebook run yang sama ada inkonsistensi antar-section:
- Section Part 5 (`[Part5] Completed`) sudah menunjukkan metrik best terbaru.
- Beberapa cell analisis generalization lama (setelahnya) masih mencetak metrik baseline lama (`CSE ROC-AUC 0.2770`, `FPR 0.6586`, verdict `CIC-SPECIFIC`).

Artinya:
- Untuk keputusan Part 5, gunakan artifact/output Part 5 (`part5_eval_matrix.csv`, `part5_best_config.json`, `part5_gate_report.json`) sebagai source of truth.
- Cell analisis generalization lama perlu direfresh agar membaca hasil best Part 5, bukan variabel baseline sebelumnya.

## 16. Part 6 Implemented: `v6_report_threshold` (Report Sync + Threshold Trade-off)

Tujuan implementasi Part 6:
- Sinkronkan section generalization/report agar tidak lagi membaca baseline lama.
- Optimasi operating threshold CSE dengan objective:
  - hard constraint `FPR <= 0.05`
  - maximize `Recall`
  - tie-break `F1`
- Tetap tanpa retrain model.

Perubahan utama di notebook:
- Config baru Part 6:
  - `PART6_VERSION = "v6_report_threshold"`
  - `PART6_ARTIFACTS_DIR`, `PART6_RESULTS_DIR`
  - `RUN_PART6_REPORT_THRESHOLD = True`
  - `PART6_FPR_CAP = 0.05`
  - `PART6_MIN_RECALL_FLOOR = 0.40`
  - `PART6_CALIB_RATIO = 0.20`
  - `PART6_CALIB_SPLIT_MODE = "stratified_multiclass_label"`
  - `PART6_THRESHOLD_GRID_POINTS = 300`
  - `PART6_STRICT_SYNC = True`
- Section baru `14.4-14.7`:
  - bootstrap + validasi source artifacts Part 5
  - recompute score CIC/CSE dengan strategy terbaik Part 5
  - stratified split CSE by multi-class label (dengan fallback rare-class terkontrol)
  - threshold sweep + selection policy + conservative alert
  - holdout evaluation + CIC compatibility check + gate report
- Refactor section `14.1-14.3` dan `15`:
  - sekarang wajib baca output Part 6 (strict sync)
  - tidak lagi menggunakan variabel baseline legacy untuk verdict final.

Artifact Part 6 yang dihasilkan:
- `artifacts/v6_report_threshold/part6_run_manifest.json`
- `artifacts/v6_report_threshold/score_distribution_part6.json`
- `artifacts/v6_report_threshold/part6_split_label_distribution.csv`
- `artifacts/v6_report_threshold/part6_threshold_sweep_calib.csv`
- `artifacts/v6_report_threshold/part6_best_threshold.json`
- `artifacts/v6_report_threshold/part6_cse_holdout_metrics.json`
- `artifacts/v6_report_threshold/part6_cic_compat_metrics.json`
- `artifacts/v6_report_threshold/part6_hulk_metrics.json`
- `artifacts/v6_report_threshold/part6_gate_report.json`
- `artifacts/v6_report_threshold/generalization_verdict_part6.json`
- `results/v6_report_threshold/part6_pr_curve_points.csv`
- `results/v6_report_threshold/part6_roc_curve_points.csv`
- `results/v6_report_threshold/generalization_comparison_part6.csv`
- `results/v6_report_threshold/final_report_part6.md`

Catatan:
- Hasil numerik final Part 6 akan terisi setelah notebook dijalankan sampai section Part 6 selesai.
- Source of truth untuk verdict final harus mengikuti artifact `v6_report_threshold`.
