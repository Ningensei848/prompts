# 子エージェント向けプロンプトの取扱説明書

## 1. この文書の目的

この文書は、Microsoft Copilot Studio で教育支援用の「子エージェント（専門チューター）」を実装する際の取扱説明書である。

対象は、次のような構成である。

- 親エージェント: 学習者との唯一の会話窓口
- 子エージェント: 特定テーマに特化した専門チューター
- ナレッジソース: 教材、社内マニュアル、仕様書、SharePoint、FAQ など
- 目的: 学習者の理解を支援しつつ、会話の長期化、専門外回答、根拠不足回答を避ける

子エージェント用プロンプトは、単体で完結する「チャットボット人格」ではなく、親エージェントから呼び出される専門処理モジュールとして扱う。

---

## 2. 基本方針

子エージェントは、ユーザーに直接返答する最終応答者ではない。

子エージェントの役割は、親エージェントへ次の情報を返すことである。

- ユーザー向け応答案
- 学習状態の判定
- 継続・終了・確認・再ルーティングの提案
- 根拠不足や専門外の判定
- 親エージェント向け補足

最終的にユーザーへ返答するかどうか、どのような文面で返答するかは、親エージェントが判断する。

---

## 3. Copilot Studio 上の設定項目

子エージェント実装時は、最低限、次を設定する。

1. Name
2. Description
3. Instructions
4. Knowledge
5. Tools
6. Inputs
7. Outputs
8. When this will be used
9. After running
10. Enabled / Disabled
11. テストケース
12. 監査・ログ確認方針

---

## 4. Name 欄の考え方

Name は短く、親エージェントや運用者が識別しやすい名前にする。

避けるべき名前:

- 教育エージェント
- チューター
- 何でも学習支援
- 汎用質問対応

推奨例:

- Excel関数チューター
- 英語時制チューター
- Python基礎チューター
- 情報セキュリティ教材チューター
- ビジネスメールチューター

Name はルーティング精度そのものよりも、運用上の識別性を優先する。  
ルーティング精度に大きく効くのは Description である。

---

## 5. Description 欄案

Description は、親エージェントが「いつこの子エージェントを使うべきか」を判断するための説明である。

汎用的な説明を書いてはいけない。  
専門テーマ、対象範囲、扱えるタスク、扱えない条件を短く具体的に書く。

### 5.1 汎用テンプレート

```text
[専門テーマ]の学習支援を担当する専門チューター。親エージェントから渡された学習者入力、会話要約、教材ソースをもとに、短い説明、段階的ヒント、確認問題、終了判定を返す。専門外・根拠不足・情報不足の場合は回答せず、親エージェントへ差し戻す。
```

### 5.2 Excel 用の例

```text
Excel の VLOOKUP、XLOOKUP、表検索、基本関数の学習支援を担当する専門チューター。親エージェントから渡された学習者入力、会話要約、教材ソースをもとに、短い説明、段階的ヒント、確認問題、終了判定を返す。専門外・根拠不足・情報不足の場合は回答せず、親エージェントへ差し戻す。
```

### 5.3 英語学習用の例

```text
英語の現在完了、過去形、現在形など時制理解の学習支援を担当する専門チューター。親エージェントから渡された英文、学習者の回答、教材ソースをもとに、短い説明、段階的ヒント、確認問題、終了判定を返す。専門外・根拠不足・情報不足の場合は回答せず、親エージェントへ差し戻す。
```

### 5.4 Python 学習用の例

```text
Python の for 文、if 文、関数、リストなど基礎文法の学習支援を担当する専門チューター。親エージェントから渡されたコード、学習者入力、教材ソースをもとに、短い説明、段階的ヒント、演習、終了判定を返す。専門外・根拠不足・情報不足の場合は回答せず、親エージェントへ差し戻す。
```

### 5.5 Description のアンチパターン

```text
教育用のボットです。
```

```text
学習者の質問になんでも答えます。
```

```text
わかりやすく説明します。
```

これらは抽象的すぎるため、親エージェントがどの子エージェントへルーティングすべきか判断しにくい。

---

## 6. Instructions 欄の取扱い

Instructions 欄には、最終版の「子エージェント指示: 専門チューター」を貼り付ける。

ただし、次の点に注意する。

