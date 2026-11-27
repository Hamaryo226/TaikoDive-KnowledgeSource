# 07｜Launcher のプロセス起動・更新・修復

対象は Launcher L。これはゲーム本体の起動・判定・描画処理を実装する章ではない。本体への引数は [連携契約](../INTEROP.md)、ゲーム・Assets のマニフェストとパッケージ形式は [配布物の契約](../game/07-platform-distribution.md)。追加の安全要件は明示して区別する。実サービスへの更新適用は試験していない。

## ゲームを別プロセスとして起動

| ID | 契約 | 根拠 |
|---|---|---|
| PLAT-02 | ランチャーは有効なゲーム実行ファイルを要求し、通常の TaikoDive プロセスが既にあれば重複起動を拒否する。プレビュー用に追跡したプロセスは除外する。子プロセスの作業ディレクトリはゲームの build ディレクトリ、引数は個別の引数として渡す。作成失敗は利用者に失敗として返す。これはプロセス名の事前検査であり、原子的な全システム単一性の保証ではない。 | L/Services/GameProcessService.cs |

## 更新マニフェストと信頼境界

| ID | 契約 | 根拠 |
|---|---|---|
| PLAT-10 | Launcher の自己更新は revision に基づき提示する。公開インターフェースは `https://dl.hamaryo.dev/launcher/update-manifest.json` とその `.sig`。本体・Assets と Launcher の3チャネルは独立に比較・記録する。 | L/Services/LauncherUpdateService.cs、L/Services/GameUpdateService.cs |
| PLAT-12 | ランチャーのマニフェスト型は revision、releaseNotes、sha256、size、publishedAt。解析・提示前に、埋め込み公開鍵でマニフェストの**完全な元バイト列**に対する RSA-PSS/SHA-256 の別置き生署名を検証する。末尾改行や整形変更も署名対象。鍵または署名がない場合と検証失敗は拒否、無署名へのフォールバックはない。EXE もサイズと SHA-256 を検査する。 | L/Models/LauncherUpdate.cs、L/docs/launcher-update-signing.md |
| PLAT-14 | ゲーム・素材のハッシュは破損検出であり、外部マニフェスト自体を署名している証拠はない。ZIP 暗号化もランチャーに鍵があるため難読化であって強いアクセス制御ではない。ランチャー署名も古い正当な署名の再送を防がない。初回の署名検証対応版と鍵移行は別途信頼できる経路を要する。秘密鍵、配布資格情報、パッケージ鍵の値は公開しない。 | L/docs/game-update-distribution.md、L/docs/launcher-update-signing.md |
| PLAT-15 | パッケージ URL は HTTPS、所定ホスト、標準ポート、チャネルの所定パスとファイル名に限定し、ユーザー情報や query は拒否。サイズ、SHA-256、名称、archive の整合性を検証する。展開先は build 配下に制限し、絶対パス、親ディレクトリ逸脱、重複や保護対象を拒否する。ポータブル版ではリンク等による展開先逸脱も拒否することを追加安全要件とする。 | L/Services/GameUpdateService.cs、L/Services/GameUpdatePackage.cs |

## 取得・適用・保護・復旧

