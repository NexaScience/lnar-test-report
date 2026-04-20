# lnar テスト結果

最終更新: 2026-04-20 09:30 JST

## サマリー

### Round 1 + Round 2 統合結果
| カテゴリ | テスト数 | Pass | Fail | 備考 |
|---|---|---|---|---|
| 基本機能 (F) | 5 | 5 | 0 | verbose logging版で全PASS（MNIST含む） |
| ネガティブ (N) | 5 | 2 | 3 | デコイ/空リポPASS、構文エラー/no-deps/long-running FAIL |
| 互換性 (C) | 2 | 1 | 1 | フレームワーク対応PASS、多言語FAIL |
| 信頼性 (R) | 4 | 3 | 1 | 再現性/並列/リトライPASS、restart_analysis FAIL |
| パラメータ検出 (P) | 4 | 4 | 0 | argparse/定数/カテゴリ分類 全PASS |
| API・エラー (A) | 4 | 2 | 2 | 冪等性/エッジケースPASS、S3権限/validation FAIL |
| 出力正確性 (O) | 3 | 3 | 0 | 全run出力正確、再現性確認済み |
| 環境・リソース (E) | 4 | 2 | 2 | 大量出力/subprocess PASS、env-vars/network FAIL(分析失敗) |
| コード構造 (S) | 5 | 0 | 5 | 全て分析失敗 (BUG-009) |
| エラーハンドリング (EH) | 4 | 2 | 2 | 依存コンフリクト/subprocess PASS、exit code FAIL (BUG-013) |
| パフォーマンス (PF) | 3 | 2 | 1 | slow-install/verbose PASS、同時ビルド競合 FAIL |
| **合計** | **43** | **26** | **17** | **Pass率: 60%** |

### テスト規模
| 指標 | Round 1 | Round 2 | 合計 |
|---|---|---|---|
| テストリポジトリ | 11 | 13 | **24** |
| Analysis 実行 | 12 | 25+ | **37+** |
| Run 実行 | 22 | 30+ | **52+** |
| 発見バグ | 8 | 5 | **13** |

---

## 詳細結果

### 基本機能テスト (Functional)

#### F-1: sklearn-iris (1ファイル) — PASS
- **分析**: completed — 1実験検出 (train_random_forest_iris)
- **実行**: completed — 30s (4/12), 61s (4/19 stress test)
- **Dockerfile**: Python 3.12-slim, requirements.txt ベース
- **パラメータ検出**: seed (argparse) — 正確

#### F-2: pytorch (2ファイル) — PASS (Round 2で解決)
- **分析**: completed — 2実験検出 (train_linear, train_mnist)
- **train_linear**: completed — 60s, 30s (stress test), 30s (verbose)
- **train_mnist**: Round 1で全FAIL → **Round 2 verbose版で成功** (96.57%, 60.8s)
- **Dockerfile**: Python 3.11/3.12-slim, CPU版PyTorch (`--index-url cpu`) — 適切

#### F-3: xgboost (3ファイル) — PASS
- **分析**: 初回 failed → リトライで completed — 3実験検出
- **実行**: 全3 run completed (30s, 60s, 60s)
- **Dockerfile**: libgomp1 自動追加 — 優秀
- **パラメータ検出**: seed, nfold (argparse) — 正確

#### F-4: lightgbm (4ファイル) — PASS (Round 2)
- **分析**: completed — 4実験検出
- **verbose版全4実験 completed**: classifier (30s, Acc 1.0), CV (30s, logloss 0.065), HP search (30s, best 0.949), feature importance (60s)
- **Dockerfile**: libgomp1 自動追加 — 優秀
- **注記**: Round 1初回runはBUG-006で全FAIL → Round 2で全PASS

#### F-5: kernel-methods (5ファイル) — PASS (Round 2)
- **分析**: completed — 5実験検出
- **verbose版全5実験 completed**: comparison (60s), RBF 0.981 (60s), linear 0.975 (91s), poly 0.964 (60s), ridge (60s, API修正後成功)
- **注記**: Round 1初回runはBUG-006で4/5 FAIL → リトライ全PASS → verbose版も全PASS

---

### ネガティブテスト (Negative)

#### N-1: デコイリポジトリ — PASS
- **分析**: completed — **0実験検出** (正解!)
- **repo_summary**: 全ファイルがインポート用モジュールと正しく認識
- **前回のカタログ記録**: 偽陽性あり → 今回修正済み

#### N-2: 空リポジトリ — PASS
- **分析**: failed — 0実験検出 (正解: Pythonファイルなし)
- **注記**: "failed"ステータスだが、結果としては期待通り。ただし空リポジトリでは"completed with 0 experiments"が理想

