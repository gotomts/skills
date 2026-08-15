---
name: review-dialogue
maintainer: gotomts
description: PR に付いた人間レビュアーの指摘を GitHub 上で即返信せず、いったん 1 枚の md にまとめて論点化し、crit のインラインコメントでラウンドを回して全件決着させてから、GraphQL でスレッドへまとめて返すためのスキル。各論点を「現状 / 選択肢の表 / 推奨」の型で書いて番号を振り、レビュアーが「R3 は a」だけで答えられる形にする。ラウンドごとに md は追記せず書き直し、決着したものは下半分の表へ 1 行に畳み、未決だけを上半分に厚く残すので、レビュアーは毎回「残り何件か」だけ見れば済む。「レビュー指摘が多くて捌けない」「指摘をまとめて」「まず対話してから返したい」「crit で見せて」「PR のレビューコメントを整理して」「指摘が 16 件付いた」「選択肢を出して」「返信をまとめて投げて」「決まったから GitHub に返して」など、人間レビュアーの指摘を一度に捌く必要がある文脈で必ず使う。CodeRabbit などの bot 指摘は採用 / Skip の 2 値判定で対話ではないので対象外 (enhance-superpowers の write-review-response の領分)。結論が出たあとのコード・文書の修正もこのスキルの外。
argument-hint: "[<PR番号 or PR URL>]  # 例: 946 / https://github.com/gotomts/socialcoffeenote/pull/946"
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Grep
  - AskUserQuestion
---

# review-dialogue

人間レビュアーの指摘が 10 件を超えると、チャットで 1 件ずつ往復するのは成立しない。かといって GitHub のスレッドに直接返すと、返信が散らばって「何が決まって何が未決か」が誰にも読めなくなる。

**1 枚の md を単一の作業面にして、決着したものを畳み、未決だけを残す。** レビュアーが毎ラウンド「あと何件か」だけ見れば済む状態を作る。GitHub へは全件決着してから、まとめて 1 回返す。

$ARGUMENTS

## 基本方針

- **途中で GitHub に返信しない。** 決着していないものに返信すると、GitHub 側と md 側で二重に議論が走る
- **レビュアーの手数を最小にする。** 論点に番号を振り、選択肢を表にし、推奨を明示する。「a」「R3 は a」だけで答えられる形にする
- **md は追記しない。ラウンドごとに書き直す**
- **`--resolve` は付けない。** スレッドを解決するかはレビュアーの判断
- crit CLI の網羅的な仕様は `crit:crit-cli` skill が持っている。ここには**この流れで実際に踏んだところだけ**を書く

## 前提チェック

```sh
command -v crit && command -v gh
```

対象 PR を特定する。引数がなければ `gh pr view --json number,title,url` で現在のブランチの PR を出して確認を取る。

作業用 md はスクラッチパッドに置く。**リポジトリにはコミットしない** — レビュー用の中間物であって成果物ではない。

---

## Phase 1: 指摘を取る

```sh
gh api graphql -f query='query{repository(owner:"O",name:"R"){pullRequest(number:N){
  reviewThreads(first:100){nodes{isResolved isOutdated path line startLine
    comments(first:20){nodes{author{login} body diffHunk}}}}}}}'
```

**`line` だけを信じない。`diffHunk` の最終行で、実際にどの行に付いたかを確かめる。**

実例: `line` の周辺を読んでも意味が通らない指摘が 1 件あった。レビュアーに「どこを指していたか」を聞き直して初めて対象が分かり、結果は**別ファイルの実装コード**だった。周辺を読んで通らなければ、推測で埋めずに聞く。

レビュアーが「チェックを入れたが認識できているか」と聞くことがある。Viewed の状態も取れる。

```sh
gh api graphql -f query='query{repository(owner:"O",name:"R"){pullRequest(number:N){
  files(first:100){nodes{path viewerViewedState}}}}}'
```

## Phase 2: 対象を全部読む

**指摘の周辺だけ読んで答えない。**

指摘は文書やコードの**構造そのもの**に向いていることがある（「この表の分け方だと正しいか判断できない」）。全体を読まないと、何が問題と言われているのか理解できない。

**「〜は無い」「文書化されていない」と書く前に必ず grep する。** 実例: 「方針は記憶にあるだけで `docs/` にない」と書いたが、ADR-0010 と ADR-0011 に明記されていた。レビュアーの前で訂正することになった。

## Phase 3: 論点にまとめる

**コメント数 ≠ 論点数。** 実例では 16 コメント → 14 論点。同じ根から出た指摘（別々の行に付いた「経緯を本文に残すな」2 件）は 1 つにまとめる。