- Instructions は 8,000 文字以内に収める。
- 詳細な代表例を大量に入れない。
- 評価用テストケースは Instructions ではなく別資料で管理する。
- SharePoint や教材ファイルにプロンプト本文を逃がして、Instructions 制限を回避しようとしない。
- ナレッジソースは事実根拠用であり、エージェントの行動制御用ではない。
- 専門テーマ、対象学習者、参照ソース欄だけを子エージェントごとに差し替える。
- 親子分離、JSON 出力、問いは最大1つ、行き詰まり時の直接解説、終了条件は削らない。

---

## 7. Child Agent の Inputs 案

子エージェントには、親エージェントから必要な状態を明示的に渡す。

自然言語の会話履歴だけに依存させると、判定が不安定になる。  
特に、ターン数、行き詰まり回数、現在フェーズは、できるだけ親側の変数として管理して渡す。

### 7.1 推奨 Inputs

| Input 名 | 型 | 必須 | 説明 |
|---|---:|---:|---|
| `learner_input` | String | 必須 | 学習者の直近発言 |
| `topic` | String | 必須 | 今回の学習テーマ |
| `turn_count` | Number | 推奨 | 当該学習セッション内のユーザー発話回数 |
| `phase` | String | 推奨 | `insight` / `final_quiz` / `completed` |
| `stuck_count` | Number | 推奨 | 「わからない」「答えを教えて」等の累積回数 |
| `previous_context` | String | 推奨 | これまでの会話要約 |
| `source_context` | String | 任意 | 親が渡す根拠情報。通常は子の Knowledge を優先 |
| `orchestrator_instruction` | String | 任意 | 親からの追加指示 |

### 7.2 Input 設定例

```text
learner_input
Display name: Learner input
Description: The learner's latest message to be handled by this tutor agent.
Data type: String
Required: Yes
```

```text
topic
Display name: Topic
Description: The learning topic selected by the parent agent.
Data type: String
Required: Yes
```

```text
turn_count
Display name: Turn count
Description: Number of learner turns in the current learning session.
Data type: Number
Required: No
Default: 0
```

```text
phase
Display name: Phase
Description: Current tutoring phase. Expected values are insight, final_quiz, or completed.
Data type: String
Required: No
Default: insight
```

```text
stuck_count
Display name: Stuck count
Description: Number of times the learner expressed confusion or asked for the answer.
Data type: Number
Required: No
Default: 0
```

```text
previous_context
Display name: Previous context
Description: Brief summary of previous exchanges in the current learning session.
Data type: String
Required: No
```

```text
orchestrator_instruction
Display name: Orchestrator instruction
Description: Additional instruction from the parent agent, such as explain and complete, quiz, or summarize.
Data type: String
Required: No
```

### 7.3 Inputs の注意

`learner_input` と `topic` は必須にする。  
`turn_count`、`phase`、`stuck_count` は親側で初期値を持つ。  
子エージェントにユーザーへ直接追加質問させる設定は、原則として使わない。  
入力不足時は、子エージェントが `need_clarification` を返し、親エージェントがユーザーへ確認する。

---

## 8. Child Agent の Outputs 案

子エージェントの返却値は、JSON 文字列1本に寄せるより、Copilot Studio の Outputs として分ける方が望ましい。

親エージェントが後続処理で参照しやすくなり、終了、再ルーティング、確認質問の分岐を組みやすくなる。

### 8.1 推奨 Outputs

| Output 名 | 型 | 説明 |
|---|---:|---|
| `status` | String | 子エージェントの処理結果 |
| `response_mode` | String | 応答方針 |
| `candidate_message` | String | 親がユーザーへ提示できる応答案 |
| `session_should_end` | Boolean | セッション終了推奨 |
| `clarification_question` | String | 親がユーザーに確認すべき質問 |
| `reroute_required` | Boolean | 再ルーティング要否 |
| `reroute_target` | String | 推奨する別エージェントまたは専門領域 |
| `orchestrator_note` | String | 親エージェント向け補足 |
| `source_sufficient` | Boolean | 根拠が十分か |
| `learner_understanding` | String | 推定理解度 |

### 8.2 `status` の値

```text
continue
quiz
complete
explain_and_complete
need_clarification
out_of_scope
insufficient_source
reroute_required
```

### 8.3 `response_mode` の値

```text
socratic
direct_explanation
step_by_step
troubleshooting
quiz
summary
reroute
```

