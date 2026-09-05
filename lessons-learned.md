# Lessons Learned — Unity ゲーム開発（Claude Code + Unity CLI）

他プロジェクトに活かすための問題と解決の記録。各実装ステップ完了時に「## Step N」セクションへ追記する。

**記録形式**: `### 1. 発生した現象 (Problem)` / `### 2. 原因 (Root Cause)` / `### 3. 解決策・実装パターン (Solution & Code Pattern)` / `### 4. 次回への教訓 (Key Takeaway)`

---

## Step 1: 迷路ロジック + EditModeテスト

### 【事例】bool配列の初期化漏れで迷路が「全部開いた盤面」になる（テストが検出）
#### 1. 発生した現象 (Problem)
迷路生成ロジック実装後のテストで「行き止まりが0個」「別シードでも同一結果」の2つが失敗。生成された迷路は全方向壁なしの平坦な盤面だった。
#### 2. 原因 (Root Cause)
`Cell`構造体（壁の有無をbool4つで保持）の配列が C# の既定で **false初期化** されるため、全方向「壁なし」状態から生成していた。掘る処理は壁を false にするだけなので、何も変化しなかった。
#### 3. 解決策・実装パターン (Solution and Code Pattern)
コンストラクタで明示的に全壁 true で初期化する。

```csharp
// 「あり」から「なし」を掘るタイプのデータは初期状態を明示する
for (var i = 0; i < Cells.Length; i++)
    Cells[i] = new Cell { North = true, South = true, East = true, West = true };
```
#### 4. 次回への教訓 (Key Takeaway)
bool配列はfalse初期化。「壁ありから掘る」設計では初期化を必ず明示する。ロジックをテストファーストに書くと、この種のバグを生成直後に検出できる。

### 【事例】テストの前提が仕様とずれていた（出口が行き止まりに含まれる）
#### 1. 発生した現象 (Problem)
「入り口・出口は行き止まりに含まれない」という自作テストが失敗。出口セル (9,9) が行き止まり（壁3方向）として検出された。
#### 2. 原因 (Root Cause)
完全迷路（全域木）では出口セルも構造上の行き止まり（葉）になり得る。除外すべきは「アイテムを置く候補」であって、構造判定そのものではない——テストの前提が仕様理解とずれていた。
#### 3. 解決策・実装パターン (Solution and Code Pattern)
テストを緩和（行き止まり存在検証のみに）。除外要件は設計書に移して実装側で対応:

> アイテム配置: 行き止まり座標から**入り口・出口を除いて**乱数で1箇所選択（design.md）
#### 4. 次回への教訓 (Key Takeaway)
テスト失敗は「実装バグ」と「テストの前提誤り」の両方があり得る。どちらの判断をしたかを設計書に残し、後続実装に反映する。

### 【事例】再コンパイル直後のテスト実行で「テスト0件」「Network error」「Thread was being aborted」
#### 1. 発生した現象 (Problem)
`recompile` 直後に `run_tests` を実行すると `Total:0`（テスト未検出）。また `Network error: An error occurred while sending the request`、Consoleには赤いエラー `Failed to handle /api/exec request: Thread was being aborted.`（発生源は Unity.Pipeline 内部）も記録された。
#### 2. 原因 (Root Cause)
コンパイル中・ドメインリロード中はテストアセンブリが未完成で、Pipelineサーバーも応答不能。リロードと重なったコマンドはスレッドごと安全に中断される（Unity.Pipeline パッケージの仕様）。
#### 3. 解決策・実装パターン (Solution and Code Pattern)
完了確認→待ち→実行の手順を必ず踏む:

```bash
unity command recompile
# recompile_status が "completed" になるまでポーリング（5秒間隔）
sleep 8   # ドメインリロード完了待ち（editor_status の domainReloadInProgress:false を確認）
unity command run_tests --mode editor
```
#### 4. 次回への教訓 (Key Takeaway)
ドメインリロード直後の接続エラー・未検出は**想定内**（unity-pipelineスキルに同旨の記載あり）。`Unity.Pipeline.*` 発生源で直後の操作が成功しているエラーは無視してよい。

## Step 2: EditorWindowによる迷路生成

### 【事例】capture_scene_view では Main Camera の調整結果を確認できない
#### 1. 発生した現象 (Problem)
スクリプトで Main Camera を真上向きに調整した後 `capture_scene_view` でスクリーンショットを撮ったが、斜め角度の画像になり調整が反映されていないように見えた。
#### 2. 原因 (Root Cause)
`capture_scene_view` は**エディターのシーンビューカメラ**（ユーザー操作対象のビュー）を撮影する。Main Camera とは無関係。
#### 3. 解決策・実装パターン (Solution and Code Pattern)
検証目的で使い分ける:
- Main Camera（ゲーム画面）→ `unity command capture_game_view`
- エディターの作業ビュー → `unity command capture_scene_view`
#### 4. 次回への教訓 (Key Takeaway)
カメラ制御を実装したら `capture_game_view` で検証する。両キャプチャの使い分けを最初から把握しておく。

