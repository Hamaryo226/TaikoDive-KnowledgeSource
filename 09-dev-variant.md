# 09 開発版差分と作業ツリー補遺

## DEV-01 範囲と証拠

本章はゲーム main `fb12417`、開発版 `ff729c1`、Launcher main `a1267f0` の比較に限る。開発版の仕様を main の仕様に混ぜない。`D/` は入力 `dev/`、`G/` は入力 `game/`、`L/` は入力 `launcher/` の相対パスであり、いずれも実行時パスではない。`dev-v-main.txt` はこの二つのゲーム入力のファイル差分である。過去の `D/docs/experimental-vs-main.md` は別の古いコミット間の記録であり、今回 main が開発版の部分集合である根拠にしてはならない。

本章の実行時パスは原則ゲーム実行ルートからの相対パス。2026年11月27日時点のソースと文書を静的に読んだ記述であり、ビルド、DxLib描画、音声、OBS、実機操作や性能は検証していない。作業ツリーの状態は観測時点のもので、入力スナップショットとは別である。

## DEV-02 曲スキャン診断・新着更新

| 項目 | 再構築上の契約 |
| --- | --- |
| 診断 | F4 → システム → 曲の読み込み診断。起動・再読み込みスキャン中に並列収集し、重大度・ファイル・行の順に重複除去済みのスナップショットを表示する。画面を開くだけでは再走査しない。`D/src/Common/Song/SongLoadDiagnostics.cs`、`D/src/Scenes/Loading/LoadSongs.cs`、`D/src/Common/Song/TJA.cs` |
| 表示と失敗分離 | エラー・警告件数、曲名、重大度、ファイル、行、理由、修正方法を表示し、長文は折り返してスクロールする。行不明なら番号を捏造しない。TJA行は空行・コメントを含む1始まり。読めない譜面は飛ばし他を続行し、スコア読込失敗だけでは除外しない。 |
| 互換性 | 不正な数値ヘッダーは警告し従来通り0、存在しない WAVE は警告し曲は残す。未知ヘッダー・コマンド、空 BALLOON、WAVE省略を新たなエラーとしない。自動修復やスコア初期化をしない。 |
| 再診断 | 修正後の再読み込みは既存の曲再読込を通す。タイトル・選曲・段位選択から実行し、演奏中は不可。曲リスト・診断を再構築し、解消項目は消える。 |
| 新着・更新 | `Info/SongLibraryChanges.json` に TJA の SHA-256 と検出履歴を保持し、起動・再読込時の過去30日分を仮想フォルダにする。初回の既存曲は基準で新着ではない。時刻だけの変更は更新ではない。削除記録を残し、元の場所から消えた同内容の移動は検出日時を継承。仮想曲は元パス・最新成績を使い、元譜面や成績を移動しない。`D/src/Common/Song/SongLibraryChanges.cs`、`D/docs/song-library-changes.md` |

## DEV-03 検索条件画面

選曲 F3 の文字検索自体は main にもある。開発版の差分は GUI 条件を追加して組み合わせる点である。F4 → システム → 曲検索の条件を作成で難易度、レベル範囲、クリア、ジャンルを選び、選曲画面で適用して F3 結果へ戻す。文字検索と GUI 条件は AND、クリアは選択難易度に適用する。全条件クリアは文字とジャンルボタンも解除する。`D/src/Scenes/SelectSong/SongSearchFilters.cs`、`D/src/Scenes/SelectSong/SongSearchOverlay.cs`、`D/docs/song-search.md`。

F3 は閉じたジャンル内も検索し、選択曲のフォルダを開いてバー中央へ移動する。段位・最近遊んだ曲は検索対象外。入力文字は全角英数・カタカナ・大文字を正規化し、条件は空白区切り AND。例えば `oni>=9 clear=2` はおに★9以上かつ同じおにでフルコン以上。ジャンルボタン複数選択は OR。入力時は IME を有効にし背後の選曲と機能キーを抑止、閉じても条件を保持し演奏開始で初期化する。開閉はそれぞれ380ms・320msで、開く途中はクリックを拾わず、閉じる途中も背後を操作させない。

