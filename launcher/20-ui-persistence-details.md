# 20 ランチャー表示・設定保存の補足

本章は 05・07 の補足であり、`L/` は提示された launcher スナップショットを示す。数値はソース上の指定で、通常 UI の寸法はデバイス非依存単位、ネイティブのウィンドウ矩形とビットマップはそれぞれ API の座標・画素である。Windows 実機で新たに再現・検証した結果ではない。特定の DPI、フォント、モニター構成での見た目の一致は未検証。

## ウィンドウの記録と復元

| 項目 | 追加仕様 |
|---|---|
| 初回と復元 | 保存配置が null の場合は 1180×760 に resize するだけで、明示移動しない。配置オブジェクトがある場合は 05 の妥当性判定・作業領域補正を経て move/resize し、その後 `isMaximized` が真かつ通常の overlapped presenter なら最大化する。配置が不正でもオブジェクトがあれば既定矩形に move/resize する。`L/MainWindow.xaml.cs::RestoreWindowPlacement`、`L/Services/WindowPlacementService.cs`。 |
| 記録の契機 | 一般的なウィンドウ変更通知すべてではなく、利用者の移動・サイズ変更終了 `WM_EXITSIZEMOVE` とシステムコマンドの最大化・元に戻すを処理後、UI キューに取得を依頼する。最小化中、または overlapped presenter でない場合は記録しない。プログラムによる復元を新規編集扱いしない。`L/MainWindow.xaml.cs::WindowMessage/CaptureUserPlacement`。 |
| 最大化中 | 通常状態の矩形を維持し、最大化フラグだけ更新する。以前の保留配置があればそれを優先し、次に設定済み配置を使う。以前の矩形がなければ位置は現在値、幅高さは 1180×760 を使う。元に戻した後の通常矩形は実際の位置・サイズから取得する。`L/MainWindow.xaml.cs::CaptureUserPlacement`。 |
| テーマ | 起動時にルートへ設定を適用する。`Light` は大文字小文字を区別せず Light、それ以外は Dark だが、永続ファイル読込時は `Dark` または `Light` 以外を未対応として保護する。OS テーマとの自動同期をこの処理から推定しない。`L/MainWindow.xaml.cs::ApplyTheme`、`L/Services/LauncherPreferencesStore.cs::LoadAsync`。 |

## 最終書込、競合、終了

