# 07 配布・更新・外部インターフェース

## 範囲と読み方

入力スナップショットはゲーム fb12417、ランチャー a1267f0、素材 7c5bb71。以下の `G/`、`L/`、`A/` はそれぞれ入力の game、launcher、assets 直下を指す。これは観察した互換契約であり、**追加の安全要件**は明示して区別する。固定版の確定方法は [00](00-architecture.md)。配布先への接続・更新適用の実機試験は行っていない。

## プラットフォームと起動

| ID | 契約 | 根拠 |
|---|---|---|
| PLAT-01 | ゲームの描画・音声・入力とローカル音量サーバーは Windows 固有機能に依存する。ランチャーも Windows アプリであり、プラットフォーム依存部を移植先の描画、音声、入力、プロセス、ファイル、時刻、通信のアダプターに隔離する。ヘッドレス試験成功は Windows 上の実描画・Discord・OBS の成功を意味しない。 | G/src/Common/Program.cs、G/src/Core/Sound/VolumeControlServer.cs、G/docs/launcher-bridge.md、L/TaikoDiveLauncher.csproj |
| PLAT-02 | ランチャーは有効なゲーム実行ファイルを要求し、通常の TaikoDive プロセスが既にあれば重複起動を拒否する。プレビュー用に追跡したプロセスは除外する。子プロセスの作業ディレクトリはゲームの build ディレクトリ、引数は個別の引数として渡す。作成失敗は利用者に失敗として返す。これはプロセス名の事前検査であり、原子的な全システム単一性の保証ではない。 | L/Services/GameProcessService.cs |
| PLAT-03 | ゲームは相対パスで設定やログを扱うため、直接起動時もゲーム配置の作業ディレクトリが必要。終了時は音量サーバーや表示資源を解放し、未保存の音量を保存する。ポート確保失敗は音量サーバーだけを無効にし、ゲーム全体を起動不能にしない。 | G/src/Common/Program.cs、G/src/Core/Sound/VolumeControlServer.cs |
| PLAT-04 | キャラクタープレビューは `--character-preview`、リプレイ連携は先頭引数 `--launcher-replay`、パス、`--user-slot`、1〜9 の順を使う。リプレイは保存場所、利用者、形式、譜面ハッシュを本体側でも検証する。リプレイ中のゲスト・2人プレイの一時変更を設定ファイルに永続化しない。 | G/src/Common/Program.cs、G/src/Common/LauncherBridge.cs、G/docs/launcher-bridge.md |

## ローカル連携・診断

| ID | 契約 | 根拠 |
|---|---|---|
| PLAT-05 | 音量 API は IPv4 ループバックの TCP 24727 のみ。`GET /volume` は music、soundEffect、master を返す。`POST /volume/music`、`/volume/sound-effect`、`/volume/master` は整数 query の value または delta を受け付け、value を優先し、キュー投入時に 202 を返す。反映はメインスレッドで行われるため応答の現在値は新値とは限らない。設定は変更後 500 ms の静止後または終了時に保存する。 | G/src/Core/Sound/VolumeControlServer.cs |
| PLAT-06 | Host はポート付きの localhost または 127.0.0.1 の単一ヘッダーのみ許し、Origin があれば拒否する。ヘッダー最大 8 KiB・64 行、要求タイムアウト 3 秒、同時最大 8 接続。異常要求は接続単位で切り離す。これはローカルプロセス認証ではない。再実装はループバック以外への公開を拒否し、無制限入力を受けない。 | G/src/Core/Sound/VolumeControlServer.cs |
| PLAT-07 | `Info/TaikoDiveLauncher/integrations.json` の version 1 で Discord とオーバーレイを制御する。Discord 公開および曲名公開は既定で無効。オーバーレイはローカルファイルで、状態を原子的に更新し、5 秒以上古ければ表示せず、終了時にクリアする。外部サービスが利用できなくてもゲーム本体の判定を変更しない。 | G/docs/launcher-bridge.md、G/src/Common/Discord/Discord.cs |
| PLAT-08 | E2E プローブは `TAIKODIVE_E2E_STATUS_PATH` 指定時だけ状態ファイルを出力する。シーン変化時または同一シーンでは最大 200 ms 間隔で、選曲・画面状態、プロセス ID、UTC 更新時刻などを出力し、同一ディレクトリの一時ファイルから置換する。未指定時はファイル操作をしない。テスト観測用で操作 API ではない。 | G/src/Core/Testing/E2EProbe.cs |
| PLAT-09 | 起動診断、フレーム、メモリ、例外は `Log/Startup.log`、`Log/Frame.log`、`Log/Memory.log`、`Log/Error/` 等に出る。ログや設定の保存失敗を成功と誤認しない。音量設定保存の例外は元実装では握りつぶされるため、ポータブル版での可視化は互換動作の追加改善として扱う。 | G/src/Common/Program.cs、G/src/Core/Sound/VolumeControlServer.cs |

## 更新マニフェストと信頼境界