**番号を振る。** `R1`〜（レビュアーの指摘由来）/ `Q1`〜（こちらから問う論点）。1 つの論点が途中で 2 つに割れたら `Q2-a` / `Q2-b` と枝番にする（番号を振り直すと、前ラウンドの会話と対応が取れなくなる）。

論点を 3 つに分ける。

| 区分 | 扱い |
|---|---|
| **要判断** | 選択肢の表を作り、推奨を書く |
| **確認だけ** | 選択肢を作らず「**案**: 〜」と 1 つ出す |
| **別 issue** | この PR では直さないもの。起票内容だけ書く |

## Phase 4: md を書く

### 冒頭に一覧表を置く

**「要判断」が何件かが一目で分かること**が要点。

```markdown
| # | 論点 | 推奨 | 判断 |
|---|---|---|---|
| R1 | ... | a | 要判断 |
| R2 | ... | 案のとおり | 確認だけ |
| R11 | ... | — | 別 issue |
```

### 各論点の型

```markdown
## R7 — <一行で論点>

> <指摘の原文をそのまま引用>

**現状**: <いま文書 / コードがどうなっているか。事実だけ>

**選択肢**

| | 内容 | メリット | デメリット |
|---|---|---|---|
| **a（推奨）** | ... | ... | ... |
| b | ... | ... | ... |

**推奨は a です。** <理由。1〜3 文>
```

### 議論の作法

- **上流にあることは、それが正しいことの根拠にならない。** 実例: ある語について「基本設計が既に使っているから造語ではない」と答えたら、「**基本設計ごと造語の可能性は？ 見逃していただけかもしれない**」と返された。正しい。既存文書を根拠にするなら、その文書自体が検証されたかを確かめる
- **前提が変われば推奨を変える。** 変えるときは「前回まで b を推していたが、〜が分かった時点で理由が消えた」と経緯を明示する。黙って差し替えるとレビュアーが自分の記憶を疑うことになる

## Phase 5: crit で見せる

```sh
crit <file>
```

**background で起動する。** フォアグラウンドで実行するとブロックする。

**URL をそのままユーザーに伝える。**

```
Crit is open at http://localhost:<port>. Leave inline comments, then click Finish Review.
```

## Phase 6: 返信する

**1 件ずつ呼ばない。bulk で 1 回。**

```python
import json, subprocess

replies = [
    {"reply_to": "c_dca0a7", "body": "Q3 に整理しました。..."},
    {"reply_to": "c_1f88b2", "body": "..."},
]
subprocess.run(
    ["crit", "comment", "--json", "--author", "Claude Code"],
    input=json.dumps(replies), text=True, check=True,
)
```

**JSON は Python の `json.dumps` で作ってパイプする。** shell の `echo` で組み立てると、zsh の `echo` が本文中の `\n` を実際の改行に変え、`crit comment --json` が `invalid character '\n' in string literal` で落ちる。

コメントを一覧するとき、**`crit comments --json` は unresolved しか返さない。** 全件は `--all`。これを知らないと「コメントが消えた」と誤認する。

**`crit comment --help` はヘルプを出さない。`--help` がコメント本文として登録される。** ヘルプは `crit --help`。

## Phase 7: ラウンドごとに md を書き直す

**追記しない。書き直す。**

- **冒頭に残件数と読む場所を書く** — 「**残り 2 件です。下の Q2 と Q4 だけ読めば足ります**」
- **決着したものは表 1 行に畳んで下半分へ移す** — `| # | 決定 |` の 2 列で足りる
- **未決だけを上半分に厚く残す**
- **ラウンド番号をタイトルに入れる** — 「（ラウンド 7・最終）」

**これが効く。** 実例ではラウンド 1 は 14 論点の長文だったが、ラウンド 3 で読むべきものが 2 件になり、最終ラウンドは 1 件だった。

## Phase 8: 未決分のコメントを再アンカーする

md を書き直すと、crit のコメントが元の行に取り残される（`drifted: true` が付く）。レビュアーから「本来の位置に移動してほしい」と言われる。

**`crit` に移動コマンドはない。レビューファイルを直接書き換える。**

```sh
crit status --json    # review_file のパスが返る (~/.crit/reviews/<session>/review.json)
cp <review_file> <review_file>.bak    # 必ず先にバックアップ
```

各コメントの `start_line` / `end_line` を新しい行に、`anchor` をその行のテキストに書き換え、`drifted` を消す。

**直すのは、上半分に残っている未決論点に付いたコメントだけでよい。** 下半分の表に畳んだ決着済みコメントは drift したままにする。レビュアーが読むのは未決分だけであり、決着済みを毎ラウンド触ると書き換えミスで壊すほうが高くつく（実例でもラウンド 3 で全件直したあと、以降は 9〜10 件が drift したまま最後まで問題なく進んだ）。

`review.json` の構造で 1 つ注意がある。

