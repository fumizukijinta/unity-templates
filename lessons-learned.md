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

### 【事例】玉が遅い・急加速する・床をすり抜けて消える（トンネリング）
#### 1. 発生した現象 (Problem)
①玉の転がりが遅い ②数秒転がると急加速する ③走行中に玉が急に消える（床突き抜け）。
#### 2. 原因 (Root Cause)
①摩擦0.6＋標準重力＋傾き15°では加速成分が小さい。②③薄いコライダー（床0.2/壁0.15）＋**Discrete衝突検出**では、高速時の1フレーム移動距離がコライダー厚を超えてすり抜ける（トンネリング）。すり抜けかけた際のソルバー補正が急加速に見えることがある。
#### 3. 解決策・実装パターン (Solution and Code Pattern)
- 速度: **重力スケール**（1〜4、デフォルト2.0）を追加し `Physics.gravity = 9.81 * scale`。摩擦デフォルト0.35に低減
- すり抜け: 玉のRigidbodyに **`collisionDetectionMode = ContinuousDynamic`** ＋床厚0.5/壁厚0.2に増厚
```csharp
rigidbody.collisionDetectionMode = CollisionDetectionMode.ContinuousDynamic;
Physics.gravity = new Vector3(0f, -9.81f * p.GravityScale, 0f);
```
#### 4. 次回への教訓 (Key Takeaway)
剛体の速度が上がるゲームでは、最初から ContinuousDynamic ＋ ある程度厚いコライダーを設定する。「遅い→加速したい」調整と「すり抜け防止」はセットで行う。速度調整は角の%指定よりも重力スケール・摩擦など物理量の調整が自然で破綻しない。

---

### 【事例】すり抜け再発 — ContinuousDynamicだけでは防げない「物理爆発」
#### 1. 発生した現象 (Problem)
ContinuousDynamic＋厚いコライダーを適用しても玉が床をすり抜けた。残された状態を診断すると、玉はボード下 y=-67 を **104 m/s** で落下中だった（すり抜け時点でも数十m/s級）。
#### 2. 原因 (Root Cause)
壁同士がオーバーラップする継ぎ目領域に玉が高速でめり込むと、PhysXソルバーが複数のBoxの押し出しを合成して**異常な加速（物理爆発）**を与える。「急加速する時がある」の正体。爆発的速度になると1物理ステップの移動距離が床厚を超え、ContinuousDynamicでも対応できない。
#### 3. 解決策・実装パターン (Solution and Code Pattern)
三重対策:
1. **速度リミッター**（ランタイム、毎FixedUpdateでクランプ）: linear 12 m/s / angular 25 rad/s
```csharp
if (velocity.sqrMagnitude > max * max)
    _rigidbody.linearVelocity = velocity.normalized * max;
```
2. **壁の下端を床に0.1めり込ませる**: 壁と床の接縫（すり抜きやすい縁）を消す
3. **Fixed Timestep 0.02→0.01**: 物理ステップ細分化
#### 4. 次回への教訓 (Key Takeaway)
高速剛体ゲームでは「すり抜け防止＝Continuous系の設定」と思い込みがちだが、**速度そのものを制限するのが最も確実**（爆発も突き抜けも速度上限で封じられる）。連続衝突検出は補助手段。コライダー同士の「面と面のぴったり接触」も縁でのすり抜けを生むため、オーバーラップさせて構築する。

---

### 【事例】低速ですり抜ける真因 — 静的コライダーをTransformで回していた
#### 1. 発生した現象 (Problem)
速度リミッター＋連続衝突検出を適用しても、**低速の「動き始め（傾け開始）のタイミング」**で玉が床をすり抜けた（物理爆発とは別の再発）。
#### 2. 原因 (Root Cause)
床・壁はRigidbodyなしの**静的コライダー**で、BoardControllerが親MazeRootをTransform回転させていた。静的コライダーを移動・回転させるのは物理エンジンのアンチパターンで、接触中の剛体との接触解決が不正確になる。ボードが傾き始めた瞬間、玉が床コライダー内部に相対めり込み、押し出し方向の誤判定で下に抜けた。
#### 3. 解決策・実装パターン (Solution and Code Pattern)
移動・回転するコライダーの親には **Kinematic Rigidbody** を付ける（Unity公式の推奨パターン）:

```csharp
rootBody.isKinematic = true;
rootBody.useGravity = false;
rootBody.interpolation = RigidbodyInterpolation.Interpolate;
```
#### 4. 次回への教訓 (Key Takeaway)
「コライダーを動かす＝Kinematic Rigidbody」は傾け迷路・回転ステージ・エレベーター系すべての基本。すり抜け対策は速度系（リミッター・連続衝突）だけでなく、**コライダーを動かす仕組みそのもの**を最初に確認する。低速ですり抜ける場合は静的コライダー移動を疑う。

---

### 【事例】玉がバウンドして壁を飛び越える（壁高＜玉直径）
#### 1. 発生した現象 (Problem)
ボードを傾けると玉がバウンドし、壁（高さ0.5）を飛び越えて隣の通路へ移動できた。
#### 2. 原因 (Root Cause)
壁高0.5に対し玉直径0.8。**壁が玉より低く**、衝突のバウンドで容易に乗り越えられた。跳ね返り0.1＋重力2倍で跳躍力もあった。
#### 3. 解決策・実装パターン (Solution and Code Pattern)
三重対策: ①壁高デフォルト1.2（玉直径の1.5倍） ②**透明な天井**（BoxColliderのみ、下面=壁上端-0.3）で跳ねても越えられない ③跳ね返り0.02に抑制。
```csharp
// 見えない天井: Rendererなし、コライダーのみ
var box = ceiling.AddComponent<BoxCollider>();
box.size = new Vector3(width + 1f, 0.2f, height + 1f); // 下面が壁上端より低くなる位置に
```
#### 4. 次回への教訓 (Key Takeaway)
転がす物体のゲームでは「障害物の高さ＞物体の跳躍高」を設計時に確認する（目安: 障害物は物体直径の1.5倍以上）。跳ね防止の保険としてコライダーだけの天井は見た目を損なわない定番手法。

---

### 【事例】天井追加で玉が乗ってしまう（蓋と落下経路の干渉）
#### 1. 発生した現象 (Problem)
壁越え対策で透明天井（下面y=0.9）を追加した直後、プレイ開始直後に玉が**天井の上**（スポーン高さy=3から落下して天井上面に着地）に乗ってしまった。
#### 2. 原因 (Root Cause)
玉のスポーン高さ（3.0）が天井より上だった。蓋を追加するとき、**玉の落下経路（入り口上空→床）との干渉**を考慮していなかった。
#### 3. 解決策・実装パターン (Solution and Code Pattern)
天井をセル単位タイル化し、**入り口セルの上だけ穴を開ける**（玉の落下経路を確保）。跳ねによる穴からの脱出は跳ね返り0.02で抑制。
#### 4. 次回への教訓 (Key Takeaway)
密閉する蓋・天井を追加する場合、「物はどこから入るか」の経路を必ず残す。追加前に「入場経路・退場経路・蓋」の3つの位置関係を確認する。

---

### 【事例】傾けても玉が転がらない — Kinematic回転の伝達と摩擦結合
#### 1. 発生した現象 (Problem)
ボードを15°傾けても玉がまったく転がらない（実測：6秒間で位置変化ゼロ）。ユーザーには「前方に傾けても転がらない、後ろに転がるように見える」と報告。
#### 2. 原因 (Root Cause)
二重の原因が重なっていた:
1. **Kinematic Rigidbodyの回転を `transform.rotation` の直接書き換えで行っていた** → PhysX上はテレポート扱いとなり、静止接触中の玉に傾斜（重力の接線成分）が作用しない
2. **玉のPhysicMaterialを frictionCombine=Maximum にしていた** → 素の床コライダー（摩擦0.6）との結合で0.6が採用され、tan15°=0.27 < 0.6 のため静止摩擦が傾斜成分に勝り、玉自体も動けない
#### 3. 解決策・実装パターン (Solution and Code Pattern)
- Kinematic回転は **`Rigidbody.MoveRotation`** を **FixedUpdate** で呼ぶ
- 玉の摩擦結合を **Minimum**（床側の高摩擦を無視して玉側0.15を採用）
- `Rigidbody.WakeUp()` を毎物理ステップ呼び、スリープによる転がり開始の遅れも排除
```csharp
// BoardController.FixedUpdate 内
var next = Quaternion.Slerp(_rigidbody.rotation, target, speed * Time.fixedDeltaTime);
_rigidbody.MoveRotation(next);  // transform.rotation への直接代入は使わない
```
#### 4. 次回への教訓 (Key Takeaway)
「動くコライダー＝Kinematic Rigidbody」で止まらず、**動かす方法も MovePosition/MoveRotation** がセット。transform直接操作は物理に伝わらない。また PhysicMaterial の Combine 設定は相手側コライダーの既定値（摩擦0.6・跳ね0）との合成結果を決めるため、**Minimum にしないと低摩擦設定が無効化される**。傾け迷路の摩擦目安: 転がり開始には tan(傾き角) > 摩擦係数。

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

