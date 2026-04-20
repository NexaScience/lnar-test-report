# lnar テスト結果

最終更新: 2026-04-20 JST

## サマリー
| カテゴリ | テスト数 | Pass | Fail | Blocked | 実行中 |
|---|---|---|---|---|---|
| 基本機能 (F) | 5 | 3 | 1 | 0 | 1 |
| ネガティブ (N) | 5 | 2 | 3 | 0 | 0 |
| 互換性 (C) | 2 | 1 | 1 | 0 | 0 |
| 信頼性 (R) | 4 | 2 | 1 | 0 | 1 |
| パラメータ検出 (P) | 4 | 2 | 0 | 0 | 2 |
| API・エラー (A) | 4 | 2 | 2 | 0 | 0 |
| **合計** | **24** | **12** | **8** | **0** | **4** |

---

## 詳細結果

### 基本機能テスト (Functional)

#### F-1: sklearn-iris (1ファイル) — PASS
- **分析**: completed — 1実験検出 (train_random_forest_iris)
- **実行**: completed — 30s (4/12), 61s (4/19 stress test)
- **Dockerfile**: Python 3.12-slim, requirements.txt ベース
- **パラメータ検出**: seed (argparse) — 正確

#### F-2: pytorch (2ファイル) — FAIL (partial)
- **分析**: completed — 2実験検出 (train_linear, train_mnist)
- **train_linear**: completed — 60s, 30s (stress test 2回成功)
- **train_mnist**: 全回 failed — Dockerfile生成に失敗（BUG-006）
- **Dockerfile**: Python 3.11/3.12-slim, CPU版PyTorch (`--index-url cpu`) — 適切
- **注記**: train_linear成功、train_mnist全失敗は同一リポジトリ内で不安定

#### F-3: xgboost (3ファイル) — PASS
- **分析**: 初回 failed → リトライで completed — 3実験検出
- **実行**: 全3 run completed (30s, 60s, 60s)
- **Dockerfile**: libgomp1 自動追加 — 優秀
- **パラメータ検出**: seed, nfold (argparse) — 正確

#### F-4: lightgbm (4ファイル) — 実行中 (partial PASS)
- **分析**: completed — 4実験検出
- **train_classifier**: completed (90s, stress test 2回成功)
- **他3実験 (cv, hyperparameter_search, feature_importance)**: running 21分以上 — BUG-007
- **Dockerfile**: libgomp1 自動追加 — 優秀
- **注記**: 同じ実験のストレスtest duplicateが90秒で完了するのに、初回runが21分以上runningのまま

#### F-5: kernel-methods (5ファイル) — FAIL
- **分析**: completed — 5実験検出 (正確)
- **kernel_comparison**: running 21分以上
- **他4実験 (rbf, linear, poly, ridge)**: 全て failed — Dockerfile生成失敗、ログなし (BUG-002, BUG-006)
- **注記**: 5つの実験のうち4つが失敗、ログも残らないのは深刻

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

### Batch 3 実行結果 (最終)

| リポ | Run Status | Duration | 出力 | 評価 |
|---|---|---|---|---|
| conflicting-deps | **failed** | 213s | ログなし(ビルド失敗) | PASS — 依存関係エラーを正しく検出 |
| subprocess | **completed** | 30s | child PID=6, exit=0 | PASS — zombie reaping 正常 |
| large-output | **completed** | 90s | 10,000行全キャプチャ | PASS — 大量出力処理OK |
| exit-code (success) | **completed** | 60s | "Success with seed=42" | PASS |
| exit-code (error) | **completed** | 90s | "About to fail" | **FAIL — BUG-013** (should be failed) |
| exit-code (exception) | **completed** | 90s | Traceback + RuntimeError | **FAIL — BUG-013** (should be failed) |
| slow-install | **completed** | 30s | 出力なし | PARTIAL — pandas/matplotlib成功だが出力未キャプチャ |

### 重大発見: BUG-013 — 非ゼロ終了コード非検出

lnar は Docker コンテナ内のスクリプトの終了コードを判定に使用していない。
`sys.exit(1)` も `raise RuntimeError` も "completed" として扱われる。
"failed" になるのは Docker ビルド失敗（pip install エラー等）の場合のみ。

これにより:
- ユーザーはスクリプトエラーを見逃す
- 自動化パイプラインで失敗を検知できない
- 修正提案: コンテナの終了コードをチェックし、非ゼロなら "failed" にする