### 8.4 Output 設定例

```text
status
Display name: Status
Description: Processing status. One of continue, quiz, complete, explain_and_complete, need_clarification, out_of_scope, insufficient_source, reroute_required.
Data type: String
```

```text
candidate_message
Display name: Candidate message
Description: A user-facing draft response that the parent agent can review, edit, and send. It must not include internal routing or state information.
Data type: String
```

```text
session_should_end
Display name: Session should end
Description: True if the current tutoring session should end.
Data type: Boolean
```

```text
clarification_question
Display name: Clarification question
Description: One question the parent agent should ask the learner if clarification is needed. Empty when not needed.
Data type: String
```

```text
reroute_required
Display name: Reroute required
Description: True if this request should be handled by another agent or reclassified by the parent.
Data type: Boolean
```

```text
orchestrator_note
Display name: Orchestrator note
Description: Internal note for the parent agent. Not intended for the learner.
Data type: String
```

---

## 9. Knowledge 設定の注意

子エージェントには、専門領域に対応するナレッジソースだけを接続する。

良い設計:

- Excel 子エージェントには Excel 教材だけを接続する。
- 英語時制 子エージェントには英語時制教材だけを接続する。
- セキュリティ 子エージェントにはセキュリティ教材だけを接続する。

悪い設計:

- 全子エージェントに同じ SharePoint 全体を接続する。
- 「教育資料全部」を全子エージェントに接続する。
- 複数の子エージェントが同じ範囲のナレッジを持つ。
- 古い教材と新しい教材を区別せずに混在させる。

ナレッジが重複すると、親エージェントがどの子エージェントを呼ぶべきか迷いやすくなる。  
また、子エージェント側でも専門外の内容に回答しやすくなる。

---

## 10. Tools 設定の注意

子エージェントに Tools を持たせる場合は、次を確認する。

- その Tool は子エージェントの専門領域に必要か。
- 親エージェントから直接使わせるべきか。
- 子エージェント経由でのみ使わせるべきか。
- 類似 Tool や類似 Agent が存在しないか。
- 実行に承認や監査が必要か。
- 削除、更新、送信など副作用を伴う Tool ではないか。

教育支援用の子エージェントでは、原則として副作用を持つ Tool は避ける。

許容しやすい Tool:

- 教材検索
- FAQ 検索
- 用語集検索
- 演習問題取得
- 学習履歴参照

慎重に扱う Tool:

- 成績登録
- ユーザー情報更新
- メール送信
- 外部システム更新
- ファイル削除
- 権限変更

---

## 11. When this will be used の設定

基本は、Description に基づいて親エージェントが呼び出す設定にする。

ただし、類似の子エージェントが多い場合は、親エージェント側にも明示的なルーティング指示を書く。

例:

```text
ユーザーが Excel の関数、表検索、VLOOKUP、XLOOKUP、セル参照について学びたい場合は、Excel関数チューターを呼び出す。
ユーザーが英語の時制、現在完了、過去形、現在形について学びたい場合は、英語時制チューターを呼び出す。
どちらにも該当しない場合は、子エージェントを呼び出す前に学習テーマを確認する。
```

子エージェントの Description だけで完全なルーティングを期待しない。  
親エージェントの Instructions 側でも、どの条件でどの子エージェントを使うかを明示する。

---

## 12. After running の推奨

子エージェント完了後に、子エージェントが直接ユーザーへ返す構成は避ける。

推奨は、親エージェントが Outputs を受け取り、次のように処理する構成である。

1. 子エージェントを呼び出す。
2. Outputs を受け取る。
3. `status` を見る。
4. `candidate_message` を必要に応じて整形する。
5. 親エージェントがユーザーへ1回だけ返答する。
6. `session_should_end` が true なら終了トピックへ進む。
7. `reroute_required` が true なら別エージェントへ再ルーティングする。
8. `need_clarification` なら親が確認質問を1つだけ出す。

「子エージェント完了直後に自動でユーザーへ送信する」設定は、親が状態判定を挟みにくくなるため、教育セッション制御には不向きである。

---

## 13. 親エージェント側で管理すべき変数

子エージェントのプロンプトだけで会話制御を完結させない。  
次の変数は、親エージェントまたは Topic 側で物理的に管理する。

