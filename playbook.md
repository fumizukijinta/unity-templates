# Unity 開発プレイブック（事前読み込み用・要点版）

過去プロジェクト（BallRolling等）の lessons-learned.md から、再発防止に効く教訓とコードパターンのみをカテゴリ別に抽出したもの。**実装作業の開始前に必ず読み込む**。詳細な経緯・診断手順が必要な場合は `lessons-learned.md` の該当事例を参照する。

メンテナンス: ステップ完了時にメインセッションが lessons-learned.md の新規事例から再発防止に効く教訓を取り出し、本ファイルへ反映する（本ファイルは整理・統合してよい）。

---

## 物理（Rigidbody・コライダー）

### すり抜け・物理爆発対策（統合チェックリスト: 優先度順）

1. **速度リミッター**（最も確実。爆発的加速も突き抜けも速度上限で封じられる）: 毎FixedUpdateでクランプ
   ```csharp
   if (velocity.sqrMagnitude > max * max)
       _rigidbody.linearVelocity = velocity.normalized * max;
   ```
2. **動くコライダーの親は Kinematic Rigidbody**。移動・回転は `MovePosition` / `MoveRotation` を FixedUpdate で呼ぶ。`transform` 直接操作は物理に伝わらない（テレポート扱い）
   ```csharp
   rootBody.isKinematic = true;
   rootBody.useGravity = false;
   rootBody.interpolation = RigidbodyInterpolation.Interpolate;
   // BoardController.FixedUpdate 内
   var next = Quaternion.Slerp(_rigidbody.rotation, target, speed * Time.fixedDeltaTime);
   _rigidbody.MoveRotation(next);
   ```
3. 動かす物体側は `collisionDetectionMode = ContinuousDynamic`（補助手段。1〜2が効いて初めて意味を持つ）
4. コライダーは厚く（床0.5 / 壁0.2級）。**面と面のぴったり接触は縁ですり抜けを生む**ため継ぎ目はオーバーラップさせる（壁の下端を床に0.1めり込ませる等）
5. それでも残るなら Fixed Timestep 0.02→0.01

- **低速ですり抜けるなら、静的コライダーを Transform で動かしていないか疑う**（物理エンジンのアンチパターン。上記2が対応策）
- 物理爆発（重なったBoxの押し出し合成による異常加速）は速度リミッターで封じる

### 摩擦・転がり

- PhysicMaterial の Combine 設定は相手コライダー（Unity既定: 摩擦0.6・跳ね0）との合成結果を決める。**低摩擦にしたい側は `frictionCombine` / `bounceCombine` = Minimum にしないと設定が無効化される**
- 転がり開始の条件: **tan(傾き角) > 摩擦係数**（摩擦0.6では15°傾けても転がらない）
- スリープによる転がり開始の遅れは、毎物理ステップ `Rigidbody.WakeUp()` で排除
- 速度調整は角度の%指定より重力スケール・摩擦など物理量の調整が自然で破綻しない（`Physics.gravity = 9.81 * scale`）

### 壁越え・跳ね・天井

- 転がす物体のゲームでは**障害物の高さ ＞ 物体の跳躍高**を設計時に確認（目安: 障害物は物体直径の1.5倍以上）
- 跳ね防止の保険として**見えない天井**（Rendererなし BoxCollider のみ）は見た目を損なわない定番手法
- 蓋・天井を追加するときは**「物はどこから入るか」の経路を必ず残す**（入り口セル上だけ穴を開ける等）。追加前に「入場経路・退場経路・蓋」の3つの位置関係を確認
- 跳ね返りは低め（0.02級）に抑制

### Rigidbody の直接操作

- `transform.localPosition` 書き換えは物理ボディに反映されない。**転送は `Rigidbody.position` + `linearVelocity`/`angularVelocity` ゼロ化 + `WakeUp()`**
- `Rigidbody.position` の書き込みは **transform へ即時反映されない**。同一フレーム内で transform を読む処理がある場合は同値で即時同期し、位置判定は物理実位置（`rb.position` を親の `InverseTransformPoint` でローカル変換）で行う

### Unity 6 (6000.x) のAPIリネーム

| 旧 | 新 |
|---|---|
| `Rigidbody.drag` / `angularDrag` | `linearDamping` / `angularDamping` |
| `PhysicMaterial` / `PhysicMaterialCombine` | `PhysicsMaterial` / `PhysicsMaterialCombine` |
| `FindFirstObjectByType` | `FindAnyObjectByType` |

古いドキュメント・記憶ベースのコードでコンパイルエラーが出たら旧API名を疑う。

