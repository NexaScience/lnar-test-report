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