### 【事例】トップダウンカメラで座標(0,0)が「左下」に映り、設計書の「左上」と不一致
#### 1. 発生した現象 (Problem)
生成確認のスクリーンショットで、入り口(0,0)の緑マーカーが画面左下に表示された。設計書には「入り口=左上」と記載していた。
#### 2. 原因 (Root Cause)
カメラを真下向き（`Quaternion.Euler(90, 0, 0)`）にすると、ワールドZ+方向が**画面上**に投影される。(0,0)が画面左下になるのは座標系として正しい。
#### 3. 解決策・実装パターン (Solution and Code Pattern)
実装（座標系として正しい投影）を優先し、requirements.md の記載を「左下=入り口、右上=出口」に修正した。
#### 4. 次回への教訓 (Key Takeaway)
画面上の位置を仕様に書くときは、カメラ角度とワールド座標の対応（Z+が画面上、等）まで設計段階で確定する。

### 【事例】Editor asmdef からロジックアセンブリへ参照追加が必要
#### 1. 発生した現象 (Problem)
Editor フォルダのツールから `MazeGenerator`（ランタイム側アセンブリ BallRolling 内）を呼ぶコードが、参照未設定ではコンパイルできない。
#### 2. 原因 (Root Cause)
`Assets/Editor/BallRolling.Editor.asmdef` の references にロジックアセンブリが含まれていなかった。
#### 3. 解決策・実装パターン (Solution and Code Pattern)
asmdef の references に `"BallRolling"` を追加してから実装。エディターツール→ゲームロジック参照はこの1行で解消する。
#### 4. 次回への教訓 (Key Takeaway)
エディターツールでゲームロジックを再利用する構成では、Editor asmdef への参照追加を先に行う（asmdefだけ置いた状態のWarning「no scripts associated with it」はスクリプト配置後に自動解消する一時的なもの）。

## Step 3: 玉と傾け操作

### 【事例】エディター再起動後にautotickが無効化され、CLIコマンドがタイムアウトする
#### 1. 発生した現象 (Problem)
エディター再起動後、`unity command eval` が「Main thread operation timed out」、`editor_status` もタイムアウト。しかしプロセスは生存・アイドルで、Pipeline HTTPサーバー（`/api/status` 直接叩き）は正常応答していた。
#### 2. 原因 (Root Cause)
autotick（バックグラウンドでのメインループ駆動）がエディター再起動で無効化されていた。エディターがフォーカスを持たないとメインスレッドのtickが止まり、コマンドのディスパッチだけが処理されない。
#### 3. 解決策・実装パターン (Solution and Code Pattern)
診断手順: ①CPU変化を見る（静止＝ハングではない）→ ②ポートファイル（`Library/Pipeline/.unity-pipeline-port`）の `evalToken` で `/api/status` を直接HTTP確認（サーバー生存確認）→ ③autotick停止と断定。
回復: エディターを**フォーカス**してもらう → `unity command set_autotick --enable true` を再実行。
#### 4. 次回への教訓 (Key Takeaway)
エディター再起動後のセッションでは autotick の再確認・再有効化を最初に行う。コマンドタイムアウト時は「プロセスCPU → HTTP直接 → autotick」の順に切り分けると原因を速断できる。

### 【事例】迷路再生成でカメラが消える（子オブジェクト巻き込み削除）
#### 1. 発生した現象 (Problem)
迷路を再生成した後、プレイモードで `capture_game_view` が「No camera found」で失敗。evalで確認するとカメラが0台だった。
#### 2. 原因 (Root Cause)
カメラをMazeRoot（傾き同期のための親）の子にしていたが、再生成時の `Undo.DestroyObjectImmediate(MazeRoot)` で**子のカメラも巻き込まれて削除**された。
#### 3. 解決策・実装パターン (Solution and Code Pattern)
削除前にカメラを退避＋欠損時に自動復元の二重対策:

```csharp
if (oldRoot != null)
{
    DetachCameraFrom(oldRoot.transform);   // カメラを親から外す（ワールド位置維持）
    Undo.DestroyObjectImmediate(oldRoot);
}
// FrameCamera内: Camera.main == null なら MainCameraタグ付きで新規作成
```
#### 4. 次回への教訓 (Key Takeaway)
「生成物の親に重要オブジェクトを置く」構成では、再生成の削除処理が子を巻き込む。削除前に退避するか、重要オブジェクトは親の外に置いてコードで追従させる。

