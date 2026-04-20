# lnar バグ・問題点一覧

最終更新: 2026-04-20

## 重大度定義
- **Critical**: サービス提供に支障
- **High**: 主要機能に影響
- **Medium**: 回避策あり
- **Low**: 軽微
- **Info**: 改善提案

---

## BUG-001: restart_analysis の S3 DeleteObject 権限エラー
- **重大度**: Medium
- **再現手順**: `restart_analysis` を呼び出す
- **エラー**: `AccessDenied: s3:DeleteObject on lnar-artifacts-prod`
- **原因**: `lnar-ecs-task-role` に `s3:DeleteObject` 権限がない
- **影響**: 失敗した分析結果をクリアして再実行できない
- **回避策**: `start_analysis` の再呼出しで新規分析として実行可能（冪等）
- **発見日**: 2026-04-19

## BUG-002: Run失敗時にstdout/stderrがS3に保存されない
- **重大度**: High
- **再現手順**: Dockerfile生成に失敗する run を開始 → failed → S3で NoSuchKey
- **影響**: 失敗原因の調査が不可能。デバッグ情報が完全に失われる
- **発見リポジトリ**: sklearn, pytorch, kernel-methods
- **発見日**: 2026-04-19

## BUG-003: 不完全な experiment dict で start_run → Pydantic ValidationError
- **重大度**: High
- **再現手順**: get_analysis から取得せず、手動構築した experiment dict で start_run を呼出し → Pydantic ValidationError（description, inputs 等が欠如）
- **影響**: run は `failed` になるが、Dockerfile も生成されずログも残らない
- **回避策**: 必ず `get_analysis` の結果から experiment dict を取得して使用する
- **備考**: MCP tool の description に「experiment dict は get_analysis から取得すべき」と明記されているが、バリデーションエラー時のフィードバックが不親切
- **発見日**: 2026-04-19

## BUG-004: xgboost 分析が初回 failed
- **重大度**: Low
- **再現手順**: test-xgboost-regression-example の start_analysis → "Analysis failed"（原因不明）
- **影響**: リトライで成功するため実用上は軽微
- **備考**: エラー詳細が "Analysis failed" のみで、根本原因が不明
- **発見日**: 2026-04-19

## BUG-005: get_run_logs の Redis Streams 保持期限後にログ消失
- **重大度**: Info
- **再現手順**: 完了から時間が経った run の get_run_logs → 空配列
- **影響**: リアルタイムログの事後参照が不可能
- **回避策**: presigned URL (stdout_url/stderr_url) でS3から取得（成功した run のみ）
- **発見日**: 2026-04-19

## BUG-006: 同一リポジトリ内で run 成功/失敗が不安定に混在
- **重大度**: High
- **再現手順**: kernel-methods の5実験を同時 start_run → 4/5 が failed (dockerfile null, ログなし), 1/5 が running
- **影響**: 同じ分析結果から開始した run でも一部だけ Dockerfile 生成に失敗する非決定的挙動
- **備考**: 同時に多数の run を開始した場合にDockerfile生成の競合が発生している可能性
- **関連**: BUG-002（ログなし）
- **発見リポジトリ**: kernel-methods (4/5 failed), pytorch (train_mnist全回failed)
- **発見日**: 2026-04-19

## BUG-007: 初回 run が異常に長時間 running のまま
- **重大度**: Medium
- **再現手順**: lightgbm の4実験を同時 start_run → 21分以上 running のまま。同じ実験のストレスtest duplicate は90秒で完了
- **影響**: 初回Dockerイメージビルドが並列 run 間で競合し、一部 run が長時間ブロックされる
- **備考**: ストレスtest duplicate が先に完了 → Dockerイメージがキャッシュ → 初回 run はキャッシュ前にビルド開始してスタック？
- **発見リポジトリ**: lightgbm (初回4 run vs ストレスtest 2 run)
- **発見日**: 2026-04-19

