# 04b リプレイ・ユーザーデータ互換補遺

調査対象はゲーム本体の固定版 G。以下は実装で確認できた挙動であり、バイナリ形式を仮定しない。

## リプレイの永続データ

`best.json` は JSON。現行 `Version` は 2。フィールド（C# 型；未記載の数値・真偽値の初期値は 0 / false）：

| オブジェクト | フィールド |
|---|---|
| ReplayData | `Version: int = 2`, `SongPath: string = ""`, `SongKey: string = ""`, `ChartHash: string = ""`, `SongTitle: string = ""`, `Course: int`, `RecordedAtUtc: DateTime`, `PlayerName: string = ""`, `PlayerTitle: string = ""`, `NamePlateType: int`, `CharaType: string = ""`, `Options: ReplayOptionData`, `Inputs: List<ReplayInputEvent>`, `Judgements: List<ReplayJudgementEvent>`, `Result: ReplayResultData` |
| ReplayInputEvent | `Time: long`, `Hand: int` |
| ReplayJudgementEvent | `Time: long`, `ChipNumber: int`, `Branch: TJA.EBranch`, `Note: TJA.ENotes`, `Judgement: NotePlay.E判定`, `TimeLag: int`, `IsDon: bool`, `IsBigNote: bool` |
| ReplayResultData | `Score`, `Great`, `Good`, `Bad`, `Roll`, `MaxCombo`: int；`Gauge`: double |
| ReplayOptionData | `Speed: double = 1.0`, `Doron: bool`, `Reverse: bool`, `Random: OptionConfig.ERANDOM`, `Skip: bool`, `NotesPosition: double`, `DonChanVoice: bool = true`, `HitSound: int` |

プレート等の追加項目が欠ける旧 Version 2 は既定値で読める。`Inputs` / `Judgements` が null なら空リストに補正。読み込み時には Version=2、Course 一致、SongKey が現行または旧キーと ordinal 一致することを要求する。譜面の現 hash と保存 hash がともに空でないときだけ両者の ordinal 一致を要求する。ファイル不存在・不正・例外は読み込み不可。

### 記録・再生

通常演奏のみ、譜面読込後にセッションと曲キーを検証して記録開始。`SaveBestReplay` 無効、単体リプレイ、オート使用、ゲスト、永続設定としての2人プレイでは記録しない。ゴースト対戦は一時的な2人化なので、それだけでは記録を禁止しない。入力は Hand 0〜3 のみ、オート中は記録しない。判定もオート中と null 音符を除く。完走時、オートを使用した場合や得点が準備時ベスト以下の場合は破棄し、厳密に高い場合のみ保存。準備時ベストは呼び出し側のベストと互換リプレイの得点の最大値。保存後、同じ難易度ディレクトリ内の best.json 以外の *.json を削除する。失敗時はログを出し記録を破棄する。

再生は入力と判定の各列の先頭から現在時刻以下のものを消費し、同時刻なら入力を先に処理する。リトライは先頭から。単体再生は1P、ゴーストは2Pに適用し、記録時のオプションを一時適用して終了時復元する。音符対応は `ChipNumber:Branch数値` の複合キーで Notes チャンネルだけを登録し、重複時は先着。判定に記録された Note を適用し、分岐は記録された判定の対応音符を譜面時刻順に並べ、分岐開始時刻以上の最初のものを使う。該当なしなら変更しない。譜面内容がロード直前に変化した場合、再生セッションを終了し再生要求を解除する。

### 列挙値・キー

列挙は定義順に 0 から採番。`EBranch`: Normal=0, Expert=1, Master=2。`ENotes`: Space=0, Don=1, Ka=2, Don_Big=3, Ka_Big=4, Roll=5, Roll_Big=6, Balloon=7, RollEnd=8, Kusudama=9。`E判定`: 通り越し=0, 不可=1, 可=2, 良=3, お手付き=4。`ERANDOM`: eNo=0, eきまぐれ=1, eでたらめ=2。モードは Off=0, Ghost=1, Replay=2。