## DEV-04 素材検査と手動ホットリロード

F4 → システム → スキン素材の検査・プレビューで `Info/Chara` のキャラを選び、JSON書式、PNG、aup2参照を検査する。任意ファイルを必須扱いしない。問題の詳細はファイル・行・理由・修正方法を示し、aup2の参照解決は本体と共通で制作環境の絶対パスを直接探索しない。PNGとaup2のプレビューは独立所有し、戻る・設定を閉じる・タブ切替で解放する。

「変更した素材を再読み込み」は手動、演奏中不可。PNG／aup2と参照先を読み直し、新表示の準備成功後に交換して旧リソースを解放する。保存途中・不正な更新では旧表示を維持し理由を出す。実画像展開・描画は未検証。`D/src/Common/Chara/AssetInspection.cs`、`D/src/Core/Util/ReloadableAsset.cs`、`D/src/Scenes/Settings/AssetInspectionMenu.cs`、`D/docs/asset-inspection.md`。

## DEV-05 演出プリセットと OBS

プリセットは `HideBackground`、`HideDancers`、`ReduceBgTextureColorTo16bit`、`UseCompressedSongSound` をまとめて設定する。非表示フラグの真は表示しない意味。画面表示の変更は即時、色深度と圧縮音声保持は次回起動時に反映。適用前の4値を同一起動中に復元可能。低負荷の実際の効果は未測定。`D/src/Common/Config/VisualPresets.cs`、`D/docs/settings-menu.md`。

| プリセット | 背景非表示 | ダンサー非表示 | 背景16bit | 圧縮音声 |
| --- | --- | --- | --- | --- |
| 配信 | 偽 | 偽 | 偽 | 真 |
| 低負荷 | 偽 | 真 | 真 | 真 |
| ノーツ集中 | 真 | 真 | 真 | 真 |

OBS連携は `EnableObsOverlay=false`、`ObsTargetScore=0` が既定。F4のシステムで有効化し、OBSブラウザソースに `http://127.0.0.1:24727/overlay` を設定する。既存のローカル音量サーバーを共用し、GET `/overlay` は透過HTML、GET `/overlay/state` は `phase`、`title`、`course`、`score`、`target` のJSONを返す。有効でない要求や許可されない要求は拒否。状態更新とブラウザ再取得は250ms間隔、取得タイムアウト1500ms。目標0は表示しない。通信失敗時は旧値を消して「接続待ち」。通常は1Pの曲・難易度・点、段位演奏中は現在課題、段位リザルトでは段位名と合計点を表示する。`D/src/Common/ObsOverlay.cs`、`D/src/Core/Sound/VolumeControlServer.cs`、`D/src/Common/Config/ConfigInI.cs`。

## DEV-06 カスタム段位

F4 → システム → カスタム段位ビルダーで曲を3つ、順序、難易度、全体・曲別条件を編集する。`DanDraft` 既定はタイトル「自作段位」、IDは32桁 GUID、ゲージ合格80・金100、曲難易度3。条件既定は Type=2、Red=50、Gold=10、Less=true。条件型0〜7は順に良・可・不可・連打・コンボ・良＋可＋連打・スコア・ゲージ、出力キーは `jp,jg,jb,r,c,h,s,g`。未満 `l` は境界値を含まず、以上 `m` は含む。しきい値は非負、金は未満条件では赤以下、以上条件では赤以上。

保存前に ID、改行のないタイトル、3曲、ゲージ順序、ローカルパスと実在する難易度0〜4を検査する。Shift_JIS へ変換不能なら書き込み前に失敗する。`Songs/CustomDan/<ID>.dan` は曲ごとに `TJA:` 相対パス、`COURSE:`、`TITLE:`、任意 `EXAM2:`、`NEXT` を並べ、最後の `[Common]` に `MAINTITLE:`、`DANTYPE:創作`、`GAUGE:赤,金`、`EXAM1:キー,赤,金,lまたはm` を出力する。同 ID の JSON は再編集用。未存在時のみ `box.def` と `CenterText.apt` を作る。原曲や既存段位は変更せず、曲再読込後に段位選択へ反映。`D/src/Common/Score/DanDraft.cs`、`D/src/Scenes/Settings/DanBuilderMenu.cs`、`D/docs/dan-builder.md`。