---

## Step 4（ゲームループ実装）

### 発生した現象
1. プレイモードのスモークテスト中、エディター非フォーカス時にフレームが完全停止（frameCount不変）。set_autotick の再有効化でも回復しない
2. テストのため玉を出口セルへ転送した際、transform.localPosition を書き換えたが物理ボディ（Rigidbody.position）は移動せず、玉が落ちない
3. 反射呼び出しで RegenerateMaze() と _loop.Restart() を個別に呼んだところ、UIテキスト（タイマー・メッセージ）が更新されず、一見実装バグに見えた

### 原因
1. エディター非フォーカス時はプレイヤーループが進行しない。autotick（SignalTick）はプレイモード中は機能しない場合がある
2. Rigidbody（Interpolate）に対する transform 直接書き換えは物理ボディへ反映されない
3. UI更新は RestartGame() 内の ApplyWaitingUi() で行われるため、内部処理を個別に呼ぶとスキップされる

### 解決策・実装パターン
1. フレーム停止時はユーザーにUnityウィンドウのフォーカスを依頼する（frameCountを2回計測して停止を確認してから）
2. 実行中オブジェクトの転送は Rigidbody.position に書き換え、linearVelocity/angularVelocity をゼロ化して WakeUp()
3. 反射で検証する際は公開メソッド相当（StartPlay / RestartGame）を経由して呼ぶ
4. これらの手順は game-tester.md の「スモークテストの手順書」に集約し、以後のスモークテストは game-tester（glm-5.3-flash）が担当

### 次回への教訓
- 実装バグゼロでも検証方法の落とし穴で時間を消費する。実機操作の検証手順は手順書化して軽量エージェントへ移管するとトークン節約になる
- 実装（game-coder）は glm-5.3[1m]、テスト・スモークテスト（game-tester）は glm-5.3-flash というモデル棲み分けを明示

### 【追記】リスタート後即時再ゴールバグ（Step 4 完了後の修正）

**発生した現象**: ゴール後リスタート→開始すると即座に★★☆が再表示されプレイ不能。bug-fixer による SpawnBall() の Rigidbody.position 書き込み修正後も実プレイで再現（反射呼び出しのスモークテストでは合格していた）。

**原因**: Update内で Space → StartPlay()（rb.position書き込み）→ 同一フレームの CheckGoal() が物理同期前の古い transform.localPosition（出口床下）を読み、即Goal成立。Rigidbody.position の書き込みは transform へ即時反映されない。スモークテストの反射呼び出しは eval（エディターループ）から行われ、次のUpdateまでに物理同期が挟まるため「Update内連続実行」を再現できていなかった（テストの呼び出しコンテキスト起因の偽陽性）。

**解決策・実装パターン**: ①SpawnBall() で rb.position 書き込み直後に transform.position も同値で即時同期 ②CheckGoal() は transform でなく物理実位置（rb.position を親の InverseTransformPoint でローカル変換）で判定。スモークテスト手順書（game-tester.md）に「同一フレーム競合の再現」ケースを追加。