## エディター拡張

- エディター拡張内でオブジェクトを生成・操作するときは `UnityEditor` 名前空間経由: **生成は `Undo.RegisterCreatedObjectUndo`、Prefab配置は `PrefabUtility.InstantiatePrefab`**。`new GameObject()` のままでは Undo できず、シーン保存・再オープンで生成物が消える
- Editor asmdef からゲームロジックを呼ぶ構成では、**asmdef の references にロジックアセンブリを先に追加**する（スクリプト配置前のWarning「no scripts associated with it」は自動解消する一時的なもの）
- **「生成物の親」に重要オブジェクト（カメラ等）を置かない**。再生成時の Destroy が子を巻き込んで消す。削除前に退避するか、親の外に置いてコードで追従させる
- .meta は手書きしない（Unityの自動生成を待つ / `recompile` でトリガー）

## Unity CLI（Pipeline）運用

- **recompile 後の定型手順**: `recompile_status` が completed になるまでポーリング（5秒間隔）→ sleep 8 + `editor_status` の `domainReloadInProgress:false` を確認 → テスト実行
- ドメインリロード直後の「テスト0件」「Network error」「Thread was being aborted」（発生源 `Unity.Pipeline.*`）は**想定内**。直後の操作が成功していれば無視してよい
- **エディター再起動後は autotick が無効化されている**。最初に `set_autotick --enable true` を再実行。エディター非フォーカス時はフレームが完全停止し得る（`frameCount` を2回計測して停止を確認→ユーザーにフォーカスを依頼）
- コマンドタイムアウト時の切り分け順: ①プロセスCPU変化（静止＝ハングではない）→ ②ポートファイル（`Library/Pipeline/.unity-pipeline-port`）の `evalToken` で `/api/status` を直接HTTP確認 → ③autotick 停止と断定
- **ハートビート更新 ≠ コマンド応答**。「ハートビートは更新されるがコマンドが Unauthorized/タイムアウト」なら半吊り状態。2回連続で不通なら待たずに①実行中エージェント停止②Unity再起動③autotick再有効化（再起動が最速）
- キャプチャの使い分け: **Main Camera（ゲーム画面）の確認は `capture_game_view`**。`capture_scene_view` はエディターのシーンビューカメラを撮るもので Main Camera と無関係

## トップダウンviewの座標系

- カメラを真下向き（`Quaternion.Euler(90, 0, 0)`）にすると**ワールドZ+方向が画面上**に投影される。(0,0)が画面左下になるのは座標系として正しい
- 画面上の位置（左上・右上等）を仕様に書くときは、カメラ角度とワールド座標の対応まで設計段階で確定する

## テスト・検証

- テスト失敗は「実装バグ」と「テストの前提誤り」の両方があり得る。どちらと判断したかを設計書に残し、後続実装に反映する
- **スモークテスト合格 ≠ 実プレイ合格**。反射呼び出し（eval）は Update 内の連続実行を再現できず偽陽性が出る実例あり。テストは「呼び出しコンテキスト（Update内連続実行 vs 外部からの呼び出し）」まで含めて実プレイを再現し、最終ゲートはユーザーの実プレイ確認とする
- 検証方法の落とし穴は手順書化して軽量エージェントへ移管するとトークン節約になる

## 仕様解釈・設計

- 曖昧な視覚要素（形状・色等）は文面で確定させず、**実装前に「〜の解釈でよいか」と確認する**か、最初の見た目確認ステップで実物を見てもらう。「玉」「車」「弾」等の日常語は自明なデフォルト形状を優先して疑う
- 「入る」「出る」等の**物理的意味（着地点・落下点）を設計段階で明確に**する。キャプチャによる動作確認が設計矛盾の発見に有効
- **グラフ連結 ≠ 物理到達可能**。穴・落下・段差など通行不能な要素を明示的にモデル化してから到達性を判定する。静的解析の「到達可能」と実プレイが食い違うときは物理条件の前提を疑う
- **再生成されるオブジェクトへの設定（タグ等）は生成側で完結させる**。後段ツールでの設定は実行順序依存の罠になる（CompareTagが無音で失敗する）
- **初期化とリセットの対称性**: Awake の初期化処理は、その状態に戻るすべての遷移（Restart等）でも明示的に行う。非対称はGoal経由/TimeUp経由で顕在化が分かれる
- bool配列は C# の既定で **false初期化**。「ありから掘る」タイプのデータは初期状態（全true等）を必ず明示する。ロジックをテストファーストに書くと生成直後に検出できる