## DEV-07 大会、遠征、実績と2P

| 機能 | 状態・形式・統合 |
| --- | --- |
| 大会 | F4 → システム → 大会進行モード。既定タイトル「TaikoDive大会」、`Active=false`、`AllowModifiers=false`、参加者・課題は空。`Info/Tournaments/<ID>.json` に ID、Title、Active、AllowModifiers、Participants、Songs、Scores、SeenPlays を、`Active.json` に現行 ID を保存。課題は Path、Hash、Course（既定3）。開始時の譜面内容を固定し、ユーザー名が一致する有効な通常演奏を課題ごとに自己最高点として集計、未演奏は0、合計同点は同順位。オート・リプレイ・特訓・ゲストは除外、ランダム等は開始前の設定次第。順位画面で更新・終了。過去の大会ファイルは消さない。`D/src/Common/Score/Tournaments.cs`、`D/docs/tournaments.md` |
| Dive遠征 | ゲストOFFで作成。`Info/ScoreData/(ユーザー)/Dive.json` に Version=1、ID、User、Depth=0、Selected=null、Outcome、Layers、SeenPlays を保存。浅瀬・海溝・深海の3層×2ルート。候補曲を索引順に循環し、鬼があれば鬼、なければ最初の難易度を使う。Aはゲージ以上、最終層は100以上、他層はその難易度のクリアゲージ以上。Bは不可がそれぞれ50、25、10未満。各ノードは Title、Path、Hash、Course、Condition。選択時に譜面ハッシュを確認、通常選曲へ移動して指定難易度で演奏。条件達成でDepthを1進め選択解除、未達なら同ルート再挑戦。3層完了で踏破。別ユーザー・パス・難易度・ハッシュや変更オプション、無効な演奏、重複IDは進行しない。新規作成は当該ユーザーの進行を置換。`D/src/Common/Score/DiveExpedition.cs`、`D/docs/dive-expedition.md` |
| 実績 | `Info/ScoreData/(ユーザー)/Achievements.json` に Version=1、Imported、Equipped（既定空）、Unlocked を保存。`first-full-combo` は金の星、`dive-complete` は青の星。初回一覧で過去フルコンと遠征完了を取り込み、通常演奏での獲得はリザルト通知、重複解除しない。装備星をネームプレート右上に図形描画し、未知装飾IDは非表示で獲得記録は保持。オート・リプレイ・特訓・ゲストは対象外。`D/src/Common/Score/Achievements.cs`、`D/src/Core/Rendering/AchievementBadge.cs`、`D/docs/achievements.md` |
| 2P統合 | リザルトで各プレイヤーのユーザー、スコア、ゲージ、難易度、オプションを個別に進行記録へ渡す。1Pの演奏IDを維持し2Pには `:2P` を付け、同じユーザーでも別演奏として重複排除する。大会・遠征・実績の適格性は各記録で判定し、既存スコア保存は別経路を維持。`D/src/Scenes/Result/Result.cs`、`D/tests/TaikoDive.Tests/Tests/進行連携Tests.cs` |

## DEV-08 main 側にあって開発版側にないもの

今回の `dev-v-main.txt` の D は main に存在し dev にないことを示す。`G/src/Common/LauncherBridge.cs`、紹介曲関連の `G/src/Common/Song/DemoSongGenre.cs`、`DemoSongPlaylist.cs`、`DemoSongPreviewFade.cs`、`DemoSongTemplate.cs`、`G/src/Scenes/Demo/DemoSongArtwork.cs`、`DemoSongShowcase.cs` が該当する。対応する `LauncherBridgeTests.cs`、`紹介曲Tests.cs`、`DanMalformedFileTests.cs`、`DefMalformedFileTests.cs`、`tests/font-render-vertical-regression.py` も開発版側にはない。これはファイル存在差であり、同じ振る舞いが別実装で全く存在しないことの証明ではない。逆に DEV-02〜07 の新規ファイルは開発版側にのみある。`D/src/Scenes/Demo/Demo.cs` 等、両方にあるファイルも内容は異なる。

