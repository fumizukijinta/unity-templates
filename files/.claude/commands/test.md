---
description: Unity Test Framework で EditMode / PlayMode テストを実行する
---

プロジェクトルート（カレントディレクトリ）に対して `unity test . $ARGUMENTS` を実行し、結果を報告して下さい。

- EditMode / PlayMode それぞれの結果を要約する（合計・合格・失敗・スキップ件数）
- 失敗したテストがあれば、テスト名・失敗メッセージ・該当コードを示し、原因の分析と修正案を提案する
- テストコードがまだ存在しない場合はその旨を報告し、`Assets/Tests/EditMode/`・`Assets/Tests/PlayMode/` への雛形作成を提案する