| 項目 | 追加仕様 |
|---|---|
| 遅延と直列化 | 配置登録は最新値と世代を保留し、以前の遅延を取り消して 400 ms 後に書く。既に実行中の配置保存の後に次を直列実行する。保存完了時、自分が保存した同じ保留オブジェクトだけを消し、後から登録された配置を消さない。失敗時は通常運転中なら保留を残す。`L/Services/WindowPlacementPersistence.cs`。 |
| 終了時の排出 | 遅延を中止し進行中の保存を待つ。保留がある間、最新登録を取得して保存を各登録につき一度試みる。保存途中に新しい配置が登録されたらそれも試みる。失敗した同一登録は無限再試行せず取り除き、最終検査と close の間には await や再ディスパッチを挟まない。close 確定後の再入登録は拒否し、close 自体が例外なら確定状態を戻す。配置保存が失敗しても終了は許され、最終値がディスクに残った保証ではない。`L/Services/WindowPlacementPersistence.cs::PrepareCloseAsync`。 |
| 編集中の設定との分離 | 配置保存は確認済み設定の複製に配置だけを加えて書き、未確定のテーマ等を巻き込まない。保存・再読込・初期化は共通 semaphore で直列化。再読込成功または初期化で配置世代を増やし、古い世代の遅延保存は無視する。保存保護中も配置書込を見送る。編集破棄は確認済み設定へ戻すが、その時点の配置は維持する。`L/Services/LauncherContext.cs::SavePlacementAsync/ReloadPreferencesAsync/ResetPreferencesAsync/RevertPreferenceEdits`。 |
| 破棄確認の所有権 | 同時遷移は一つのみ。確認の拒否だけでなくダイアログ表示例外も同意ではない。処理中または終了確定済みの終了要求は拒否する。通常終了は未保存設定があれば「破棄して終了」と「キャンセル」を示し、既定はキャンセル。確認後に編集を破棄し、配置排出後に close する。アップデートも同一の遷移所有権を保ち、インストーラー開始に成功した場合に排出して終了する。`L/Services/PreferencesTransitionCoordinator.cs`、`L/MainWindow.xaml.cs::RequestCloseAsync`。 |
| ファイルと検証 | 標準はローカルアプリデータの `TaikoDiveLauncher/launcher.json`。開発用環境変数 `TAIKODIVE_LAUNCHER_SETTINGS_DIRECTORY` 指定時はそのディレクトリ内。`.lock` を共有不可で開き、競合時は 25 ms 間隔で最大100回待って再試行する。ロード時の実バイトと保存直前の実バイトを比較し、外部変更・削除・新規作成があれば上書きを保護する。`L/Services/LauncherPreferencesStore.cs`。 |
| 未対応値と復旧 | JSON の未知または重複プロパティ、未対応テーマ・バナー、構文や読込エラーでは既定値を返しても保存可とはしない。復旧の初期化は現ファイルがロード時と一致することを確認し、既存内容を `.backup`、競合時は `.backup.1` 以降に新規作成、ディスク flush とバイト再読込検証の後に既定値を公開する。退避失敗なら初期化しない。`L/Services/LauncherPreferencesStore.cs::LoadAsync/BackupAndResetAsync`。 |
| 公開後検証 | UTF-8 BOM なし JSON と改行を `.launcher.tmp` に書き、既存ファイルの有無に応じ move で公開する。公開後に読み直して期待バイトと一致した場合だけロード済みバイトを更新する。公開・検証に失敗したら保存不可にし再読込を要求する。既に公開されたファイルを自動で巻き戻す処理とは限らない。テンポラリ削除は best effort。`L/Services/LauncherPreferencesStore.cs::PublishAsync`。 |

## 配置、詳細画面とフォーカス

| 対象 | 配置・操作 |
|---|---|
| 楽曲画面のレスポンシブ | 幅を最大 1280 とみなした値が 900 未満、または高さ 650 未満なら狭幅。ナビゲーションの曲順・フォルダー・紹介曲を縦 3 行にし、通常幅は曲順とフォルダーを同じ行、紹介曲を次行に配置する。狭幅ではフォルダー操作欄も下段、ドロップ領域の最小高さ 240・高さ 300、曲順領域高さ 520。通常幅ではドロップ領域最小高さ 280、両領域の高さは自動で、内容領域はスクローラーの高さに合わせる。ナビ項目間は行列とも 12。`L/Pages/SongsPage.xaml`、`L/Pages/SongsPage.xaml.cs::UpdateLayoutState`。 |
| ホーム | 幅 820 未満で狭幅パディングを用い、ヒーロー内部余白 16、キャラ表示領域高さ 300、名前札最大幅 320。それ以外はそれぞれ 32、260、360。`L/Pages/HomePage.xaml.cs::HomePage_SizeChanged`。 |
| 最近遊んだ曲の詳細 | 曲ボタンから開く自己ベストダイアログは、タイトルを最大3行で省略しツールチップに全文を置く。難易度は「かんたん、ふつう、むずかしい、おに、裏」の順。記録のある最後の難易度、なければ先頭を選択し、開いた際にそのコンテナーへプログラムでフォーカスを試みる。背景のモーダル面への左押下で閉じ、ダイアログ内部の押下では閉じない。`L/Controls/SongScoreDialog.xaml.cs`。 |
| 詳細の寸法 | 内容スクロール領域幅は `clamp(root幅−80, 220, 480)`、高さは `clamp(root高さ−220, 160, 600)`。内容幅 360 未満なら難易度3列、以上なら5列。項目幅は `max(64, (内容幅−4)/列数)`。狭幅では王冠要約をスコアの下・左揃え、通常幅は右列・右揃え。難易度の王冠画像 24×24、選択結果の画像 40×40、結果内訳は3列2行。`L/Controls/SongScoreDialog.xaml`、`L/Controls/SongScoreDialog.xaml.cs::UpdateAvailableSize/UpdateContentLayout`。 |
| 確認時の既定ボタン | ページ移動の未保存確認と名前札の破棄確認は戻る側を既定とする。ランチャー更新確認もキャンセル既定。一方、修復確認は修復側が既定であり、すべてのダイアログがキャンセル既定という意味ではない。`L/MainPage.xaml.cs`、`L/Pages/NamePlatePage.xaml.cs`、`L/Pages/LauncherSettingsPage.xaml.cs`。 |