## BUG-008: エッジケースリポジトリの分析で一律 "failed" を返す
- **重大度**: Medium
- **再現手順**: 有効なPythonスクリプトを含むリポジトリ (multi-lang, long-running) を分析 → failed
- **期待動作**: 有効なPythonスクリプトがあれば分析は completed になり実験を検出すべき
- **影響**: 一部の正常なリポジトリが分析できない
- **詳細**:
  - **empty-repo**: failed は妥当（Pythonファイルなし）
  - **syntax-error**: failed は妥当（構文エラー）
  - **no-deps**: failed — 依存関係がなくても分析は完了すべき
  - **multi-lang**: failed — Python + R の混在リポで分析失敗
  - **long-running**: failed — 有効なPythonスクリプト（signal.alarm使用）なのに失敗
- **発見日**: 2026-04-19

## BUG-009: Non-ML Python スクリプトが分析に失敗する
- **重大度**: Critical
- **再現手順**: ML系ライブラリ (sklearn, torch, xgboost等) をインポートしない純粋なPythonスクリプトのリポジトリを分析
- **影響**: Batch 2の全8リポジトリ (env-vars, file-output, multi-module, network, memory, unicode, stdin, non-ml) が分析失敗
- **詳細**: argparse + random/sys/csv を使用するがsklearn/torch/xgboostを使用しないスクリプトは分析に失敗する。Batch 3リポジトリ (lnarが認識するargparseパターンを持つ) はリトライで成功
- **結論**: lnarの分析エンジンはML系のインポートパターンを前提としている
- **発見日**: 2026-04-20

## BUG-010: XGBoost 3.2.0 API破壊的変更への未対応
- **重大度**: Low
- **再現手順**: xgb.cv() を使用するスクリプトを実行 → XGBoost 3.2.0ではDataFrameではなくlistを返す
- **影響**: xgboost CVスクリプトがAPIエラーで失敗
- **備考**: スクリプト側のエラーでありlnar自体のバグではないが、lnarがバージョン固定で対応できる可能性あり
- **発見日**: 2026-04-20

## BUG-011: sklearn 1.8.0 API破壊的変更
- **重大度**: Low
- **再現手順**: mean_squared_error に squared=False を渡すスクリプトを実行 → sklearn 1.8.0で squared パラメータが削除済み
- **影響**: kernel ridge スクリプトがAPIエラーで失敗
- **備考**: スクリプト側のエラーでありlnar自体のバグではない
- **発見日**: 2026-04-20

## BUG-012: 分析の間欠的失敗率が高い
- **重大度**: Medium
- **再現手順**: 新規リポジトリの分析を初回実行
- **影響**: Batch 3の全5リポジトリで2-3回の分析試行が必要だった。Round 1のデータと合わせると、初回分析の約40%が失敗する
- **備考**: リトライで成功するため、一過性のバックエンド問題を示唆
- **発見日**: 2026-04-20

### BUG-013: lnar は非ゼロ終了コードをfailedとして検出しない
- **重大度**: Critical
- **再現手順**:
  1. `sys.exit(1)` を含むスクリプトを実行 → run status = "completed"
  2. 未処理の `RuntimeError` を含むスクリプトを実行 → run status = "completed"
- **期待動作**: 非ゼロ終了コード → status = "failed"
- **実際動作**: Docker コンテナ内のスクリプトがエラー終了しても "completed" と表示
- **影響**: ユーザーはスクリプトが成功したと誤認する。失敗検出は Docker ビルド失敗のみ
- **テスト結果**:
  - `sys.exit(0)` → completed (正しい)
  - `sys.exit(1)` → completed (誤り、should be failed)
  - `raise RuntimeError` → completed (誤り、should be failed)
  - pip install 失敗 → failed (正しい、ビルドフェーズ)
- **追加観察**: Traceback は stderr ではなく stdout に出力された
- **発見リポジトリ**: test-lnar-edge-exit-code
- **発見日**: 2026-04-20

---

## 統計

| 重大度 | 件数 |
|---|---|
| Critical | 2 (BUG-009, BUG-013) |
| High | 3 (BUG-002, BUG-003, BUG-006) |
| Medium | 4 (BUG-001, BUG-007, BUG-008, BUG-012) |
| Low | 2 (BUG-010, BUG-011) |
| Info | 2 (BUG-004, BUG-005) |
| **合計** | **13** |