| ID | 契約 | 根拠 |
|---|---|---|
| PLAT-16 | 検査→ダウンロード→検証→適用→完了／失敗を表示する。途中ファイルがある場合は Range で続きから取得し、206 の開始・終了・全長を照合する。200 が返れば先頭からやり直す。内容長超過、宣言長不一致、完成時ハッシュ不一致は拒否し、成功時のみ完成ファイルへ移す。既存完成ファイルもサイズと SHA-256 を再検査する。 | L/Services/ResumableDownload.cs、L/Models/GameUpdate.cs |
| PLAT-17 | 適用前に既存ファイルと削除対象をバックアップし、適用途中で失敗すれば変更をロールバックする。成功したチャネルのインストール状態には build ディレクトリ、版、revision、時刻とファイル一覧を保存。更新中にゲームを操作させず、失敗や復元失敗は明示し、完全復元を未確認のまま成功としない。 | L/Services/GameUpdateService.cs、L/Models/GameUpdate.cs |
| PLAT-18 | 設定、ユーザー情報、スコア、リプレイ、スクリーンショット、ログ、ランチャー本体を更新対象から守る。素材は `src/` 自体でなくその配下を build 直下に適用。公式 Songs のファイルは更新するが他の曲は削除しない。Texture と Sound に限り一覧にない旧ファイルをバックアップ後に削除し、Songs と Info は比較削除しない。`Info/Chara/` は配布対象、`Info/Chara3D.json` と `Info/User.ini` は保護する。 | L/docs/assets-update-distribution.md、L/docs/game-update-distribution.md、A/src/ |
| PLAT-19 | 整合性検査は保存済み一覧があればオフラインで不足・サイズ違い・ハッシュ違いを検出し、一覧外の追加ファイルは報告しない。一覧がない旧環境では最新パッケージを取得して比較後に削除。モデルパスと animations.glb も検査し、修復は通常更新と同じバックアップ付き適用後に再検査する。 | L/docs/assets-update-distribution.md |
| PLAT-20 | ランチャー自己更新は検証済み EXE をステージし、現行 EXE と設定の復旧用バックアップを確保して、別プロセスが親終了を待って置換・再起動する。ロールバックは信頼できるバックアップに限る。署名鍵を欠くローカルビルドは動作しても更新を受け入れられない。書込み権限不足やファイルロックは失敗として通知し、無条件の管理者権限を前提にしない。 | L/Services/LauncherUpdateService.cs、L/Services/LauncherRollbackService.cs、L/docs/launcher-update-signing.md |

## 公開順と検証

| ID | 契約 | 根拠 |
|---|---|---|
| PLAT-22 | 検証・署名後、実体を先に公開し、ランチャーでは EXE、署名、マニフェストの順に置く。マニフェストは最新 1 件を指す。公開途中の一時的不整合はクライアントが拒否し、後で再試行する。同一チャネルの公開と整理は直列化し、実行中の配布はキャンセルしない。容量予算を先に確認し、現行・直前・猶予期間内のパッケージを整理から保護する。 | L/docs/launcher-update-signing.md、L/docs/game-update-distribution.md、L/docs/r2-storage-management.md |
| PLAT-23 | R2 書込み、ソース取得、署名の権限は CI で分離・最小化し、秘密をソース・ログ・マニフェストに置かない。署名鍵は配布ストレージと別の信頼境界で管理する。公開前にテスト、署名・ハッシュ・サイズ検証、別配置での更新、ユーザーデータ保持と起動を確認する。配布ツールが署名ファイルの存在を確認することと暗号学的な署名検証は別である。 | L/docs/launcher-update-signing.md、L/docs/game-update-distribution.md、L/tools/ci/test_workflow_policy.py、G/tools/ci/test_release_workflow.py、A/tools/ci/test_release_workflow.py |

## 受入例

| ID | 前提・操作 | 期待結果 |
|---|---|---|
| PLAT-AT-01 | 通常ゲームが稼働中にランチャーから再起動を指示 | 二重起動を拒否する。別のプレビューだけが稼働中の場合はその追跡分を除外する。 |
| PLAT-AT-05 | ダウンロードが途中で切断され、その後正しい Range 応答を返す | 途中位置から再開し、全長と SHA-256 が一致した場合のみ完成扱い。Range 非対応の 200 なら先頭から取得。 |
| PLAT-AT-06 | ランチャーマニフェストの署名が欠落、または署名後に内容を変更 | 解析・更新提示前に拒否し、旧実行ファイルを維持する。 |
| PLAT-AT-07 | 素材更新で Texture の旧公式ファイルが消え、独自 Songs とユーザー設定が存在 | 旧 Texture はバックアップ後に削除、独自曲とユーザー設定は保持。適用中の失敗時は変更を復元する。 |
| PLAT-AT-08 | 保存済み一覧のファイルを変更し、一覧外の個人ファイルも追加 | オフライン検査で変更ファイルを報告し、追加ファイルは報告しない。 |
| PLAT-AT-09 | 書込み不可の配置で更新適用を試みる | 失敗を表示し、成功と表示しない。バックアップ・復元の結果を確認できる。 |