- ファイルに紐づくコメントは `files["<path>"].comments[]`
- **review-level コメント（`r_` 接頭辞）は `review_comments[]` という別の配列にいる。** `files` 側を走査しても見つからない

各コメントが持つのは `id` / `start_line` / `end_line` / `body` / `anchor` / `drifted` / `author` / `scope` / `resolved` / `resolved_round` / `carried_forward` / `review_round` / `replies[]`。

## Phase 9: 全件決着してから GitHub へまとめて返す

`crit comments --json`（unresolved のみ）が空になり、レビュアーが Finish Review で承認したら投稿する。

### 9-1. thread ID を取る

```sh
gh api graphql -f query='query{repository(owner:"O",name:"R"){pullRequest(number:N){
  reviewThreads(first:100){nodes{id line comments(first:1){nodes{databaseId body}}}}}}}'
```

### 9-2. 返信本文を JSON に用意し、投稿前にユーザーへ全文を見せて承認を取る

```json
[{"thread": "PRRT_kwDOG3w92c6ZdnFq", "line": 55, "body": "**確定。3 つとも直します。**\n\n..."}]
```

**外向きの操作で、取り消しは削除にしかならない**（通知は消えない）。**必ず承認を取ってから投げる。** ここは 1 回だけの承認ゲートで、代理で判断しない。

### 9-3. thread ID の実在を照合する

```python
real = {n["id"] for n in threads}          # 9-1 で取った id 群
bad = [r["thread"] for r in replies if r["thread"] not in real]
assert not bad, bad
```

**必ずやる。** 実例では 1 件、ID の末尾の大文字を小文字で写していた（`...ZdphX` → `...Zdphx`）。照合しなければ、その 1 件だけ静かに落ちていた。

### 9-4. 投稿する

```python
import json, subprocess, time

q = ('mutation($t:ID!,$b:String!){addPullRequestReviewThreadReply'
     '(input:{pullRequestReviewThreadId:$t,body:$b}){comment{id}}}')
for r in replies:
    subprocess.run(["gh", "api", "graphql", "-f", "query=" + q,
                    "-F", "t=" + r["thread"], "-F", "b=" + r["body"]], check=True)
    time.sleep(0.4)
```

**`-F` で渡す**（`-f` ではない）。1 件ずつ、間に 0.4 秒ほど置く。

### 9-5. 投稿できたか照合する

```sh
gh api graphql -f query='query{repository(owner:"O",name:"R"){pullRequest(number:N){
  reviewThreads(first:100){nodes{line comments{totalCount}}}}}}'
```

**成功件数を数えるだけで終わりにしない。** PR 側から見て `totalCount < 2` の thread が 0 件であること、つまり**未返信が 0 件であること**を確かめる。

**`--resolve` はしない。** スレッドを解決するかはレビュアーの判断。

## このスキルが扱わないもの

- **CodeRabbit など bot の指摘** — 採用 / Skip の 2 値判定であって対話ではない。enhance-superpowers の `write-review-response` の領分
- **`crit pull` / `crit push`** — PR の diff 上にコメントを出し入れするコマンドで、別ファイルの md を単一の作業面にするこの流れとは対象が違う
- **コードや文書を直すこと** — 結論が出たあとの修正は別の作業。決着したら、対象と担当（別セッションに投げるか、この場でやるか）だけ決めて渡す

## 踏んだ罠

| 罠 | 実際に起きたこと |
|---|---|
| `crit comment --help` はヘルプを出さない | `--help` が**コメント本文として登録された**。ヘルプは `crit --help` |
| `crit comments --json` は unresolved のみ | 全件は `--all`。気づかず「コメントが消えた」と誤認しかけた |
| review-level コメント（`r_`）は別の配列 | `review.json` の `review_comments` にあり、`files` 側を走査しても見つからない |
| shell の `echo` で JSON を組み立てた | zsh の `echo` が本文中の `\n` を改行に変え、`crit comment --json` が `invalid character '\n' in string literal` で落ちた。**`json.dumps` で作ってパイプする** |
| thread ID を手で写して 1 文字壊した | `...ZdphX` を `...Zdphx` と書いていた。**投稿前に実在を照合していなければ、その 1 件だけ静かに落ちていた** |
| 実測せずに「文書化されていない」と書いた | ADR-0010 と ADR-0011 に明記されていた。**「無い」と書く前に必ず grep する** |
| 先に決まっていることを正しさの根拠にした | 「基本設計が既に使っているから造語ではない」と答えたら「**基本設計ごと造語の可能性は？**」と返された。**上流にあることは、それが正しいことの根拠にならない** |
| `line` を信じて周辺だけ読んだ | 指摘の対象が別ファイルの実装コードだった。**`diffHunk` の最終行で確かめ、通らなければ聞く** |