| 変数 | 型 | 用途 |
|---|---:|---|
| `State.Topic` | String | 現在の学習テーマ |
| `State.NumOfTurns` | Number | 学習セッション内のユーザー発話回数 |
| `State.StuckCount` | Number | 行き詰まり表明の回数 |
| `State.Phase` | String | `insight` / `final_quiz` / `completed` |
| `State.LastChildStatus` | String | 子エージェントの前回 status |
| `State.SessionShouldEnd` | Boolean | 終了すべきか |
| `State.RerouteRequired` | Boolean | 再ルーティングすべきか |
| `State.PreviousContext` | String | 会話要約 |

### 13.1 ターン数カウンター

ユーザー発話ごとに `State.NumOfTurns` を +1 する。

```text
On user message:
State.NumOfTurns = State.NumOfTurns + 1
```

### 13.2 行き詰まりカウンター

ユーザー発話に次のような表現が含まれる場合、`State.StuckCount` を +1 する。

```text
わからない
答えを教えて
もう無理
意味がわからない
先に説明して
何をすればいいかわからない
```

実装上は、完全一致ではなく、意図判定またはトピック分岐で扱う。

### 13.3 強制終了条件

次の場合は、子エージェントを呼び出す前に終了または要約へ進める。

```text
State.NumOfTurns >= 6
State.Phase == "completed"
State.SessionShouldEnd == true
```

プロンプトだけで終了させようとせず、条件分岐で物理的にブレーキをかける。

---

## 14. 親エージェント側の処理フロー案

```text
1. ユーザー発話を受け取る。
2. State.NumOfTurns を +1 する。
3. 行き詰まり表明があれば State.StuckCount を +1 する。
4. 学習テーマを判定する。
5. テーマに対応する子エージェントを選ぶ。
6. 子エージェントに Inputs を渡す。
7. 子エージェントの Outputs を受け取る。
8. status を判定する。

   - continue:
     candidate_message を親が整形してユーザーへ返す。

   - quiz:
     candidate_message を返し、State.Phase を completed 予定にする。

   - complete:
     candidate_message を返し、終了トピックへ進む。

   - explain_and_complete:
     candidate_message を返し、終了トピックへ進む。

   - need_clarification:
     clarification_question をユーザーへ返す。

   - insufficient_source:
     根拠不足として、回答できない旨を親が説明する。

   - out_of_scope:
     専門外として、親がテーマ再確認または別エージェントを選ぶ。

   - reroute_required:
     reroute_target を参考に別エージェントへ再ルーティングする。

9. State.PreviousContext を更新する。
10. 必要に応じて State.Phase を更新する。
```

---

## 15. 親エージェントの Instructions に入れるべき補足

親エージェント側にも、子エージェント利用ルールを書く。

例:

```text
教育支援において、ユーザーへの最終応答は必ず親エージェントが行う。子エージェントは専門領域の応答案と状態判定を返すだけであり、子エージェントの内部判断をユーザーにそのまま見せてはいけない。

子エージェントから受け取った candidate_message を確認し、必要に応じて自然な文に整形してからユーザーへ返す。status が need_clarification の場合は clarification_question を1つだけ返す。status が reroute_required または out_of_scope の場合は、内部構造を見せずに適切な専門エージェントへ再ルーティングする。status が complete または explain_and_complete の場合は、セッションを終了する。
```

---

## 16. 妥当性

この設計は、次の点で妥当である。

### 16.1 親子分離が明確

親エージェントが唯一の会話窓口となり、子エージェントは専門処理に限定される。  
これにより、複数エージェントが同時にユーザーへ返答する混乱を防げる。

### 16.2 子エージェントの責務が単一

子エージェントは「特定テーマの教育支援」だけを担当する。  
これは、子エージェントを単一責任のモジュールとして扱う設計に合っている。

### 16.3 ルーティング精度を上げやすい

Description に専門範囲を具体的に書くため、親エージェントが呼び出すべき子エージェントを判断しやすい。

### 16.4 出力を親が制御できる

`status`、`candidate_message`、`session_should_end`、`reroute_required` を分けることで、親エージェントが次の行動を安定して決められる。

### 16.5 プロンプト依存を減らせる

`turn_count`、`stuck_count`、`phase` を親側の変数で管理するため、会話が長くなっても終了条件を維持しやすい。

### 16.6 ソクラテス式問答の暴走を防げる

