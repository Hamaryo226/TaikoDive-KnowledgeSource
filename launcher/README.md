# TaikoDive Launcher の再実装仕様

基準は独立した Launcher L `a1267f0`。ゲーム本体の仕様はここに含めない。構成、画面、永続化、プロセス起動、ゲーム・Assets の管理と更新、Launcher 自身の更新を記載する。

## 目次

| 範囲 | 仕様 | 何を実装／比較するか |
|---|---|---|
| 00 | [構成・責務](00-architecture.md) | 参照版、ローカル設定、起動・編集・更新の境界 |
| 04 | [設定・曲データ編集](04-data-editor.md) | 読取・書込制限、ライブラリ、ZIP 取込、外部 Songs |
| 05 | [画面・操作](05-launcher.md) | ホーム、プロフィール、楽曲、プレイ記録、起動設定と更新画面 |
| 06b | [AUP2 プレビュー](06b-media-preview.md) | Launcher 専用のパーサ・表示 |
| 07 | [プロセス起動・更新・修復](07-update-and-launch.md) | 別プロセス起動、取得・適用・署名・復旧 |
| 08 | [適合試験](08-conformance.md) | 合成データ、実画面、更新の異常経路 |
| 09 | [作業ツリー観測](09-worktree-notes.md) | 固定版とは区別する非規範の観測 |
| 13 | [参照版の検証結果](13-baseline-verification.md) | ヘッドレス、画面試験、失敗と未検証 |
| 18 | [入力設定エディター](18-input-editor.md) | 監視、キーコード変換、ゲームパッド割当 |
| 20 | [表示と永続化](20-ui-persistence-details.md) | 配置、テーマ、保存保護、名前札、プレビュー |
| 27 | [Launcher の構成分析](27-architecture-tradeoffs.md) | OS 依存箇所と代替 UI 基盤、比較条件 |

[本体＋Assets](../game/README.md) との間の起動引数、ファイルの所有権、パッケージは [連携契約](../INTEROP.md) を参照。Launcher の可動性と実際のゲーム演奏は、別に検証する。