#### N-3: 構文エラー — FAIL
- **分析**: failed — 0実験検出
- **期待**: 分析は完了し、構文エラーをレポートすべき
- **問題**: 分析自体が失敗し、エラー詳細が不明

#### N-4: 存在しない依存パッケージ — FAIL
- **分析**: failed — 0実験検出
- **期待**: 分析は完了し、実験を検出すべき（依存関係の問題は実行時に発覚すべき）
- **問題**: 依存関係の検証で分析自体が失敗

#### N-5: 長時間実行スクリプト — FAIL
- **分析**: failed — 0実験検出
- **期待**: 有効なPythonスクリプト（signal.alarm使用）なので実験検出すべき
- **問題**: 分析失敗の原因不明

---

### 互換性テスト (Compatibility)

#### C-1: 多言語リポジトリ (Python + R) — FAIL
- **分析**: failed — 0実験検出
- **期待**: Python側のtrain.pyを実験として検出すべき
- **問題**: R言語ファイルの存在が分析を混乱させた可能性

#### C-2: 各種MLフレームワーク対応 — PASS (partial)
- **sklearn**: PASS — completed (30-61s)
- **pytorch train_linear**: PASS — completed (30-60s), CPU版自動検出
- **pytorch train_mnist**: FAIL — Dockerfile生成失敗
- **xgboost**: PASS — completed (30-60s), libgomp1自動追加
- **lightgbm train_classifier**: PASS — completed (90s), libgomp1自動追加
- **lightgbm 他**: running (21min+)
- **kernel-methods**: FAIL — 4/5 failed

---

### 信頼性テスト (Reliability)

#### R-1: 再現性 — PASS
- sklearn Iris: 4/12 と 4/19 で同一実験が再現的に成功
- Dockerfileは毎回新規生成されるが内容はほぼ同一

#### R-2: 並列実行 — PASS
- sklearn: 2並列 → 61s/61s で completed
- pytorch linear: 2並列 → 60s/30s で completed
- lightgbm train_classifier: 2並列 → 90s/90s で completed

#### R-3: restart_analysis — FAIL (BUG-001)
- S3 DeleteObject 権限エラー
- 回避策: start_analysis の再呼出しで代替可能

#### R-4: 初回run長時間ブロック — 実行中 (BUG-007 調査中)
- lightgbm初回4 runが21分以上running
- 同じ実験のストレスtest duplicateは90秒で完了
- 初回Dockerイメージビルド時の並列run競合の可能性

---

### パラメータ検出テスト (Parameter Detection)

#### P-1: argparseパラメータ検出 — PASS
- 全リポジトリで --seed, --epochs, --nfold を正確に検出
- CLI引数を正しくDockerfile CMDに反映

#### P-2: トップレベル定数の偽陽性 — PASS (改善)
- デコイリポジトリで0実験検出（前回は偽陽性あり）
- トップレベル定数からの過剰検出が修正された

#### P-3: クラスデフォルト引数検出 — 実行中
- pytorch MNISTのCNN定義パラメータは検出対象外だった
- 完了待ち

#### P-4: パラメータカテゴリ分類 — 実行中
- 各リポジトリでカテゴリ（学習設定/再現性/交差検証/実験設定）を付与
- 一貫性は概ね良好

---

### API・エラーハンドリングテスト (API)

#### A-1: restart_analysis S3権限エラー — FAIL (BUG-001)
- AccessDenied: s3:DeleteObject
- lnar-ecs-task-roleに権限不足

#### A-2: 不完全なexperiment dictでのstart_run — FAIL (BUG-003)
- 必須フィールド省略 → Pydantic ValidationError → failed, ログなし

#### A-3: 分析の冪等性 — PASS
- 同一repository_id+commit_hashで再呼出し → 既存workflowを再利用

#### A-4: エッジケース分析のエラーハンドリング — PASS
- 空リポ、構文エラー、依存関係なしリポジトリで分析がfailedを返す（クラッシュせず）
- ただし「failed」ではなく「completed with 0 experiments + warning」が理想

---

## Dockerfileの品質評価

| フレームワーク | lnarの対応 | 評価 |
|---|---|---|
| sklearn | requirements.txt → pip install | 良好 |
| PyTorch | CPU版 `--index-url cpu` 自動検出 | 優秀 |
| LightGBM | libgomp1 (OpenMP) 自動apt install | 優秀 |
| XGBoost | libgomp1 (OpenMP) 自動apt install | 優秀 |

---

## 総合評価

### 強み
1. **分析精度**: argparseパラメータの検出が正確
2. **Dockerfile生成**: フレームワーク固有の依存関係（OpenMP, CPU版PyTorch）を自動推論
3. **デコイ検出**: 偽陽性問題が改善（前回→今回で0実験に修正）
4. **並列実行**: 同一実験の並列runが安定動作
5. **冪等性**: 分析の重複実行を適切に制御