フォルダ単位の比較では、開発版の `src/Scenes/Settings` に素材検査・段位・大会・遠征・実績の各メニューと文字入力部分が増え、`src/Common/Score` に進行保存群が増える。一方 main の `src/Common/Song` と `src/Scenes/Demo` には上記紹介曲群がある。実行時に作られる `Songs/CustomDan`、`Info/Tournaments`、ユーザー別 Dive・実績 JSON、`Info/SongLibraryChanges.json` は開発版機能の出力先であり、入力ツリーに実ファイルが存在するという意味ではない。

## DEV-09 現行未コミット作業ツリー：スナップショットとは別

以下は ローカルの GitHub 作業フォルダーにあった3作業ツリーについて、観測時の `git status --short` と一部差分・未追跡文書を読んだ結果。各作業ツリーの HEAD と上記固定コミットの同一性は検証していない。未追跡ファイルは通常の `git diff --stat` に含まれない。以下を DEV-02〜08 の確定コミット差分へ上書きしてはならない。

| 作業ツリー | 観測した差分 |
| --- | --- |
| TaikoDive | 追跡済み変更は `src/Core/Rendering/AnimationTexture.cs` と `src/Scenes/Demo/Demo.cs`。未追跡に `docs/demo-song-showcase.md`、`src/Common/Song/DemoSongGenre.cs`、`DemoSongPlaylist.cs`、`DemoSongPreviewFade.cs`、`DemoSongTemplate.cs`、`src/Scenes/Demo/DemoSongArtwork.cs`、`DemoSongShowcase.cs`、`tests/TaikoDive.Tests/Tests/紹介曲Tests.cs`。紹介枠1〜13、main曲と自動／手動サブ4曲、13枠への繰り返し、紹介音声の0.5秒フェードを文書化。`Info/TaikoDiveLauncher/demo-songs.json` は version 2、enabled、entries の mainSong・autoSubSongs・subSongs、Songs相対パス。旧 version 1 は読込み移行し保存時に version 2 化。音源不足等は当該枠を飛ばし、全枠無効や非対応形式等では標準紹介へ戻す。ゲーム実行と外観は未検証。 |
| TaikoDive-Launcher | 追跡済み27ファイルに変更・削除。ホーム、入力・起動・楽曲画面、保存サービス、README、テスト、スモークツール等を変更し、旧入力デバイス画像2枚・そのREADME・`Pages/HomePage.Cards.cs` を削除。未追跡には紹介曲画面・モデル・保存サービス・テスト、演奏活動画面・モデル・サービス・テスト、コントローラー描画、プレビュー画像と文書がある。紹介設定はゲーム側と共有しゲーム実行中の保存拒否、保存時 `.launcher.bak`、未保存移動の確認を記載。演奏活動集計コードはローカル日付換算で365日・月曜始まり53週を生成する。その他の詳細な振る舞いは未検証。 |
| TaikoDive-Launcher-Recovery | `Pages/LauncherSettingsPage.Recovery.cs`、`Pages/LauncherSettingsPage.xaml`、`Tests/LauncherRecoveryUiTests.cs` を変更。差分では設定の再読込・退避して初期化のボタン処理を削除し、保護中の案内文を変更している。保存保護全体が削除されたとは限らない。 |

## DEV-10 検証境界

根拠は各記載の `D/`、`G/`、`L/`、入力差分一覧と作業ツリーの読み取り。テストファイルの存在はテスト成功を意味しない。本補遺ではゲーム・ランチャー・OBS の起動、永続化の実書込み、ネットワーク、2P、画像・音声・パフォーマンスの実測は行っていない。Windows、日本語コードページ、DxLib、使用可能な素材・譜面やユーザーデータに依存する箇所は対象環境で別途確認すること。
