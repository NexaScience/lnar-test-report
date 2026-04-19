# lnar バグ・問題点一覧

## 既知の問題

### BUG-001: restart_analysis の S3 DeleteObject 権限エラー
- **重大度**: Medium
- **再現手順**: restart_analysis を呼び出す
- **エラー**: `AccessDenied: s3:DeleteObject action not authorized`
- **影響**: 失敗した分析を再実行できない
- **発見日**: 2026-04-19

### BUG-002: Run 失敗時に stdout/stderr が S3 に保存されない
- **重大度**: High
- **再現手順**: experiment dict から dockerfile が null の状態で start_run → failed → S3 に NoSuchKey
- **影響**: 失敗原因の調査が不可能
- **発見日**: 2026-04-19

### BUG-003: 分析済み実験の start_run で dockerfile が null になるケース
- **重大度**: High
- **再現手順**: get_analysis で取得した experiment を start_run に渡す → run_config に dockerfile: null → FAILED
- **影響**: 以前成功していた実験が新しい run では失敗する
- **備考**: 4/12 の run (01c14a50) では dockerfile が正常に生成されていた。4/19 の run (0dc004d4) では null
- **発見日**: 2026-04-19