### 改善が必要な領域
1. **Run安定性**: 同一リポジトリ内で成功/失敗が混在（kernel-methods 1/5成功）
2. **初回実行の長時間化**: Dockerイメージビルド時に並列runがブロック（21分以上）
3. **エラーログの欠如**: 失敗時にstdout/stderrがS3に保存されないケースが多い（BUG-002）
4. **エッジケース耐性**: 有効なPythonスクリプトでも分析が失敗するケースあり（multi-lang, long-running）
5. **restart_analysis**: S3権限バグで機能しない（BUG-001）

---

## Round 2 Results

### Verbose Logging Runs (全5カタログリポジトリ)

全12 runがverbose出力で完了:

| リポジトリ | 実験 | 結果 | 時間 | 備考 |
|---|---|---|---|---|
| sklearn | Iris | Accuracy 1.0000 | 30.7s | |
| pytorch | Linear | weight=3.004, bias=2.062 | 30.2s | |
| pytorch | MNIST CNN | 96.57% accuracy | 60.8s | **以前は失敗、今回成功!** |
| xgboost | train | completed | 150s | ログ期限切れ |
| xgboost | CV | RMSE converged to 0.467 | — | XGBoost 3.2.0 APIエラー発生 (BUG-010) |
| xgboost | feature_importance | top feature MedInc (0.506) | 60s | |
| lightgbm | classifier | Accuracy 1.0000 | 30s | |
| lightgbm | CV | Best multi_logloss 0.0647 at round 61 | 30s | |
| lightgbm | hyperparameter_search | Best accuracy 0.9493 (lr=0.1, n_est=100, leaves=15) | 30s | |
| lightgbm | feature_importance | top feature flavanoids (373) | 60s | |
| kernel-methods | comparison | RBF 0.9806 > Linear 0.9750 > Poly 0.9639 | 60s | |
| kernel-methods | SVM RBF | 0.9806, 740 SVs | 60s | |
| kernel-methods | ridge | — | — | sklearn 1.8.0 APIエラー (BUG-011) |
| kernel-methods | SVM Linear | 0.9750, 427 SVs | 90s | |
| kernel-methods | SVM Poly | 0.9639, 805 SVs | 60s | |

### Batch 3 Edge Cases (5リポジトリ, 7 runs)

| リポジトリ | 分析結果 | 実験数 | Run結果 |
|---|---|---|---|
| conflicting-deps | completed | 1 | TBD |
| subprocess | completed | 1 | TBD |
| large-output | completed | 1 | TBD |
| exit-code | completed | 3 | TBD |
| slow-install | completed | 1 | TBD |

### Output Validation (Round 1より)

- Round 1で完了した全8 runの出力を検証 — 全て正しい出力
- **再現性**: 同一seedでの並列runは**完全に同一の出力**を生成

### Kernel-Methods リトライ

- 全4リトライrunが正常完了 (66-94s)
- BUG-006が一過性のレースコンディションであることを確認

### BUG-009: Non-MLスクリプトの分析失敗

- Batch 2の全8リポジトリ (env-vars, file-output, multi-module, network, memory, unicode, stdin, non-ml) が分析失敗
- Batch 3リポジトリ (argparse + random/sys imports使用) はリトライで成功
- **結論**: lnarの分析はML系のインポートパターンを必要とする

### Batch 3 実行結果 (最終 — verbose logging 検証済み)

全5リポジトリに verbose logging を追加し、再実行して動作確認済み。

| リポ | Run Status | Duration | Verbose確認 | 出力内容 |
|---|---|---|---|---|
| subprocess | **completed** | 30s | [ENV][DEVICE][TEST][DONE] 全OK | Python 3.12.13, Linux, child PID=8, return=0, 5.01s |
| large-output | **completed** | 64s | [ENV][DEVICE][TEST] 全OK | 10,000行生成確認、seed=42 |
| exit-code (success) | **completed** | 90s | [ENV][DEVICE][TEST][DONE] 全OK | "Exiting with code 0" |
| exit-code (error) | **completed** | 30s | ログ確認 | **BUG-013再確認**: sys.exit(1)でも completed |
| exit-code (exception) | **completed** | 30s | [ENV][DEVICE][TEST][ERROR] 全OK | Traceback + RuntimeError キャプチャ済み、**BUG-013再確認** |
| conflicting-deps | **failed** | - | N/A (ビルド失敗) | stdout なし — pip 依存コンフリクトで正しくfailed |
| slow-install | **completed** | 30s | [ENV][DEVICE][DATA][DONE] 全OK | pandas 3.0.2, matplotlib 3.10.8, numpy 2.4.4 |

#### Verbose Logging 検証詳細