保存先は `Info/ScoreData/<ユーザー名>/<SongKey>/Replay/course-<Course>/best.json`。ユーザー名中の OS 禁止ファイル名文字は `_` にし、空白だけなら `Default`。SongKey は譜面のフルパスを `\` 区切りにし、アプリ基準ディレクトリ内ならその相対パスを使い、Invariant 大文字化して UTF-8 の SHA-256 を大文字16進にする。基準外は絶対パスのまま。旧キーは絶対パスを同様に大文字化・ハッシュ。ChartHash は譜面ファイル内容そのものの SHA-256 大文字16進であり SongKey と別物。譜面未存在・読取失敗時は空。選曲中の hash キャッシュはパス（大文字小文字無視）、サイズ、UTC 更新時刻で検証し、演奏準備・読込直前は強制再計算する。互換性キャッシュはユーザーフォルダ、SongKey、Course、ChartHash の組。

旧 `Info/Replay/<正規化ユーザー名>/<旧SongKey>` は新しい Replay ディレクトリがないときディレクトリごと移動する。旧 SongKey のレコードを許容するが、ゲームを移動した後では旧絶対パス由来キーが変わり、旧置き場の探索自体が失敗し得る。

出典: `game/src/Common/Replay/ReplayManager.cs`, `game/src/Common/Score/ScorePath.cs`, `game/src/Common/Song/TJA.cs`, `game/src/Scenes/Game/NotePlay.cs`, `game/src/Scenes/SelectSong/OptionConfig.cs`。

## User.ini

`Info/User.ini` の `1P_User_`〜`9P_User_` をユーザー番号に対応させる。各接頭辞のキーは `NamePlateType`（int、既定 0）、`Name`（既定「どんちゃん」）、`Title`（既定「ドンだーデビュー」）、`CharaType`（文字列、既定 `0`）。ファイル欠如や個別項目欠落でも既定値で文字画像を生成する。パーサーは UTF-16 LE/BE BOM、UTF-8 BOM を検出し、それ以外は Shift_JIS として読むため、BOM なし UTF-8 を互換扱いしない。行頭 `;` をコメントとし、`=` 分割の最初の値だけを使いキーと値を trim、キーは大文字小文字を区別する。整数不正は `int.Parse` で例外になり得る。名前は表示だけでなくスコア・リプレイ保存先の識別子なので、変更で別データになる可能性がある。

別プロセスの書込みと文字コード保持は [Launcher のプロフィール編集](../launcher/04-data-editor.md)。本体と Launcher でキーの解釈が異なる点に注意する。

出典: `game/src/Common/NamePlate/NamePlateParse.cs`, `game/src/Common/Score/ScorePath.cs`。

## 段位履歴

`Info/ScoreData/<ユーザー>/<SongKey>/DanHistory.jsonl` に UTF-8 BOM なしで1プレイ1行追加する。通常曲の Score.dat や既存の履歴は上書きしない。段位譜面で、オート使用・リプレイ・ゲストのいずれでもない場合のみ保存。結果には失敗も含める。列挙値は文字列、フィールド名は以下の PascalCase（リプレイの camelCase とは異なる）。

- Record: `Version:int=1`, `PlayedAt:DateTimeOffset`（ローカル時刻とオフセット）, `DanPath`, `MainTitle`, `DanType`: string、`Gauge`, `GaugeRed`, `GaugeGold`: double、`Passed`, `GoldPassed`, `FailedDuringPlay`: bool、`StartedSongCount`, `DisplayRank`: int、`Total:Stats`, `Songs:List<Song>`, `Conditions:List<Condition>`。
- Stats: `Score`, `Great`, `Good`, `Bad`, `Roll`, `Hit`, `MaxCombo`: int。Hit は Great + Good + Roll。
- Song: `Tja`, `Title`: string、`Course`: ECourse、`Result`: Stats。
- Condition: `ExamNumber:int` は1始まり、`SongNumber:int?` は曲別なら1始まり・全曲共通なら null、`Type`, `Range`: enum、`Red`, `Gold`, `Actual`: double、`Passed`, `GoldPassed`: bool。未定義課題を除外し全曲共通課題は1件のみ記録。

More は実績 >= 閾値、Less は実績 < 閾値で合格。全課題とゲージを満たし全曲開始済みで途中失敗がないことが通常合格の条件、金合格は通常合格にも依存。曲切替直前・リザルト開始時に Score / Combo / Gauge を確定し、当時の条件と実績を保存する。

出典: `game/src/Common/Score/DanPlayHistory.cs`, `game/src/Scenes/Result/DanResult.cs`。

## 時間と中断、JSON の注意

リプレイ `Time` は演奏カウンタの譜面時刻（ミリ秒、開始前は負もあり得る）。`TimeLag` は判定時刻から音符の譜面時刻を引いて int 化したミリ秒で、正は遅い。Hand は 0=左ドン、1=右ドン、2=左カッ、3=右カッ。オート切替時はその時点で記録を取り消す。セッション終了でも未確定記録を破棄し、リトライ時には記録を初期化する。

リプレイは `ConfigManager` により UTF-8 JSON、インデント付き、フィールド名 camelCase、enum は文字列として保存する（上記表は実装型の識別子で、実際の JSON キーは例えば `songPath`, `chartHash`）。JSON をバイナリ形式と解釈しない。`RecordedAtUtc` は記録開始時の UTC 日時。

競合時は具体的な補足を優先する。ポーズの終了操作は [01](01-game-scenes.md) と [17](17-gameplay-effects.md)、ランチャーの履歴画面は [05](../launcher/05-launcher.md) と [20](../launcher/20-ui-persistence-details.md)、ゲーム設定の `Setting.ini` 非移行とバックアップは [04d](04d-settings-and-atomic.md)。形式や字幕の古い非公開版を含むすべての INI 形式を互換対象に追加するものではない。

出典: `game/src/Common/Config/ConfigManager.cs`, `game/src/Scenes/Game/NotePlay.cs`, `game/src/Common/Replay/ReplayManager.cs`, `game/src/Common/Program.cs`。