問いは最大1つ、行き詰まり時は直接解説、一定ターンで終了という制約により、問答が不必要に長引くリスクを抑えられる。

---

## 17. 実装上の注意

### 17.1 Instructions は短く保つ

詳細な教育理論、代表例、評価ケースをすべて Instructions に入れない。  
Instructions には、実行時に必要な行動ルールだけを入れる。

別管理にすべきもの:

- 詳細な設計思想
- 代表出力例
- 評価用テストケース
- 教材作成ルール
- 運用手順書
- 変更履歴

### 17.2 ナレッジを重複させない

複数の子エージェントに同じナレッジソースを広く接続しない。  
重複がある場合は、親エージェントのルーティングが不安定になる。

### 17.3 子エージェントを増やしすぎない

子エージェントを細かく分けすぎると、保守コストとルーティング複雑度が上がる。

分割すべき条件:

- 専門領域が明確に異なる。
- 使う教材やナレッジが異なる。
- ガバナンスや権限が異なる。
- 再利用性が高い。
- 親エージェントが持つツールやトピックが増えすぎている。

分割しなくてよい条件:

- 同じ教材内の小さな章違い。
- 応答スタイルだけが違う。
- ツールや権限が同じ。
- 単なる言い換えで処理できる。

### 17.4 出力 JSON の厳密性を過信しない

Instructions で JSON 出力を指定しても、モデル出力が常に完全な JSON になるとは限らない。  
可能なら、Copilot Studio の Outputs として各値を分けて受け取る。

JSON 文字列を使う場合は、親エージェント側で次を考慮する。

- JSON パース失敗時のフォールバック
- 必須キー欠落時の処理
- 想定外 status の処理
- 空の candidate_message への対応
- 長すぎる candidate_message の切り詰め

### 17.5 `candidate_message` をそのまま信頼しない

子エージェントの `candidate_message` は「応答案」であり、最終回答ではない。  
親エージェントは、次を確認してからユーザーへ返す。

- 専門外情報が混じっていないか。
- 内部状態が漏れていないか。
- 複数質問が入っていないか。
- ソース不足なのに断定していないか。
- 終了条件に反して会話継続していないか。
- 文体が親エージェント全体のトーンと合っているか。

### 17.6 終了は物理制御する

「6ターン以上なら終了」などの制御は、プロンプトだけでなく Topic / Condition / Variables で実装する。

推奨:

```text
If State.NumOfTurns >= 6:
  子エージェントを呼ばず、親エージェントが要約して終了する。
```

または:

```text
If child.session_should_end == true:
  candidate_message を返した後、終了トピックへ進む。
```

### 17.7 行き詰まり判定は親側で持つ

`stuck_count` は子エージェント任せにしない。  
親エージェントがユーザー発話を見て増減させる。

ただし、子エージェントからの `orchestrator_note` を参考に、親が `stuck_count` を調整してもよい。

### 17.8 クイズを必須にしない

教育セッションでは確認クイズが有効な場合がある。  
ただし、業務手順、トラブルシュート、添削、要約のようなケースでは不自然な場合がある。

子エージェントが `quiz` を返しても、親エージェントは文脈上不要と判断したら要約に切り替えてよい。

### 17.9 内部構造をユーザーに見せない

ユーザー向け文面に次を出さない。

- 子エージェント
- 親エージェント
- ルーティング
- status
- phase
- stuck_count
- turn_count
- source_usage
- orchestrator_note
- JSON

内部制御は、あくまで裏側の運用情報として扱う。

### 17.10 テストは正常系だけでなく異常系を含める

最低限、次のケースをテストする。

| ケース | 期待結果 |
|---|---|
| 専門テーマ内の概念質問 | `continue` または `direct_explanation` |
| 学習者が誤答 | 短いフィードバック + 問い1つ |
| 「わからない」を2回表明 | `explain_and_complete` |
| 手順を求める | `step_by_step` |
| エラー相談 | `troubleshooting` |
| 5ターン以上継続 | `quiz` または `complete` |
| 専門外質問 | `out_of_scope` または `reroute_required` |
| 根拠ソース不足 | `insufficient_source` |
| 曖昧な質問 | `need_clarification` |
| 子エージェント出力欠落 | 親がフォールバック |

---

## 18. 評価用テストケース例

### 18.1 Excel VLOOKUP

入力:

