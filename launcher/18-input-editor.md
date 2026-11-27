# 18｜Launcher の入力設定・プレビュー用監視

対象は L。実演奏を判定しない。ここで取得した値を [本体の保存スキャンコードと実入力仕様](../game/18-input-runtime.md) につなぐ。根拠：`L/Services/{InputLabelService,KeyboardInputMonitor,ControllerInputMonitor,InputBindingsStore,InputBindingsEditor}.cs`、`L/Pages/InputSettingsPage.xaml.cs`。静的読解であり、実機認証ではない。

## キーボードの検出・変換と保存

| 項目 | 契約 |
|---|---|
| Launcher監視 | `KeyboardInputMonitor` は virtual-key 7〜254を昇順走査し、現在の高位bitによる新規押下または低位bitの押下履歴を使う。Escは除外。変換成功した最初の1件を返す。`ResetEdges` は現在値で前状態を初期化。短いタップの履歴bitにはOS依存性がある。 |
| 入力イベントから保存値 | `InputLabelService.ToDxLibKeyCode`：Pauseは常に197。scan codeが0なら `MapVirtualKey(vkey, 4)` を用い、下位8bitをscan codeにする。戻り値の上位byteが非零、元のextendedフラグ、またはHome/End/PageUp/PageDown/矢印/Insert/Deleteのvkeyならextended。scan codeが0のままなら変換不能。 |
| extended | 結果 = `(scan code & 255) OR 128`（extended時）。指定scan codeが非零ならvkeyによるextended補完は行わず、受け取ったフラグを使う。Pauseは例外。 |
| 10進ID復元 | Launcher `DeviceId` は `vendorId:productId:deviceOrdinal` の10進文字列。実際の復元は保存された個別数値フィールドからIDを再生成してグループ化し、未接続デバイスも表示する。ID文字列のパーサによる復元は確認できない。16進VID/PID表記をそのまま10進文字列として扱わない。 |
| 保存 | `Setting.json` の `p1Keys` / `p2Keys` 内に `kaLeft`, `donLeft`, `donRight`, `kaRight` の整数配列と、それぞれ末尾に `Controllers` を付けた配列。Controller要素は `vendorId`, `productId`, `deviceIndex`, `deviceOrdinal`, `deviceName`, `inputType`（Button/Switch）, `inputIndex`, `inputValue`。Launcher保存はゲーム実行中拒否、既存ルートを保ちUTF-8で原子的に書く。`InputBindingsStore`。 |
| 読取 | Launcherではキー配列自体が無ければデフォルト、空配列なら空。0〜255以外を除外。VID/PIDはushort範囲へクランプし、controller配列欠落は空。 |
| 編集 | `InputBindingsEditor.AssignKeyboard/AssignController` は他プレイヤーで同一入力使用中なら新規割当を拒否。同一プレイヤー内では入力を他スロットから移動し重複を除く。解除は選択プレイヤーのみ。controller同一性はVID/PID/ordinal/type/index/valueであり、表示名とdeviceIndexは含めない。手編集JSON一般の排他保証ではない。 |

## コントローラー

| 項目 | 契約 |
|---|---|
| 毎回の再列挙 | `ControllerInputMonitor.Update` は `RawGameControllers` の現在順に状態を同期。消えた参照は除去、新参照は挿入、既存参照は移動する。参照同一性で一時状態を維持。読取例外時は未初期化に戻し次回同期。 |
| 初期化 | 初回読取成功時は現在ボタン・スイッチを前回へ複写するので、接続時に既に押下中のものはedgeにならない。holdは成立する。 |
| 入力 | button edgeは現在trueかつ前回false。switch edgeは現在がCenter以外かつ前回と異なる；別方向からの直接遷移もedge。割当switchのholdは現在値の完全一致。indexは0始まり、表示B/Sは1始まり。 |
| switch値 | 0 Center、1 ↑、2 ↗、3 →、4 ↘、5 ↓、6 ↙、7 ←、8 ↖。斜めは対応する上下左右の同時成立ではなく独立値。 |
| axis閾値・帰還 | 監視はaxis配列を読取に渡すが、axisの値をedge/holdや割当に使用しない。よって閾値もヒステリシスも中立復帰条件もこの版には存在しない。軸をボタン化する閾値を推測して実装しない。ドライバが別途button/switchとして露出する場合はその状態に従う。 |
| 検出順 | `TryGetPressed` はデバイス列挙順、各デバイスでボタン昇順、次にswitch昇順の最初のedgeだけ返す。 |

## 受入例

| 条件 | 期待結果 |
|---|---|
| 保存値 VID=4660、PID=22136、ordinal=1 | Launcher IDは `4660:22136:1`。保存済み個別フィールドから再生成。 |
| 通常EnterとテンキーEnter | 28と156として別扱い。Pauseは197。PageUpは201でありテンキー9の73ではない。 |