## プレイ履歴のグラフ

| 対象 | 配置・操作 |
|---|---|
| 周辺レイアウト | ページ幅 820 未満で狭幅パディング。選手選択は最小幅128・最大幅210、各表示切替ボタン最小幅63。グラフ面は初期高さ160、履歴リスト最大高さ320、凡例の色見本は10×10・角丸3。`L/Pages/PlayActivityPage.xaml`、`L/Pages/PlayActivityPage.xaml.cs`。 |
| セル寸法 | 横幅 `w=max(440, スクロール領域の実幅)`、間隔 `g=w<560 ? 2 : 3`、刻み `s=(w−2g)/53`、グラフ高さ `7s+31`。各セルの左は `g+列×s`。日別は上 `4+行×s`、幅高さとも `s−g`。週別・累計は上4、幅 `s−g`、高さ `7s−g` の一つのボタン内部を7段に分割し、行間を g とする。週・累計の塗り段数は `ceil(値×7/最大値)`。`L/Pages/PlayActivityPage.xaml.cs::Render/Reflow`、`L/Services/PlayActivity.cs::WeeksShown`。 |
| 月ラベル | 文字サイズ12、不透明度0.72。x は `g+週×s`、y は `7s+10`。前の表示ラベルから31以上離れ、右端にも31の余地があるときのみ表示する。幅440未満の表示領域は横スクロールし、狭くなった時には末尾表示を試みる。`L/Pages/PlayActivityPage.xaml.cs::Reflow/PlotScroll_SizeChanged`。 |
| フォーカスと選択 | グラフ全セルをそれぞれ Tab 停止にせず、現在のセルだけを Tab 停止にする。フォーカス獲得で停止位置を移す。矢印は 05 の移動単位で範囲内の別セルにフォーカスし、範囲外でも対象キーは処理済みにする。クリック選択は枠線1を付け期間と件数・一覧を更新する。キーボードのフォーカス移動とクリック選択は別処理である。各セルに期間・回数のアクセシブル名と `PlayCell-モード-日付` の識別子を付ける。`L/Pages/PlayActivityPage.xaml.cs::MoveFocus/SetTabStop/Select`。 |

## 名前札とキャラクタープレビュー