```text
VLOOKUPで社員番号から名前を出したいけど、何を検索値にすればいい？
```

期待:

```text
status: continue または direct_explanation
response_mode: socratic または direct_explanation
candidate_message: 検索値は社員番号であることに気づかせる、または短く説明する
```

### 18.2 行き詰まり

入力:

```text
わからない。もう答えを教えて。
```

前提:

```text
stuck_count: 2
```

期待:

```text
status: explain_and_complete
response_mode: direct_explanation
session_should_end: true
```

### 18.3 専門外

Excel 子エージェントへの入力:

```text
英語の現在完了を教えて
```

期待:

```text
status: reroute_required
reroute_required: true
reroute_target: 英語時制チューター
```

### 18.4 根拠不足

入力:

```text
社内独自ルールでは、この関数を使ってよいことになっている？
```

前提:

```text
source_context: なし
```

期待:

```text
status: insufficient_source
source_sufficient: false
candidate_message: 断定しない
```

---

## 19. 運用チェックリスト

公開前に次を確認する。

### 19.1 プロンプト

- [ ] Instructions は 8,000 文字以内か。
- [ ] 専門テーマが明記されているか。
- [ ] 対象学習者が明記されているか。
- [ ] 参照ソースが明記されているか。
- [ ] ソクラテス式問答の条件が明記されているか。
- [ ] 問いは最大1つと明記されているか。
- [ ] 行き詰まり時の直接解説が明記されているか。
- [ ] 終了条件が明記されているか。
- [ ] JSON または Outputs の返却契約が明記されているか。

### 19.2 Description

- [ ] 専門領域が具体的か。
- [ ] 「何でも答える」説明になっていないか。
- [ ] 親が呼び出す条件を判断できるか。
- [ ] 他の子エージェントと説明が重複していないか。

### 19.3 Inputs / Outputs

- [ ] `learner_input` が必須か。
- [ ] `topic` が必須か。
- [ ] `turn_count` を渡せるか。
- [ ] `stuck_count` を渡せるか。
- [ ] `candidate_message` を受け取れるか。
- [ ] `session_should_end` を受け取れるか。
- [ ] `reroute_required` を受け取れるか。
- [ ] `clarification_question` を受け取れるか。

### 19.4 Knowledge / Tools

- [ ] 専門領域に対応する Knowledge だけを接続しているか。
- [ ] 他の子エージェントと Knowledge が過度に重複していないか。
- [ ] 不要な Tools を接続していないか。
- [ ] 副作用のある Tools を不用意に持たせていないか。
- [ ] Tool の利用権限が適切か。

### 19.5 親エージェント連携

- [ ] 親が最終応答する構成になっているか。
- [ ] 子エージェント出力をそのまま自動送信しない構成になっているか。
- [ ] `status` ごとの分岐が実装されているか。
- [ ] `turn_count >= 6` のような物理終了条件があるか。
- [ ] 専門外時の再ルーティングがあるか。
- [ ] 根拠不足時の安全な文面があるか。

### 19.6 テスト

- [ ] 正常系テストを実施したか。
- [ ] 行き詰まりテストを実施したか。
- [ ] 専門外テストを実施したか。
- [ ] 根拠不足テストを実施したか。
- [ ] JSON 崩れまたは Outputs 欠落時のフォールバックを確認したか。
- [ ] 親だけがユーザーへ返答しているか確認したか。

---

## 20. 推奨する最終構成

最終的には、次の構成にする。

```text
親エージェント
  - ユーザーとの唯一の会話窓口
  - 学習テーマ判定
  - 子エージェント選択
  - turn_count / stuck_count / phase 管理
  - 子エージェント Outputs の検証
  - 最終応答生成
  - 終了制御
  - 再ルーティング

子エージェント
  - 専門テーマに限定した教育支援
  - 教材・ナレッジ参照
  - 応答案生成
  - 理解度・終了・根拠不足判定
  - 親への構造化出力

Knowledge
  - 専門領域ごとに分離
  - 重複を避ける
  - 古い教材と新しい教材を混在させない

Topic / Variables
  - ターン数管理
  - 行き詰まり管理
  - 終了条件
  - 再ルーティング
```

この構成により、プロンプトだけに依存せず、Copilot Studio の設定、変数、条件分岐、Outputs を組み合わせて、教育用マルチエージェントを安定して運用できる。