| ID | 契約 | 根拠 |
|---|---|---|
| PLAT-10 | ゲーム、素材、ランチャーの更新状態は別管理。ゲーム・素材は version 比較、ランチャーは revision に基づき更新を提示する。公開インターフェースは `https://dl.hamaryo.dev/game/game-update-manifest.json`、`https://dl.hamaryo.dev/assets/assets-update-manifest.json`、`https://dl.hamaryo.dev/launcher/update-manifest.json` とその `.sig`。 | L/Services/GameUpdateService.cs、L/Services/LauncherUpdateService.cs |
| PLAT-11 | ゲーム・素材外部マニフェストの型は version、revision、releaseNotes、packageFileName、packageUrl、sha256、publishedAt が文字列または日時、size が 64 ビット整数、archive が format・encryption・payload・keyId の文字列を持つオブジェクト。minimumGameVersion、minimumAssetsVersion、minimumLauncherVersion も文字列としてモデルに存在する。内部 `package-files.json` は version、revision と files 配列を持ち、各要素は path、sha256、size。実際の互換性判定は型の存在だけで推定しない。 | L/Models/GameUpdate.cs、L/Services/GameUpdateService.cs |
| PLAT-12 | ランチャーのマニフェスト型は revision、releaseNotes、sha256、size、publishedAt。解析・提示前に、埋め込み公開鍵でマニフェストの**完全な元バイト列**に対する RSA-PSS/SHA-256 の別置き生署名を検証する。末尾改行や整形変更も署名対象。鍵または署名がない場合と検証失敗は拒否、無署名へのフォールバックはない。EXE もサイズと SHA-256 を検査する。 | L/Models/LauncherUpdate.cs、L/docs/launcher-update-signing.md |
| PLAT-13 | ゲーム・素材の外側は WinZip AES-256 の ZIP、中身は `payload.bin` だけ。これは内側の通常 ZIP で、更新ファイルと `package-files.json` を格納。名称はゲームが `TaikoDive_Update_v{major}.{minor}.{patch}_win-x64_{commit7}.zip`、素材が `TaikoDive_Assets_v{major}.{minor}.{patch}_win-x64_{commit7}.zip`。パスワードは正規パッケージ名に対する HMAC-SHA256 を Base64 化したもの。鍵 ID 変更時は対応ランチャーを先に配布する。 | L/docs/game-update-distribution.md、L/docs/assets-update-distribution.md |
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

## 配布運用と品質ゲート

| ID | 契約 | 根拠 |
|---|---|---|
| PLAT-21 | private ソースを公開せず、クリーンな CI ビルドから生成物を作る。ゲーム・素材の公開版は `0.<run_number>.0` とし、コミットの先頭 7 桁を追跡する。公開可能な RELEASE_NOTES は空欄不可、最大 4000 文字で、private コミット件名を自動公開しない。素材は src の更新を配布の入力とする。 | G/RELEASE_NOTES.md、A/RELEASE_NOTES.md、L/docs/game-update-distribution.md、L/docs/assets-update-distribution.md |
| PLAT-22 | 検証・署名後、実体を先に公開し、ランチャーでは EXE、署名、マニフェストの順に置く。マニフェストは最新 1 件を指す。公開途中の一時的不整合はクライアントが拒否し、後で再試行する。同一チャネルの公開と整理は直列化し、実行中の配布はキャンセルしない。容量予算を先に確認し、現行・直前・猶予期間内のパッケージを整理から保護する。 | L/docs/launcher-update-signing.md、L/docs/game-update-distribution.md、L/docs/r2-storage-management.md |
| PLAT-23 | R2 書込み、ソース取得、署名の権限は CI で分離・最小化し、秘密をソース・ログ・マニフェストに置かない。署名鍵は配布ストレージと別の信頼境界で管理する。公開前にテスト、署名・ハッシュ・サイズ検証、別配置での更新、ユーザーデータ保持と起動を確認する。配布ツールが署名ファイルの存在を確認することと暗号学的な署名検証は別である。 | L/docs/launcher-update-signing.md、L/docs/game-update-distribution.md、L/tools/ci/test_workflow_policy.py、G/tools/ci/test_release_workflow.py、A/tools/ci/test_release_workflow.py |

## 受入例

| ID | 前提・操作 | 期待結果 |
|---|---|---|
| PLAT-AT-01 | 通常ゲームが稼働中にランチャーから再起動を指示 | 二重起動を拒否する。別のプレビューだけが稼働中の場合はその追跡分を除外する。 |
| PLAT-AT-02 | 音量 API に正当なローカル取得を行い、次に music の値を指定 | 取得は 200、更新受付は 202。メインスレッド更新後に取得した値が変化し、設定は遅延保存される。 |
| PLAT-AT-03 | Origin 付き要求または Host 不正、上限超過の要求 | 受理しない。他の正当な接続を継続できる。 |
| PLAT-AT-04 | プローブ環境変数なしで起動し、次にテスト用パス指定で起動 | 前者はプローブ出力なし。後者は完全な状態ファイルが作られ、シーン遷移を観測できる。 |
| PLAT-AT-05 | ダウンロードが途中で切断され、その後正しい Range 応答を返す | 途中位置から再開し、全長と SHA-256 が一致した場合のみ完成扱い。Range 非対応の 200 なら先頭から取得。 |
| PLAT-AT-06 | ランチャーマニフェストの署名が欠落、または署名後に内容を変更 | 解析・更新提示前に拒否し、旧実行ファイルを維持する。 |
| PLAT-AT-07 | 素材更新で Texture の旧公式ファイルが消え、独自 Songs とユーザー設定が存在 | 旧 Texture はバックアップ後に削除、独自曲とユーザー設定は保持。適用中の失敗時は変更を復元する。 |
| PLAT-AT-08 | 保存済み一覧のファイルを変更し、一覧外の個人ファイルも追加 | オフライン検査で変更ファイルを報告し、追加ファイルは報告しない。 |
| PLAT-AT-09 | 書込み不可の配置で更新適用を試みる | 失敗を表示し、成功と表示しない。バックアップ・復元の結果を確認できる。 |