**次回への教訓**: リアルタイム物理ゲームのテストは「呼び出しコンテキスト（Update内連続実行 vs 外部からの反射呼び出し）」まで含めて実プレイを再現しないと偽陽性になる。スモークテスト合格後も実機プレイでの最終確認を挟む。

---

## Step 5（アイテムと評価画面実装）

### 発生した現象
1. ユーザー実プレイで玉がアイテムにぶつかっても消えず ITEM GET! も表示されない（スモークテストでは取得成功していた）
2. 別プレイで、出口の穴を経由しないと到達できない行き止まりにアイテムが配置され取得不能（ユーザーの指摘で発覚）

### 原因
1. Stepツールの実行順序依存: Step2再実行はMazeRootごと破棄（玉も消滅）し、Step3で再生成される玉のタグは Untagged のまま（タグ設定はStep5のみ）。ItemPickup の CompareTag("Ball") が無音で失敗
2. ItemPlacer の候補選定が出口セルを通行可能として扱っていた。出口は床の無い穴で「通過＝ゴール落下」のため、出口より奥の行き止まりは物理的に到達不能。実測では66.5%の迷路に該当行き止まりが存在し、全行き止まりの13.4%が該当

### 解決策・実装パターン
1. Step3_PlaySetup.SpawnBall が玉生成時に必ず "Ball" タグを設定（Step5_ItemSetup.RegisterBallTag を public 化して再利用）。ItemPickup はタグ不一致時に警告ログを出すよう修正
2. ItemPlacer に「出口を通行不能としたBFS（FindReachableDeadEnds）」を導入し、入り口から出口を通らず到達可能な行き止まりのみ候補に。50シード一括検証で到達不能配置0件を確認

### 次回への教訓
- 編集ツールがオブジェクトを再生成する場合、そのオブジェクトへの設定（タグ等）は生成側で完結させる。後段ツールでの設定は実行順序依存の罠になる
- 「理論上到達可能（グラフ連結）」と「物理的に到達可能（通路が通行できる形で存在）」は別物。穴・落下・段差など通行不能な要素を明示的にモデル化してから到達性を判定する
- ユーザーの実プレイ観察（スクリーンショット等）は最重要のバグ報告ソース。静的解析の「到達可能」判定と食い違うときは物理条件の前提を疑う

---

## Step 6（調整と仕上げ）

### 発生した現象
1. Step6検証中、エディターが「半吊り」状態になった（HTTPサーバーは生存・ハートビートも更新されるが、eval等のコマンドがUnauthorized/タイムアウトで一切応答しない）。テストエージェント停止後も自然回復せず、Unity再起動で復旧
2. TimeUp経由のリスタート後、待機中（WaitingToStart）に玉が表示されたままになる（Goal経由では発生しない）
3. ビルドで FindFirstObjectByType 非推奨警告（CS0618）

### 原因
1. プレイモード残留中に別コマンドが流入する等でPipelineディスパッチャーのキューが詰まった可能性が高い（前回の RestoreSceneSetupTask 失敗と同系）。ハートビート更新とコマンド処理は別経路のため「生存しているが応答しない」状態が生じる
2. RestartGame() が玉の非表示化を行っていなかった（Awakeの初期化処理のみで待機状態へ戻す処理が無かった）
3. Unity 6 API の非推奨化

### 解決策・実装パターン
1. 半吊り状態は unity close → unity open の再起動で復旧。再起動後は autotick 有効化を忘れない（再現性あり・過去2回同様）
2. RestartGame() に _ball.SetActive(false) を追加（Awakeと同じ扱い）
3. FindFirstObjectByType → FindAnyObjectByType に置換
4. ビルド検証: StandaloneWindows64 成功（116MB・177秒・エラー0）。Pipeline警告はリリースビルドでは無効化されるのが正常

### 次回への教訓
- Pipelineが「ハートビートは更新されるがコマンドが応答しない」なら半吊り。早期にエージェントを停止してエディター再起動した方が、タイムアウト待ちよりも総合的に速い
- 状態を「初期状態に戻す」処理はAwakeだけでなく、状態に戻るすべての遷移（Restart）でも明示的に行う。初期化とリセットの非対称はGoal経由/TimeUp経由で顕在化が分かれる
