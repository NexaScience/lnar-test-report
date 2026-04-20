# lnar テスト計画

## 概要
lnar はGitHubリポジトリのコードをクラウド上で自動分析し、検出された実験をCPU上で実行するサービスである。
本テストでは、分析（Analysis）と実行（Run）の両機能を体系的にテストする。

## テストカテゴリ

### 1. 基本機能テスト (Functional)
- **F-1**: 単一実行ファイルリポジトリの分析・実行 (sklearn-iris, 1ファイル)
- **F-2**: 複数実行ファイルリポジトリの分析・実行 (pytorch, 2ファイル)
- **F-3**: 中規模リポジトリの分析・実行 (xgboost, 3ファイル)
- **F-4**: 大規模リポジトリの分析・実行 (lightgbm, 4ファイル)
- **F-5**: 最大規模リポジトリの分析・実行 (kernel-methods, 5ファイル)

### 2. ネガティブテスト (Negative)
- **N-1**: 実行ファイルなし（デコイ）リポジトリの分析 → 0実験検出が正解
- **N-2**: 空リポジトリ（Pythonファイルなし）の分析
- **N-3**: 構文エラーのあるPythonファイルの分析・実行
- **N-4**: 存在しない依存パッケージの実行
- **N-5**: 長時間実行スクリプト（タイムアウトテスト）

### 3. 互換性テスト (Compatibility)
- **C-1**: 多言語リポジトリ（Python + R）の分析
- **C-2**: 各種MLフレームワーク対応 (sklearn, pytorch, xgboost, lightgbm)

### 4. 信頼性テスト (Reliability)
- **R-1**: 同一実験の再実行で同一結果を得られるか（再現性）
- **R-2**: 同一実験の並列実行（同時複数run）
- **R-3**: 分析の再実行（restart_analysis）
- **R-4**: 失敗後の再実行

### 5. パラメータ検出テスト (Parameter Detection)
- **P-1**: argparseパラメータの検出精度
- **P-2**: トップレベル定数の検出（偽陽性チェック）
- **P-3**: クラスデフォルト引数の検出
- **P-4**: パラメータカテゴリ分類の正確性（HYPER/DATA/ARCH/INFRA/FIXED）

### 6. API・エラーハンドリングテスト (API)
- **A-1**: restart_analysis の S3権限エラー
- **A-2**: 不正なrepository_idでの操作
- **A-3**: 不正なcommit_hashでの操作
- **A-4**: 完了済みrunのログ取得

## テスト対象リポジトリ一覧

| ID | リポジトリ | テストカテゴリ | 期待実験数 |
|---|---|---|---|
| R1 | test-sklearn-iris-example-1-3 | F-1 | 1 |
| R2 | test-pytorch-simple-training-2-4 | F-2 | 2 |
| R3 | test-xgboost-regression-example-3-6 | F-3 | 3 |
| R4 | test-lightgbm-classification-example-4-10 | F-4 | 4 |
| R5 | test-kernel-methods-example-5-4 | F-5 | 5 |
| R6 | test-ml-training-toolkit-0-0 | N-1 | 0 |
| R7 | test-lnar-edge-empty-repo | N-2 | 0 |
| R8 | test-lnar-edge-syntax-error | N-3 | 1? |
| R9 | test-lnar-edge-no-deps | N-4 | 1? |
| R10 | test-lnar-edge-multi-lang | C-1 | 1 (Python only?) |
| R11 | test-lnar-edge-long-running | N-5 | 1 |

## 実行環境
- コンピュートタイプ: cpu-general のみ（GPU不使用）
- 分析時間: 3-5分/リポジトリ
- 実行時間: 30秒〜数分（通常ケース）

---

## Round 2: 拡張テスト計画 (2026-04-20)

Web調査とRound 1の知見に基づく追加テスト。

### 7. 出力正確性テスト (Output Validation)
- **O-1**: 完了した run の stdout 内容が期待値と一致するか検証
- **O-2**: 同一seed の2 run で stdout が完全一致するか（再現性）
- **O-3**: 数値出力の妥当性チェック（Accuracy 0-1, Loss > 0 等）

### 8. 環境・リソーステスト (Environment & Resources)
- **E-1**: 環境変数注入（set_env_var → コンテナ内で参照）
- **E-2**: メモリ大量確保（100MB → コンテナ制限テスト）
- **E-3**: 大量stdout出力（10,000行以上）
- **E-4**: ネットワークアクセス（外部URLへのHTTPリクエスト）

### 9. コード構造テスト (Code Structure)
- **S-1**: 複数モジュールの相対import（utils/パッケージ構造）
- **S-2**: ファイル出力（output/results.json への書き込み）
- **S-3**: Unicode/日本語コード・出力
- **S-4**: stdin要求スクリプト（input()使用 → タイムアウト期待）
- **S-5**: 非MLスクリプト（データ処理ユーティリティ）

### 10. エラーハンドリングテスト (Error Handling)
- **EH-1**: 明示的 sys.exit(1) → failed + stderr
- **EH-2**: 未処理例外（RuntimeError）→ failed + traceback in stderr
- **EH-3**: 依存関係コンフリクト（numpy>=2.0 vs scipy==1.7.3）
- **EH-4**: サブプロセス生成 + zombie reaping

### 11. パフォーマンス・スケーラビリティテスト (Performance)
- **PF-1**: 重い依存関係のインストール時間（pandas + matplotlib）
- **PF-2**: 複数実験の同時実行時のDockerイメージビルド競合
- **PF-3**: pull_repository → 再分析のワークフロー

### Round 2 テスト対象リポジトリ一覧

| ID | リポジトリ | テストカテゴリ | 検証ポイント |
|---|---|---|---|
| R12 | test-lnar-edge-env-vars | E-1 | 環境変数注入 |
| R13 | test-lnar-edge-file-output | S-2 | ファイル出力 |
| R14 | test-lnar-edge-multi-module | S-1 | 相対import |
| R15 | test-lnar-edge-network-download | E-4 | ネットワークアクセス |
| R16 | test-lnar-edge-memory-intensive | E-2 | メモリ制限 |
| R17 | test-lnar-edge-unicode-paths | S-3 | Unicode対応 |
| R18 | test-lnar-edge-stdin-required | S-4 | stdin処理 |
| R19 | test-lnar-edge-non-ml-script | S-5 | 非ML検出 |
| R20 | test-lnar-edge-conflicting-deps | EH-3 | 依存関係コンフリクト |
| R21 | test-lnar-edge-subprocess | EH-4 | サブプロセス |
| R22 | test-lnar-edge-large-output | E-3 | 大量出力 |
| R23 | test-lnar-edge-exit-code | EH-1, EH-2 | 終了コード |
| R24 | test-lnar-edge-slow-install | PF-1 | 重い依存関係 |
| (既存) | 各完了run | O-1, O-2, O-3 | 出力検証 |
