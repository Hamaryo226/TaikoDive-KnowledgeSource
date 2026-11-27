# 07｜本体の起動・外部連携と本体／Assets の配布契約

対象はゲーム G と Assets A。本体は直接起動できる。外部アプリとの引数・共有ファイルは [連携契約](../INTEROP.md)、パッケージを取得して既存環境へ適用する実装は [Launcher の更新仕様](../launcher/07-update-and-launch.md)。固定版の記号は [00](00-architecture.md)。配布先への接続・更新適用の実機試験は行っていない。

## 本体の起動・実行時 API・診断

| ID | 契約 | 根拠 |
|---|---|---|
| PLAT-01 | ゲームの描画・音声・入力とローカル音量サーバーは Windows 固有機能に依存する。プラットフォーム依存部を移植先の描画、音声、入力、プロセス、ファイル、時刻、通信のアダプターに隔離する。ヘッドレス試験成功は Windows 上の実描画・Discord・OBS の成功を意味しない。 | G/src/Common/Program.cs、G/src/Core/Sound/VolumeControlServer.cs、G/docs/launcher-bridge.md |
| PLAT-03 | ゲームは相対パスで設定やログを扱うため、直接起動時もゲーム配置の作業ディレクトリが必要。終了時は音量サーバーや表示資源を解放し、未保存の音量を保存する。ポート確保失敗は音量サーバーだけを無効にし、ゲーム全体を起動不能にしない。 | G/src/Common/Program.cs、G/src/Core/Sound/VolumeControlServer.cs |
| PLAT-05 | 音量 API は IPv4 ループバックの TCP 24727 のみ。`GET /volume` は music、soundEffect、master を返す。`POST /volume/music`、`/volume/sound-effect`、`/volume/master` は整数 query の value または delta を受け付け、value を優先し、キュー投入時に 202 を返す。反映はメインスレッドで行われるため応答の現在値は新値とは限らない。設定は変更後 500 ms の静止後または終了時に保存する。 | G/src/Core/Sound/VolumeControlServer.cs |
| PLAT-06 | Host はポート付きの localhost または 127.0.0.1 の単一ヘッダーのみ許し、Origin があれば拒否する。ヘッダー最大 8 KiB・64 行、要求タイムアウト 3 秒、同時最大 8 接続。異常要求は接続単位で切り離す。これはローカルプロセス認証ではない。再実装はループバック以外への公開を拒否し、無制限入力を受けない。 | G/src/Core/Sound/VolumeControlServer.cs |
| PLAT-08 | E2E プローブは `TAIKODIVE_E2E_STATUS_PATH` 指定時だけ状態ファイルを出力する。シーン変化時または同一シーンでは最大 200 ms 間隔で、選曲・画面状態、プロセス ID、UTC 更新時刻などを出力し、同一ディレクトリの一時ファイルから置換する。未指定時はファイル操作をしない。テスト観測用で操作 API ではない。 | G/src/Core/Testing/E2EProbe.cs |
| PLAT-09 | 起動診断、フレーム、メモリ、例外は `Log/Startup.log`、`Log/Frame.log`、`Log/Memory.log`、`Log/Error/` 等に出る。ログや設定の保存失敗を成功と誤認しない。音量設定保存の例外は元実装では握りつぶされるため、ポータブル版での可視化は互換動作の追加改善として扱う。 | G/src/Common/Program.cs、G/src/Core/Sound/VolumeControlServer.cs |

## 本体と Assets の配布物

| ID | 契約 | 根拠 |
|---|---|---|
| PLAT-10 | ゲーム本体と Assets の更新状態は別管理。いずれも version を比較する。公開インターフェースは `https://dl.hamaryo.dev/game/game-update-manifest.json` と `https://dl.hamaryo.dev/assets/assets-update-manifest.json`。 | L/Services/GameUpdateService.cs、L/Services/LauncherUpdateService.cs |
| PLAT-11 | ゲーム・素材外部マニフェストの型は version、revision、releaseNotes、packageFileName、packageUrl、sha256、publishedAt が文字列または日時、size が 64 ビット整数、archive が format・encryption・payload・keyId の文字列を持つオブジェクト。minimumGameVersion、minimumAssetsVersion、minimumLauncherVersion も文字列としてモデルに存在する。内部 `package-files.json` は version、revision と files 配列を持ち、各要素は path、sha256、size。実際の互換性判定は型の存在だけで推定しない。 | L/Models/GameUpdate.cs、L/Services/GameUpdateService.cs |
| PLAT-13 | ゲーム・素材の外側は WinZip AES-256 の ZIP、中身は `payload.bin` だけ。これは内側の通常 ZIP で、更新ファイルと `package-files.json` を格納。名称はゲームが `TaikoDive_Update_v{major}.{minor}.{patch}_win-x64_{commit7}.zip`、素材が `TaikoDive_Assets_v{major}.{minor}.{patch}_win-x64_{commit7}.zip`。パスワードは正規パッケージ名に対する HMAC-SHA256 を Base64 化したもの。鍵 ID 変更時は対応ランチャーを先に配布する。 | L/docs/game-update-distribution.md、L/docs/assets-update-distribution.md |
| PLAT-21 | private ソースを公開せず、クリーンな CI ビルドから生成物を作る。ゲーム・素材の公開版は `0.<run_number>.0` とし、コミットの先頭 7 桁を追跡する。公開可能な RELEASE_NOTES は空欄不可、最大 4000 文字で、private コミット件名を自動公開しない。素材は src の更新を配布の入力とする。 | G/RELEASE_NOTES.md、A/RELEASE_NOTES.md、L/docs/game-update-distribution.md、L/docs/assets-update-distribution.md |

Assets のリポジトリ `src/` の配下は、ゲームの `build` 直下に配置する。バージョンとファイル一覧はゲーム本体の配布物とは別。パッケージの公開・整理は同じチャネルで直列化し、生成物を先に、マニフェストを後に公開する。ソース・資格情報は配布物に入れない。公開前にサイズ・ハッシュ、合成インストールでの素材解決と本体起動を確認する。適用時の利用データ保護は [Launcher の更新仕様](../launcher/07-update-and-launch.md) に従う。

## 本体側の受入例

| ID | 前提・操作 | 期待結果 |
|---|---|---|
| PLAT-AT-02 | 音量 API に正当なローカル取得を行い、次に music の値を指定 | 取得は 200、更新受付は 202。メインスレッド更新後に取得した値が変化し、設定は遅延保存される。 |
| PLAT-AT-03 | Origin 付き要求または Host 不正、上限超過の要求 | 受理しない。他の正当な接続を継続できる。 |
| PLAT-AT-04 | プローブ環境変数なしで起動し、次にテスト用パス指定で起動 | 前者はプローブ出力なし。後者は完全な状態ファイルが作られ、シーン遷移を観測できる。 |
