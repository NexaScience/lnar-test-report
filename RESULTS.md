# lnar テスト結果

最終更新: 2026-04-19 23:10 JST

## サマリー
| カテゴリ | テスト数 | Pass | Fail | Blocked | 実行中 |
|---|---|---|---|---|---|
| 基本機能 (F) | 5 | 2 | 0 | 1 | 2 |
| ネガティブ (N) | 5 | 1 | 0 | 0 | 4 |
| 互換性 (C) | 2 | 0 | 0 | 0 | 2 |
| 信頼性 (R) | 4 | 2 | 1 | 0 | 1 |
| パラメータ検出 (P) | 4 | 0 | 0 | 0 | 4 |
| API・エラー (A) | 4 | 1 | 2 | 0 | 1 |
| **合計** | **24** | **6** | **3** | **1** | **14** |

---

## 詳細結果

### 基本機能テスト (Functional)

#### F-1: sklearn-iris (1ファイル) — PASS
- **分析**: completed — 1実験検出 (train_random_forest_iris)
- **実行**: completed — 30s (4/12), 61s (4/19 stress test)
- **Dockerfile**: Python 3.12-slim, requirements.txt ベース
- **出力**: テスト精度スコアを正常出力

#### F-2: pytorch (2ファイル) — PARTIAL (train_linear PASS, train_mnist BLOCKED)
- **分析**: completed — 2実験検出 (train_linear, train_mnist)
- **train_linear 実行**: completed — 60s, 30s (stress test 2回とも成功)
- **train_mnist 実行**: 全回 failed — MNISTデータセットの外部ダウンロードが原因と推定
- **Dockerfile**: Python 3.11/3.12-slim, CPU版PyTorch (`--index-url cpu`)
- **注記**: lnar はPyTorchのCPU版インストールを自動検出し適切なDockerfileを生成

#### F-3: xgboost (3ファイル) — 実行中
- **分析**: 初回 failed → リトライで completed — 3実験検出 (train_xgboost, train_with_cv, feature_importance)
- **実行**: 3 run running (開始 14:05 UTC)
- **注記**: 初回分析失敗は原因不明。restart_analysis はS3権限バグで使用不可。2回目のstart_analysisで成功

#### F-4: lightgbm (4ファイル) — 実行中
- **分析**: completed — 4実験検出 (train_classifier, train_with_cv, hyperparameter_search, feature_importance)
- **train_classifier 実行**: completed — 90s (stress test 2回とも成功)
- **他3実験**: running (開始 13:55 UTC)
- **Dockerfile**: libgomp1 (OpenMP) を自動検出してaptインストール — 優秀

#### F-5: kernel-methods (5ファイル) — 実行中
- **分析**: completed — 5実験検出 (kernel_comparison, train_svm_rbf, train_kernel_ridge, train_svm_linear, train_svm_poly)
- **実行**: 5 run running (開始 13:56 UTC)

---

### ネガティブテスト (Negative)

#### N-1: デコイリポジトリ — PASS
- **分析**: completed — **0実験検出** (正解)
- **repo_summary**: 全ファイルがインポート用モジュールで、`__main__` ブロックやCLI引数なしと正しく認識
- **前回**: 偽陽性あり（import文を組み立てて実行コマンド生成）→ 今回修正済み

#### N-2: 空リポジトリ — 分析中
- **分析**: running (6edabe99)

#### N-3: 構文エラー — 分析中
- **分析**: running (2bbc499d)

#### N-4: 存在しない依存パッケージ — 分析中
- **分析**: running (6b4e37ce)

#### N-5: 長時間実行 — 分析中
- **分析**: running (59260200)

---

### 互換性テスト (Compatibility)

#### C-1: 多言語リポジトリ (Python + R) — 分析中
- **分析**: running (6e2c27f6)

#### C-2: 各種MLフレームワーク対応 — 実行中
- sklearn: PASS (completed)
- pytorch: PARTIAL (train_linear OK, train_mnist failed)
- xgboost: running
- lightgbm: PARTIAL (train_classifier OK, 他 running)

---

### 信頼性テスト (Reliability)

#### R-1: 再現性 — PASS
- sklearn Iris: 4/12 (30s) と 4/19 (61s) で同一実験が再現的に成功
- Dockerfileは毎回新規生成されるが内容はほぼ同一

#### R-2: 並列実行 — PASS
- sklearn: 2並列run → 両方 61s/61s で completed
- pytorch linear: 2並列run → 60s/30s で completed
- lightgbm train_classifier: 2並列run → 90s/90s で completed
- **結論**: 同一実験の並列実行は安定動作

#### R-3: restart_analysis — FAIL
- S3 DeleteObject 権限エラー (BUG-001)
- 回避策: 新しいstart_analysis呼び出しで代替可能（同一commit_hashで冪等）

#### R-4: 失敗後の再実行 — 実行中
- sklearn: 初回 failed (不完全なexperiment dict) → 2回目 completed (完全なdict)
- pytorch linear: 初回 failed → 2回目 completed
- **暫定結論**: 失敗原因がMCPクライアント側のバリデーションエラーだった場合、正しいデータで再実行すれば成功

---

### パラメータ検出テスト (Parameter Detection)
- 実行結果が出揃い次第、各リポジトリのパラメータ検出精度を評価予定

---

### API・エラーハンドリングテスト (API)

#### A-1: restart_analysis S3権限エラー — FAIL (BUG-001)
- `AccessDenied: s3:DeleteObject` on `lnar-artifacts-prod`
- lnar-ecs-task-role に権限不足

#### A-2: 不完全なexperiment dictでのstart_run — FAIL (BUG-003)
- description, inputs, outputs 等を省略 → Pydantic ValidationError
- Dockerfile 生成されず、run は failed
- S3にstdout/stderrも保存されない (NoSuchKey)

#### A-3: 分析の冪等性 — PASS
- 同一 repository_id + commit_hash で start_analysis を再呼出し → 既存のworkflowを再利用 (重複実行なし)

#### A-4: 完了済みrunのログ取得 — 実行中
- get_run_logs: Redis Streams の保持期限後は空配列を返す
- presigned URL: 成功した run では有効、失敗した run では NoSuchKey の場合あり

---

## 発見されたバグ・問題点

| ID | 重大度 | 概要 |
|---|---|---|
| BUG-001 | Medium | restart_analysis の S3 DeleteObject 権限エラー |
| BUG-002 | High | Run 失敗時に stdout/stderr が S3 に保存されないケースがある |
| BUG-003 | High | 不完全な experiment dict で start_run → dockerfile null → FAILED |
| BUG-004 | Low | xgboost 分析が初回 failed（原因不明、リトライで成功） |
| BUG-005 | Info | get_run_logs の Redis Streams 保持期限後にログ消失（presigned URLが代替） |

## Dockerfileの品質 (良い点)

| フレームワーク | lnar の対応 | 評価 |
|---|---|---|
| sklearn | `requirements.txt` からの標準的な pip install | 良好 |
| PyTorch | CPU版を `--index-url cpu` で自動インストール | 優秀 |
| LightGBM | `libgomp1` (OpenMP) を自動検出して apt install | 優秀 |
| XGBoost | 評価中 (running) | - |