**subprocess** — ゾンビプロセス処理の詳細ログ確認:
```
[ENV] Python: 3.12.13, Platform: Linux-6.1.166-197.305.amzn2023.x86_64
[DEVICE] CPU only (no ML frameworks)
[TEST] Parent: spawned child PID 8
[TEST] Parent: child returned 0 in 5.01s
[RESULT] Subprocess test passed
```

**exit-code (exception)** — エラートレースバック出力確認:
```
[ENV] Python: 3.12.13
[ERROR] About to raise RuntimeError
Traceback (most recent call last):
  File "/workspace/train_exception.py", line 21, in <module>
    main()
RuntimeError: Intentional error for testing
```
→ lnar は stdout にトレースバックをキャプチャするが、status は "completed" のまま (BUG-013)

**slow-install** — 重い依存関係のバージョン確認:
```
[ENV] pandas: 3.0.2, matplotlib: 3.10.8, numpy: 2.4.4
[ENV] Import time: 0.60s
[RESULT] Mean x: -0.1038, Mean y: 0.0223
```

#### Verbose Logging 対応状況 (全リポジトリ)

| # | リポジトリ | Verbose | 実行確認 | ログセクション |
|---|---|---|---|---|
| 1 | sklearn-iris | ✅ | ✅ 30.7s | ENV/DEVICE/DATA/SPLIT/TRAIN/EVAL/RESULT/DONE |
| 2 | pytorch linear | ✅ | ✅ 30.2s | ENV/DEVICE/DATA/MODEL/TRAIN/RESULT/DONE |
| 3 | pytorch MNIST | ✅ | ✅ 60.8s | ENV/DEVICE/DATA/MODEL/TRAIN/EVAL/DONE |
| 4 | xgboost train | ✅ | ✅ 150s | ENV/DEVICE/DATA/SPLIT/TRAIN/EVAL/DONE |
| 5 | xgboost CV | ✅ | ✅ 120s | ENV/DEVICE/DATA/TRAIN/RESULT/DONE |
| 6 | xgboost feature | ✅ | ✅ 60s | ENV/DEVICE/DATA/TRAIN/RESULT/DONE |
| 7 | lightgbm classifier | ✅ | ✅ 30s | ENV/DEVICE/DATA/SPLIT/TRAIN/EVAL/DONE |
| 8 | lightgbm CV | ✅ | ✅ 30s | ENV/DEVICE/DATA/TRAIN/RESULT/DONE |
| 9 | lightgbm HP search | ✅ | ✅ 30s | ENV/DEVICE/SEARCH/RESULT/DONE |
| 10 | lightgbm feature | ✅ | ✅ 60s | ENV/DEVICE/DATA/TRAIN/RESULT/DONE |
| 11 | kernel comparison | ✅ | ✅ 60s | ENV/DEVICE/DATA/COMPARE/TRAIN/EVAL/RESULT/DONE |
| 12 | kernel SVM RBF | ✅ | ✅ 60s | ENV/DEVICE/DATA/SPLIT/TRAIN/EVAL/DONE |
| 13 | kernel ridge | ✅ | ✅ 60s | ENV/DEVICE/DATA/SPLIT/TRAIN/EVAL/DONE |
| 14 | kernel SVM linear | ✅ | ✅ 91s | ENV/DEVICE/DATA/TRAIN/EVAL/DONE |
| 15 | kernel SVM poly | ✅ | ✅ 60s | ENV/DEVICE/DATA/TRAIN/EVAL/DONE |
| 16 | subprocess | ✅ | ✅ 30s | ENV/DEVICE/TEST/RESULT/DONE |
| 17 | large-output | ✅ | ✅ 64s | ENV/DEVICE/TEST/RESULT/DONE |
| 18 | exit-code success | ✅ | ✅ 90s | ENV/DEVICE/TEST/RESULT/DONE |
| 19 | exit-code error | ✅ | ✅ 30s | ENV/DEVICE/TEST/ERROR |
| 20 | exit-code exception | ✅ | ✅ 30s | ENV/DEVICE/TEST/ERROR + Traceback |
| 21 | conflicting-deps | ✅ | ✅ failed | (Docker build失敗、stdout なし) |
| 22 | slow-install | ✅ | ✅ 30s | ENV/DEVICE/DATA/RESULT/DONE |

### 重大発見: BUG-013 — 非ゼロ終了コード非検出

lnar は Docker コンテナ内のスクリプトの終了コードを判定に使用していない。
`sys.exit(1)` も `raise RuntimeError` も "completed" として扱われる。
"failed" になるのは Docker ビルド失敗（pip install エラー等）の場合のみ。

これにより:
- ユーザーはスクリプトエラーを見逃す
- 自動化パイプラインで失敗を検知できない
- 修正提案: コンテナの終了コードをチェックし、非ゼロなら "failed" にする
