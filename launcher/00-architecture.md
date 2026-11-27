# 00｜TaikoDive Launcher の構成と移植設計

**参照版 L**：TaikoDive-Launcher `main` の `a1267f089ee5b0dda23e411ddfe93f14116e9ff9`（2026-11-23 22:14:35 JST）。元実装は C#/.NET 10、Windows App SDK／WinUI。`L/` は Launcher のソース。ゲーム本体の描画・譜面解析・判定・素材ロードをここで実装しない。

## 責務

| 部分 | 入り口 | 所有する責務 |
|---|---|---|
| ホスト・画面 | 設定、テーマ、入力、表示幅 | ホームから各編集画面への遷移、未保存・キャンセル、ウィンドウ配置 |
| ランチャー自身の状態 | `launcher.json`、OSのローカルアプリデータ | 選択インストール先、テーマ、配置等を管理。本体の `Setting.json` とは別 |
| インストールの管理 | 選択されたゲーム配置、共有設定、プロフィール、曲と記録 | 対象パスを検証し、許可された編集・確認・表示のみ行う。集計の正本は本体が書いた履歴 |
| 起動とプレビュー | 実行ファイルと引数、入力や3Dの要求 | 本体を別プロセスとして起動。プレビューを通常の演奏や二重起動判定と区別 |
| 更新・修復 | 検証済みマニフェストとパッケージ | 本体、Assets、Launcher 自身の各チャネルを管理し、適用と復旧に責任を持つ |

## 実装・試験の順序

1. Launcher 固有設定・保存保護・画面遷移を作り、ゲームの実行を必要としないフィクスチャで確認する。
2. 共有ファイルの読取と許可された編集、曲の検索・インポート・記録表示を合成インストールで試験する。
3. 子プロセスの起動・プレビューと、更新・修復・自己更新を隔離して試験する。
4. 固定された本体と Assets を使い、[連携契約](../INTEROP.md) を確認する。本体の演奏互換は本体の[適合試験](../game/08-conformance.md)で独立に判定する。

根拠：`L/TaikoDiveLauncher.csproj`、`L/MainWindow.xaml.cs`、`L/Services/LauncherContext.cs`、`L/Services/GameProcessService.cs`、`L/Services/LauncherPreferencesStore.cs`。OS依存部の分離・移植候補は [Launcher の構成分析](27-architecture-tradeoffs.md)。