| 対象 | 追加仕様 |
|---|---|
| 名前札の基準面 | プレビュー基準面 345×112 を余白8の Viewbox で等比拡縮する。背景は 326×104、左7・上8。基礎画像 `Texture/NamePlate/0.png` の y=208 から 326×104 を背景として切り出す。アニメーション面は初期 338×41、文字面 345×112。`L/Controls/NamePlateAnimationView.xaml`、`L/Controls/NamePlateTextRenderer.cs::RenderBaseLayers`。 |
| 素材不足 | 種別ディレクトリは2桁名を先に探し、なければ整数名を使う。`Anime.aup2` が存在し読込可能ならアニメーション、そうでなければ `Base.png` の静止画、両方なければ「プレビュー素材が見つかりません」。新たな読込で旧非同期結果が戻っても反映しないよう世代を照合する。タイマー間隔は 1000/30 ms。`L/Controls/NamePlateAnimationView.xaml.cs::ShowNamePlateAsync`。 |
| 名前と称号 | ゲーム設定の `fontOedo`、`fontDomCasual`、`fontFallback` を用い、既定はそれぞれ FOT-大江戸勘亭流 Std E、Dom Casual、Comic Sans MS。名前が空でなく全て ASCII 32〜126 なら Dom Casual、それ以外は大江戸。称号は大江戸。指定フォント生成失敗時はフォールバックを試すが、その失敗まで必ず回復する保証はない。名前は18、称号16のポイント相当を96/72倍してピクセル化し、名前は幅188以内に横方向のみ縮小、称号は幅245以内に縦横等比縮小する。`L/Controls/NamePlateTextRenderer.cs::Render`。 |
| 文字の描画 | 透明画像にパスを描画。名前は左上 (3,3)、黒の幅6・丸接合縁取りと RGB (250,250,250) の塗り。称号は (0,0) から黒塗りで縁なし。背景中心 x=170 をインクの中心として横配置し、名前の上端 y=56、称号の中心 y=52。画像計測には余白6を足し、空の計測は16×16を代用する。`L/Controls/NamePlateTextRenderer.cs::RenderText`、`L/Controls/NamePlateAnimationView.xaml.cs::PositionText`。 |
| 3D 対応検査 | 実行前に `TaikoDive.dll` のメタデータから `TaikoDive.DonModelPreview` 型を探し、アニメーション要求ならさらに `RenderAnimation` メソッドの存在を要求する。未対応なら更新案内を出し、未知の引数で通常ゲームを誤起動しない。`L/Services/Character3DPreviewService.cs::IsSupported/RenderCoreAsync`。 |
| 3D 実行と失敗 | 一意の一時ディレクトリにモデルルート、部品、色、2P向き等の要求を書き、非表示プロセスに `--character-preview`・要求パス・出力パスを個別引数で渡す。60秒の制限。終了コード非ゼロまたは PNG 不在なら `.error.txt` があればその内容、なければ一般的な描画失敗を返す。アニメーションは count 2〜120、有限で正の duration を要求し、各 PNG を読む。タイムアウトはモデルフォルダー確認を案内する。`L/Services/Character3DPreviewService.cs::RenderCoreAsync`。 |
| 中止と手当 | 新規要求や画面離脱で古い要求をキャンセルし、キャンセル結果をエラー表示しない。実際の失敗は画像を消し理由を表示。終了時に未終了の子プロセスツリーを停止し、追跡から除外、作成した一時ディレクトリだけ削除を試みる。静止画は最大8件のキャッシュで、設定・向きと実行ファイル、使用中モデル等のパス・サイズ・更新時刻をキーにし、明示クリア後の古い生成結果を再格納しない。ファイル内容のハッシュによる完全な変更検知ではない。`L/Pages/Character3DPage.xaml.cs::UpdatePreviewAsync`、`L/Services/Character3DPreviewService.cs`。 |

## 確認観点

| 条件 | ソースから期待される結果 |
|---|---|
| 配置の遅延保存中に移動してから閉じる | 進行中の保存の後に新しい保留を試みてから close。保存失敗時も終了しうるため、永続化成功とは分けて確認する。 |
| テーマ編集未確定のまま位置だけ更新 | 確認済みテーマと新しい位置を保存し、未確定テーマは位置書込で混入しない。破棄して終了した場合も位置は保持対象。 |
| ロード後に別プロセスが設定を書換 | バイト不一致を検出し勝手に上書きしない。再読込または保護された復旧操作が必要。 |
| 曲詳細で幅を 360 境界で変える | 内容幅が未満なら3列、以上なら5列。開いた際の選択項目へのフォーカス、背景クリックと内部クリックを区別する。 |
| 3D プレビューに未対応の本体または不足モデル | 更新案内または失敗理由を表示し、誤って通常ゲームを起動しない。ユーザー設定の保存成功とは別に扱う。 |

上記はコードから導いた受入観点であり、本章の作成時に Windows 実機操作・テストを開始していない。