### 【事例】入り口を「穴」にすると玉が迷路に入れない（設計矛盾を動作確認で発見）
#### 1. 発生した現象 (Problem)
ステップ2で入り口・出口の両セルの床を穴にしたところ、入り口上にスポーンした玉が穴から迷路の下へ落下し、迷路に入れなかった。
#### 2. 原因 (Root Cause)
「玉は入り口から入り、出口から出る」の正しい解釈は「入り口セルの床に**着地**し、出口セルの穴から**落下**してクリア」。入り口まで穴にする必要はなかった。
#### 3. 解決策・実装パターン (Solution and Code Pattern)
床生成の除外条件を出口のみに変更。design.mdに「出口セルのみ穴。入り口セルは床あり」と明記して仕様を確定した。
#### 4. 次回への教訓 (Key Takeaway)
「入る」「出る」の物理的意味（着地点・落下点）を設計段階で明確にする。キャプチャによる動作確認が設計矛盾の発見に有効。

### 【事例】Unity 6のAPIリネームでコンパイルエラー（rigidbody.drag等）
#### 1. 発生した現象 (Problem)
`Rigidbody.drag`/`angularDrag`、`PhysicMaterial` の使用でUnity 6（6000.6）でコンパイルエラーになる。
#### 2. 原因 (Root Cause)
Unity 6で `Rigidbody.drag`→`linearDamping`、`angularDrag`→`angularDamping`、`PhysicMaterial`→`PhysicsMaterial`（`PhysicsMaterialCombine`）にリネームされた。
#### 3. 解決策・実装パターン (Solution and Code Pattern)
Unity 6用の新API名で記述する（古いドキュメント・記憶ベースのコードに注意）。
#### 4. 次回への教訓 (Key Takeaway)
Unity 6系では物理APIの新名称を使う。コンパイルエラーが出たら旧API名を疑う。

---

### 【事例】「正方形の玉」の解釈ミス（立方体→球体、実際に触って判明）
#### 1. 発生した現象 (Problem)
要件「正方形の玉を転がす」を「立方体（Cube）の玉」と実装したところ、ユーザーは球体を想定していた。「玉＝球体」が自然な解釈であり、立方体は傾けても角が引っかかって転がらず、ゲームとして成立しなかった。
#### 2. 原因 (Root Cause)
曖昧な複合語（「正方形」＋「玉」）を文字通り解釈した。ユーザーの意図は「正方形（グリッド）の迷路を転がる玉」だが、実装前に形状の解釈を確認しなかった。
#### 3. 解決策・実装パターン (Solution and Code Pattern)
`PrimitiveType.Cube` → `PrimitiveType.Sphere` に変更し、要件書・設計書の記載を「球体の玉」に修正（「正方形」は迷路の区画を指す表現と整理）。
#### 4. 次回への教訓 (Key Takeaway)
視覚的な要素（形状・色など）は文面で確定させず、**最初の見た目確認ステップで実物を見てもらう**か、実装前に「〜の解釈でよいか」と一言確認する。「玉」「車」「弾」等の日常語はデフォルト形状が自明な場合、その自明側を疑う。

---

## 一般（環境構築時に記録済みの事例）

### 【事例】Unityエディタ拡張でのオブジェクト生成とUndoの不整合
#### 1. 発生した現象 (Problem)
AIがエディタウィンドウからオブジェクトを生成するコードを実行した際、画面上には生成されるが、Ctrl+Z（Undo）を押しても消えず、シーンを保存して開き直すと生成したオブジェクトが消失する。
#### 2. 原因 (Root Cause)
`Instantiate` や `new GameObject()` をそのまま使用していたため、Unityのエディタ側のシリアライズ管理（シーンの変更検知）に登録されていなかった。また、Undoシステムへの登録が行われていなかった。
#### 3. 解決策・実装パターン (Solution & Code Pattern)
エディタ拡張内でオブジェクトを生成・操作する場合は、必ず `Undo.RegisterCreatedObjectUndo` を使用し、プレハブを配置する場合は `PrefabUtility.InstantiatePrefab` を使用する。

```csharp
// 修正後のベストプラクティス
GameObject go = new GameObject("SpawnedObject");
// 生成直後に必ずUndoに登録する（これにより変更検知とUndoが有効になる）
Undo.RegisterCreatedObjectUndo(go, "Create SpawnedObject");
```
#### 4. 次回への教訓 (Key Takeaway)
Unityのエディタ拡張でシーン上の何かを変更・生成する場合は、ランタイム用のコードをそのまま使わず、必ず `UnityEditor` 名前空間の `Undo` や `PrefabUtility` を経由させること。
