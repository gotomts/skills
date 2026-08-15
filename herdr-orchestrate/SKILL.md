---
name: herdr-orchestrate
maintainer: gotomts
description: herdr 上で親セッションをオーケストレーターとして使い、issue (GitHub 番号 / Linear ID) から worktree + workspace + タブを作り、子 Claude セッションを起動して初回プロンプトを投入し、Linear 起点なら status を In Progress に進める。起動済みの子セッション群を巡回して blocked / done を拾い、規約で決まることは即答し・方針判断はサブエージェントに諮って決定し・設計文書の承認だけ人間に上げるモード (/loop に乗せて自動巡回もできる) と、未コミット・未 push を数えてから worktree / タブ / ブランチをどこまで消すか確認して落とすモードも持つ。「ABC-123 立ち上げて」「issue 46 を別セッションで走らせて」「並行で回したい」「子セッション起動して」「各セッションの状況どう」「blocked ないか見て」「巡回して」「ABC-123 落として」「あのタブ閉じて」「もう要らないから片付けて」「自動で巡回しといて」など、複数 issue の並行進行・起動済みセッションの様子見・子セッションの後片付けを示唆する文脈で必ず使う。対象は起動元セッションと同じリポジトリの workspace のみで、別リポジトリの子には触らない。monorepo では「v2 の方だけ見せて」「web の ABC-155 立ち上げて」のようにタブ名の短縮キーでグループを分けて扱える。worktree を作るだけで子セッションを起こさないなら wt-start を使う。
argument-hint: "[<issue-id> | patrol [<group>] | stop <issue-id>]  # 例: ABC-123 / web ABC-155 / patrol v2 / stop ABC-123"
allowed-tools:
  - Bash
  - Read
  - Agent
  - AskUserQuestion
  - ListAgents
  - SendMessage
---

# herdr-orchestrate

親セッション (このセッション) を「issue を配る側」に固定し、実作業は issue ごとの worktree で走る子 Claude セッションに任せるためのスキル。herdr は pane 内の agent の状態 (`idle` / `working` / `blocked` / `done`) を認識するので、親は各タブを手で覗かなくてもポーリングで詰まりを拾える。

## 前提チェック (最初に必ず)

```sh
test "${HERDR_ENV:-}" = 1
```

`HERDR_ENV=1` でなければ herdr 管理下の pane で動いていない。その場合は「herdr の外なので worktree タブを作れない」と伝えて中断する。herdr 外から socket を叩くと、どの workspace を親とみなすかが決まらず誤ったところにタブが生える。

親自身の位置は環境変数で分かる: `HERDR_WORKSPACE_ID` / `HERDR_TAB_ID` / `HERDR_PANE_ID`。

## スコープ: 起動元リポジトリのみ

**このスキルが触ってよいのは、起動元セッションと同じリポジトリの workspace だけ。** 他リポジトリの子には、起動も巡回も停止もしない。親はそのリポの文脈を持っていないので、何が起きているか判断する材料がない。

herdr は 1 画面に複数リポジトリを同居させ、`herdr agent list` は**全 workspace の agent を無差別に返す**。絞らずに使うと別プロジェクトの作業中セッションにプロンプトを投げたり落としたりできてしまう。

### 対象 workspace の求め方

```sh
main_repo=$(dirname "$(git rev-parse --git-common-dir)")
herdr worktree list --cwd "$main_repo"
```

`--cwd` を渡すと、そのリポジトリの worktree だけが返る (渡さないと**フォーカス中の workspace のリポジトリ**が対象になり、親がどの pane に居るかで結果が変わる。必ず明示する)。

**`--cwd` に渡す値は `git rev-parse --show-toplevel` では求めない。** 親自身がリンク worktree で動いていると、`--show-toplevel` は main working dir ではなく worktree 自身のパスを返す。それを渡した `herdr worktree create` は必ず `linked_worktree_source` エラーで落ちる ("New and open worktree actions start from the repo parent workspace.")。`worktree list` の方はどちらのパスでも通ってしまうため、list で動いた式をそのまま create に流用して踏んだ。**`--cwd` を取る herdr のコマンドは、このスキル内では全部 `$main_repo` を使う。**

親がリンク worktree で動くのは例外ではない。worktree を切って作業しているセッションからさらに子を起こすのが、このスキルの主な使われ方。

返る各 worktree の `open_workspace_id` が対象の workspace ID。このフィールドが無いものは herdr で開いていない worktree (`wt` などで作ったもの) なので、対象外として扱う。

これに親自身の `HERDR_WORKSPACE_ID` を加えた集合が、このスキルの守備範囲。

### 絞り込みの適用

- **巡回モード** — `herdr agent list` の結果を `workspace_id` が上の集合に入るものだけに絞る。範囲外の agent は状態も報告しない (「別リポで 3 つ動いています」も余計な情報)
- **停止モード** — 対象が範囲外なら実行せず、「別リポジトリ (`<repo_name>`) の workspace なので、そちらのセッションから操作してください」と伝えて止まる
- **起動モード** — `herdr worktree create --cwd "$main_repo"` が自リポに閉じるので追加の絞りは不要

**`SendMessage` はこの絞り込みを迂回する。** `ListAgents` は herdr と無関係に、**全リポジトリ・全プロジェクトのセッションを無差別に返す** (過去に立てたものも残る。数十行になる)。しかも `workspace_id` も `cwd` も返さないので、その出力だけでは自リポか判定できない。`SCN-250` のような名前は隣のリポジトリにも普通に存在する。

したがって、**送ってよい相手の集合は必ず `herdr agent list` 側で先に確定し、`ListAgents` は補助にしか使わない**。宛先確定の具体的な手順は巡回モード Step 2 に置く。

ユーザーが明示的に別リポの workspace ID を指定してきた場合だけは例外だが、そのときも「別リポですが本当に操作しますか」と 1 問確認する。issue ID の取り違えで隣のプロジェクトを消すのが一番ありがちな事故。

### 同一リポ内をグループで分ける

monorepo では 1 つのリポジトリに複数の取り組みが同居する。全部まとめて巡回すると、今どれが詰まっているのか読み取れなくなるので、タブ名に短縮キーを付けて分ける。

```
v2:ABC-123      ← グループ "v2"
lp:ABC-140      ← グループ "lp"
web:ABC-155     ← グループ "web"
ABC-99          ← prefix なし = 未分類
```

タブ名に持たせるのは、巡回・停止のたびに所属を引き直さずに済むから。`herdr tab list` の label を `:` で切るだけで分かる。

**グループの由来は Linear project とは限らない。** monorepo のサブディレクトリ (`web/` など issue tracker と対応しない区切り) や、ユーザーの頭の中にしかない区切り (「今週の緊急対応」) も同じ仕組みに乗る。project が付いていない issue も、Linear を使わないリポジトリも普通にあるので、**project から自動で決めようとしない**。

**短縮キーの決め方** — 対応表はこのスキルに書かない (公開リポジトリに非公開 project 名を焼き込まないため)。実行時に次の順で決める:

1. ユーザーが明示していれば (「web の ABC-155 立ち上げて」「v2 で」) それを使う — 確認不要
2. `herdr tab list` で既存タブの prefix 集合を取る (label の `:` より前)
3. issue に Linear project があれば、その名前から候補を作る (英数字トークンを 1〜2 語、小文字、記号なし)
4. 候補が既存 prefix に含まれていればそれを採用 — 確認不要
5. 決まらなければ、既存 prefix の一覧を見せて「どのグループに入れますか? (一覧から選ぶ / 新しいキーを書く / 分類しない)」と 1 問確認

同じグループに別のキーを付けると分割が壊れるので、候補を作る前に必ず既存 prefix を見る。

**グループが 1 つしか無いリポジトリでは prefix を付けない。** 既存タブに prefix 付きが 1 つも無く、ユーザーの指定も無ければ、そのまま `ABC-123` で作る。区切りが無いところに持ち込んでもタブ名が伸びるだけになる。

**適用**:

- **起動モード** — タブ label と agent 名の両方に prefix を入れる (`v2:ABC-123` / `v2-abc-123`)。agent 名は小文字と `-` だけなので `:` は使えない
- **巡回モード** — 引数に短縮キーがあればそれだけ (`patrol v2`)。無ければグループごとに分けて報告する。混ぜて一覧にしない
- **停止モード** — 対象の照合に prefix 込みの label を使う。`ABC-123` だけ言われた場合、同じ ID が複数グループに現れることは無いので prefix なしで照合してよい

## モード判定

- 引数が `patrol` / 「状況」「巡回」「様子」「blocked」等を含む → **巡回モード** (短縮キーが続けば、そのグループだけ)
- 「落とす」「閉じる」「片付ける」「終わり」「もういい」等 → **停止モード**
- 引数が issue 識別子、または「立ち上げて」「走らせて」等 → **起動モード**
- 判断がつかない → 1 問で確認する

---

## 起動モード

### Step 1: issue を特定する

引数の形で取得先が決まる。

| 形 | 例 | 取得先 |
|---|---|---|
| `^[0-9]+$` | `46` | GitHub (`gh`) |
| `^[A-Z]+-[0-9]+$` | `ABC-123` | Linear |
| GitHub issue URL | `https://github.com/o/r/issues/123` | GitHub (`gh`) |

**GitHub:**

```sh
gh issue view <N> --json number,title,body,labels
```

リポジトリは起動元の remote から自動解決される。URL 形式なら `--repo <owner/repo>` を付ける。

**Linear:**

Linear MCP が使えるならそれで取得する (本文をそのまま扱えるため)。無ければ `linear issue view <ID>` にフォールバックする。

取得に失敗したら (auth 切れ、issue 不在) そこで止めて原因をユーザーに見せる。推測した issue 内容で worktree を作ると、後から気づいたときにブランチごと捨てる羽目になる。

### Step 2: 3 つの名前を決める

herdr は用途ごとに別の名前空間を使うので、1 つの issue に対して 3 つ決める。制約が違うので使い回せない。

| 用途 | 例 (グループなし) | 例 (グループ `v2`) | 制約 |
|---|---|---|---|
| ブランチ名 | `fix/flaky-login-redirect-issue-ABC-123` | 同左 | git の制約のみ |
| タブ label | `ABC-123` | `v2:ABC-123` | 自由 (大文字可) |
| agent 名 | `abc-123` | `v2-abc-123` | **小文字始まり、`[a-z0-9_-]` のみ、1〜32 文字** |

グループ分けをするリポジトリかどうかは「同一リポ内をグループで分ける」の手順で決める。ブランチ名には prefix を入れない — グループは herdr 上の見え方の話で、git 側の履歴に持ち込む理由がない。

agent 名の制約は実際にエラーで弾かれる (`invalid_agent_name`)。issue ID をそのまま渡さず小文字化する。

**ブランチ命名規約** — 対象リポジトリの既存ブランチに合わせる。`git branch --list` で数本見れば規約が読める。読めない場合の既定は:

```
<type>/<kebab-slug>-issue-<ISSUE-ID>
```

- **type**: `feat | fix | refactor | docs | chore | test` を issue タイトルから推測 (Add/Implement → `feat`、Fix/Bug → `fix`、Refactor → `refactor`、Doc → `docs`)。推測がつかなければ `feat`
- **kebab-slug**: lowercase 3〜5 単語。日本語タイトルは英訳する
- **ISSUE-ID**: Linear は `ABC-123`、GitHub は番号のみ (`46`)

既定をそのまま使わず既存ブランチを見るのは、`wt-start` (手動作業用) と規約が食い違っているリポジトリが実在するため。同じリポジトリ内で形が混ざる方が害が大きい。

### Step 3: base とモデルを決める

**base** — 既定は default branch のリモート追跡参照。ローカル `main` は fetch 遅れで古いことがあるため使わない。

**ただし `origin/main` を指定するだけでは最新にならない。** リモート追跡参照は最後に fetch した時点で止まっている。親は issue を仕込み続けて長時間生きるので、親の起動が古いほど子は古い土台から作業を始める。**base を決める前に必ず fetch する。**

default branch が `main` でないリポジトリを踏む可能性があるので、あわせて検出する:

```sh
git fetch origin
git remote set-head origin --auto
default_branch=$(git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's|refs/remotes/origin/||')
base="origin/$default_branch"
base_sha=$(git rev-parse --verify "${base}^{commit}")   # Step 4 で見せ、Step 5-1 でそのまま渡す
```

`set-head --auto` を省くと default branch の変更に追従しない。**`git fetch` は `refs/remotes/origin/HEAD` を更新しない** — リモート側で default branch が `main` から `develop` に変わっても、fetch 後の `origin/HEAD` は `main` を指したままになる。

fetch は `--cwd` と違って**実行場所を選ばない**。`refs/remotes` は `.git` を共有するので、親がリンク worktree に居ても main working dir 側と同じ値になる。

**fetch か `set-head` に失敗したら止める。** ネットワーク断や認証切れを握り潰して古い base で子を立てると、子は最初から古い土台で作業し、後で作り直すことになる。「`<失敗したコマンド>` に失敗しました。手元の `<base>` (`<SHA>`) で立てますか」と 1 問確認する。

**`default_branch` が空なら `main` で代用せず止めて base を聞く。** default branch が `master` や `develop` のリポジトリで `main` を仮定すると、間違った土台で子が立つか、`main` が無くて `worktree create` が `invalid reference` で落ちる。どちらも黙って進んでよい状態ではない。

保証できるのは **fetch した時点まで**。承認待ちの間に default branch が進むことはあるが、そこで取り直さず承認時の `base_sha` で作る — 人間が見たものと違う土台で子が立つ方が害が大きい。だから Step 5-1 に渡すのはブランチ名ではなく `base_sha` (不変) にする。親は issue を続けて仕込むので、承認待ちの間に別の issue のための fetch が走って `origin/<default>` が進むのは普通に起きる。子が走り出した後に進むのも同じで、それは子自身が扱う。

**モデル** — `herdr agent start` は素の `claude` を起動するので、何も渡さないとアカウント既定のモデルになる。親がシェルのモデル選択で選んだモデルは子に伝播しない。issue ごとに明示する。

| モデル ID | 想定 |
|---|---|
| `claude-opus-5[1m]` | 既定。実装・設計・調査を伴う通常の issue |
| `claude-fable-5` | 速度と性能のバランスを取りたいとき |
| `claude-sonnet-5` | 文言修正・typo・docs 更新など、判断がほぼ要らない issue |
| `claude-haiku-4-5-20251001` | 機械的な一括置換レベル |

`[1m]` サフィックスは 1M context の指定。付け忘れると 200K で立つので、Opus 5 を選ぶときは必ず付ける。zsh の glob 文字なのでコマンド上は必ずシングルクォートで囲む。

親は issue の重さから 1 つ推奨を決め、Step 4 の承認ブロックに載せる。

### Step 4: 一度だけ承認を取る

worktree 作成・タブ生成・子セッション起動・プロンプト投入をまとめて 1 回で見せる。子が走り出すと止めるコストが高いので、投入する初回プロンプトの本文まで見せてから実行する。

```
issue:   ABC-123 / Add push notification opt-in
branch:  fix/push-notification-opt-in-issue-ABC-123
base:    origin/main @ 4a7c19e   (fetch 済み。この SHA で作る)
model:   claude-opus-5[1m]   (実装と設計判断を伴うため)
グループ: v2
タブ名 / セッション名:  v2:ABC-123   (agent 名: v2-abc-123)
Linear:  Backlog → In Progress   (現 status は issue 取得時の実測値)
初回プロンプト:
  <実際に投げる本文をそのまま>

これで起動しますか? (yes / モデルを変える / branch 名を変える / プロンプトを直す)
```

### Step 5: 実行

**5-1. worktree + workspace を作る**

```sh
main_repo=$(dirname "$(git rev-parse --git-common-dir)")
herdr worktree create \
  --cwd "$main_repo" \
  --branch <branch> \
  --base "$base_sha" \
  --label <ISSUE-ID> \
  --no-focus
```

`--cwd` は main working dir でなければならない (`--show-toplevel` を使うと親がリンク worktree のとき `linked_worktree_source` で落ちる。理由は「対象 workspace の求め方」)。

`--base` にはブランチ名ではなく Step 3 の `base_sha` を渡す。ブランチ名だと承認後に `origin/<default>` が進んだ場合、承認ブロックで見せた SHA と違う土台で作られる。`--base` は commit SHA をそのまま解決する。

`--no-focus` は親の視界を奪わないため。起動直後に画面が飛ぶと、次の issue を仕込む作業が中断される。作られた workspace はサイドバーで親リポの下にグループ表示され、配置は `~/.herdr/worktrees/<repo>/<branch-slug>` になる。

JSON から 3 つの ID を取る。ID は不透明な文字列なので、必ず JSON から読む — `wE:p1` のような形から自分で組み立てない:

```sh
.result.workspace.workspace_id   # 例 wE
.result.root_pane.tab_id         # 例 wE:t1
.result.root_pane.pane_id        # 例 wE:p1
```

**5-2. Linear の status を進める**

Linear 起点の issue なら、worktree ができた時点で status を更新する:

```sh
linear issue update <ISSUE-ID> -s "In Progress"
```

`--team` は不要 (issue ID から解決される)。同じ値を設定しても成功するので、既に In Progress でも気にしなくてよい。

これを入れているのは、**GitHub 連携の自動遷移が push 時にしか発火しない**ため。worktree を作って子を走らせた段階では push されていないので、Linear 上は Todo のまま残る。並行で 5 本も走らせると、Linear を見ても何に着手済みか分からなくなる。後で push されたとき自動遷移が走るが、同じ状態への遷移なので競合しない。

**子を起こす前に済ませる。** 起動の後ろに回すと、子が走り出した時点で「起動完了」と判断してこのステップごと落ちる。実際に worktree だけ生えて Linear が Backlog のまま残る事故が起きている。worktree が無いのに status だけ進む方が害が小さいので、順序はこちらを優先する。

Backlog の issue も対象。Todo を経由せず直接 In Progress にしてよい。

GitHub issue 起点の場合、issue に status の概念がないのでこのステップは飛ばす。

**5-3. タブに issue ID を付ける**

`worktree create --label` は workspace 名にしか効かず、タブ名は `1` のまま残る。タブ一覧で issue を見分けたいので明示的に rename する:

```sh
herdr tab rename <tab_id> <tab-label>
```

`<tab-label>` は Step 2 の表で決めたタブ label (グループがあれば `v2:ABC-123`、無ければ `ABC-123`)。以降 5-5 の `/rename` と巡回モードの宛先照合が、すべてこの文字列を使う。

**5-4. 子 Claude を起動する**

```sh
herdr agent start <agent-name> --kind claude --pane <pane_id> -- --model '<model-id>'
```

`--` より後ろが `claude` にそのまま渡る。承認済みのモデルをここで渡さないと、子はアカウント既定のモデルで立つ。`claude-opus-5[1m]` の `[1m]` は zsh の glob として展開されうるので、シングルクォートを外さない。

agent 名を付けておくと、以降の `agent prompt` / `agent read` / `agent get` をその名前で引ける。名前なしで手動起動された agent は `pane_id` でしか指定できず、巡回時の扱いが面倒になる。

pane が対話シェルプロンプトに居ないと失敗する。作りたての worktree pane なら通常は問題ない。失敗したら `herdr agent explain` で検出状態を見る。

**5-5. Claude Code のセッション名を付ける**

```sh
herdr agent prompt <agent-name> "/rename <tab-label>"
```

これは本命のプロンプトより先に送る。Claude Code はセッション名を未設定のままだと直近の作業内容を要約したタイトルを出し続けるので、サイドバーの表示が作業のたびに変わって issue と結びつかなくなる。`/rename` を通すとタイトルがそこで固定される。

**渡すのは 5-3 で付けたタブ label と同一の文字列にする** (グループがあれば `v2:ABC-123`、無ければ `ABC-123`)。大文字のままでよい — 小文字制約を持つのは agent 名だけ。ここを issue ID だけにすると、巡回モードが宛先を引くときにタブ label と一致せず、名前の衝突検知 (巡回 Step 2) も prefix の分だけ弱くなる。

**タイトルが変わるのを確認してから 5-6 に進む。確認を飛ばさない。** `agent prompt` を連続で撃つと、1 通目が送信される前に 2 通目が入力欄へ届き、**2 つが連結したまま未送信で残る**。`agent prompt` は成功を返し `agent_status` も `idle` なので、送信結果からは気づけない。issue 本文ごと未送信で残り、起動したつもりの子が何もしていない状態になる。

```sh
for i in $(seq 1 15); do
  t=$(herdr agent get <agent-name> | jq -r '.result.agent.terminal_title_stripped')
  [ "$t" = "<tab-label>" ] && break
  sleep 1
done
```

入力欄が汚れたら `herdr agent send-keys <agent-name> esc` で消してから送り直す。

**5-6. 初回プロンプトを投入する**

```sh
herdr agent prompt <agent-name> "<本文>"
```

`--wait` は付けない。親は制御を即座に取り戻して次の issue に進む必要がある。

ここは巡回モードと違って `SendMessage` に置き換えない。5-5 を通す前の子は自動生成名で登録されていて、タブ label では引けない。名前で引けない相手に送ると別の子に届く。

本文の組み立て方 — 子は issue の文脈を一切持っていないので、これだけで作業を始められる形にする:

```
<ISSUE-ID>: <title>

<issue 本文>

このリポジトリの worktree (branch: <branch>) で作業しています。
まず着手前に方針を 3〜5 行で整理して、そこで一度止まってください。

止まる基準:
- 既存の規約・コードを読めば決まることは、自分で決めて理由を 1 行残し、
  止まらず進めてください。
- 方針が分かれてどちらでも動くものは、あなたの推奨とその理由を書いて
  止まってください (オーケストレーターが判断して返します)。
- 設計文書や実装計画の承認を求める段階に来たら、行頭に [GATE] と書いて
  止まってください。これは人間の承認が要る箇所なので、代理承認されません。
```

**「迷ったら全部止まる」にしない**のが要点。何でも止める指示にすると `blocked` が量産され、親がそれを 1 件ずつ人間に上げることになって、並行で走らせた意味が消える。一方で「一度止まる」自体は残す — 子が黙って最後まで走ると、方針違いに気づくのが PR 段階になる。「止まらず最後までやって」と明示された issue なら、その行だけ外す。

### Step 6: 制御を返す

作ったものを 3 行で報告して終わる。`Linear:` 行は書き置きではなく、実測した値を載せる:

```sh
linear issue view <ISSUE-ID> --json | jq -r '.state.name'
```

```
ABC-123 起動: workspace wE / タブ ABC-123 / branch fix/...-issue-ABC-123
worktree: ~/.herdr/worktrees/<repo>/<slug>
Linear:   In Progress   ← 実測値
次: 続けて別の issue を起動するか、patrol で様子を見る
```

実測するのは、5-2 を飛ばしても定型文なら報告が成功に見えてしまうため。`In Progress` 以外が返ったら、その場で 5-2 をやり直してから報告する。

---

## 巡回モード

### Step 1: 全体の状態を取る

```sh
herdr agent list
```

返った agent を「スコープ」で求めた workspace 集合で絞る。

見るフィールド:

- `agent_status` — 状態 (下記)
- `name` — `agent start` で名付けたもの。**手動で立てたタブは空**になるので、その場合は `pane_id` を指定に使う
- `tab_id` — `herdr tab list` の label と突き合わせると issue ID が分かる
- `terminal_title_stripped` — Claude Code のセッション名。このスキルが起動した子は 5-5 でタブ label と同じ文字列に `/rename` してあるので、タブ label がそのまま入る。手動タブで `/rename` されていないものは直近の作業を要約したタイトルになり、何をしているかの手掛かりとして読める

`agent_status` の意味:

- `idle` — 入力待ち。タブは UI で確認済み
- `done` — 未確認のまま作業が終わった状態の idle。**見に行くべき筆頭**
- `blocked` — 承認 UI か質問を検出。**止まっている**
- `working` — 実行中。触らない
- `unknown` — agent は居るが分類できない

### Step 2: 要注意のものだけ状況を取る

`blocked` と `done` に絞る。`working` を触ると出力が途中で意味を成さず、トークンを食うだけになる。

取り方は 2 通りあるが、**等価ではない。片方は読み出しで、もう片方は子への書き込み。**

| | `herdr agent read` | `SendMessage` |
|---|---|---|
| 子への影響 | 無し (画面を写すだけ) | 子の会話に入る。コンテキストとトークンを使わせ、止まっていた子を再開させうる |
| コスト | 1 回 1,500〜3,000 トークン | 1 往復 100 トークン台。ただし `ListAgents` (実測 100 行 ≒ 端末読み 1 回分) が別途要る |

差は 10〜30 倍だが、それは対象が複数あるラウンドの話。`ListAgents` は 1 巡につき 1 回だけ引き、得た `[ref]` をそのラウンドの間持ち回る (Step 2 で引く必要が無ければ Step 4 で引く)。**対象が 1 件しか無いラウンドは端末を読む方が安い。**

#### `blocked` は端末を読む。子に送らない

ゲート (「ゲート: 代理承認しない停止」) かどうかの判定が、この時点でまだ済んでいないため。ゲートで止まっている子に親が接触すると、子は接触自体を進行の合図と読んで実装に入ることがある。判定前の探りでも同じ事故が起きるので、`blocked` への送信は Step 4 でゲートでないと確定した後に限る。

承認ダイアログで止まっている子がメッセージを受け取れるかも分かっていない。破棄されるのか、ダイアログに答えた後で遅れて割り込むのかが未確認で、後者なら親の質問が任意のタイミングで子の会話へ入る。

#### `done` は子に聞く (既定)

**宛先はこの 2 段で確定する。名前だけで送らない。**

1. Step 1 で絞った集合のうち、**このスキルが起動した子だけ**を残す。`herdr agent list` の `name` が起動モード Step 2 の表の agent 名と一致し、かつ `terminal_title_stripped` がタブ label と一致するもの (グループを付けないリポジトリなら prefix 無しの形で一致する)。どちらか欠けるのは手動で立てたタブなので、端末を読む方に回す
2. `ListAgents` を引き、その名前に**完全一致する `interactive` の行がちょうど 1 行**であることを確認する。次のどれかなら端末を読む方に切り替える (`pane_id` 指定なら一意に効く)
   - **一致が 0 行** — その名前では登録されていない (`/rename` が未反映 / 表示名が違う)
   - **一致するのが `Remote Control` の行だけ** — 子が古いバイナリで起動されている。ローカル配送は unix socket 経由で、対応していないプロセスには届かない。reply-only なので名前では送れない
   - **2 行以上** — 別リポジトリか過去セッションと名前が衝突している

`ListAgents` は herdr と無関係に全リポジトリ・過去分を返し、`workspace_id` も `cwd` も持たない。突き合わせの鍵は名前しか無いので、一致行が 1 行であることの確認が唯一の衝突検知になる。

確定したら行末の `[ref]` を読んで送る。

```
SendMessage  to: "<セッション名> [<ref>]"
```

別セッション宛の初回送信は `[ref]` が要る (誤爆防止の確認)。`ref` は不透明値なので issue ID から組み立てず、必ず `ListAgents` の出力から読む。

送る本文はこれで固定する:

```
状況確認です。回答だけ返してください。作業は再開しないでください。

1. 直近で変更したファイル
2. commit したか / 未 commit か
3. 次の 1 手
4. 止まっているなら、何を待っているか
```

冒頭の 1 行を落とさない。これが無いと、聞かれた「次の 1 手」をそのまま実行し始める。人間は様子を見ただけのつもりでいる。

項目を固定するのは、自己申告の精度が子の直前の作業で変わるため。長く実装に潜っていた子ほど要約が曖昧になり、親が一番知りたいこと (実は詰まっている) を落とす。項目が埋まらないこと自体を信号として使う。

#### 端末を読む (フォールバック)

次のどれかに当たるときは生ログを読む。

- 対象が `blocked` (上記)
- 宛先確定の 2 段を通らなかった — 手動タブ / 一致が 0 行 / 一致が `Remote Control` のみ / 2 行以上
- **そのラウンドの他の対象を処理し終えても返答が無い** (automode の「次の周回まで待つ」にしない。単発の `patrol` では次の周回が来ない)
- 返ってきた自己申告が要領を得ない

```sh
herdr agent read <agent-name-or-pane-id> --source recent-unwrapped --lines 120
```

無応答を放置しない。返事が来ないものを報告から落とすと、詰まっている子が一度も表に出ないまま残る。automode では静かに起き続ける。

### Step 3: 親が要約して報告する

読んだ生ログをそのまま貼らない。1 件あたり 2〜3 行に落とす:

```
[v2]
  ABC-123  blocked — DB マイグレーションの後方互換を壊してよいか確認待ち
  ABC-124  done    — 実装+テスト完了。PR 未作成
  ABC-125  working — 触らない

[lp]
  ABC-140  done    — 文言修正完了

[未分類]
  ABC-99   idle    — 手動で立てたタブ
```

グループが 1 つしか無ければ見出しを出さずに平らに並べる。手動で立てたタブも `agent list` に出るので、状態は報告に含めてよい。

### Step 4: 回答する

止まっている子への対応は 3 層に分かれる。**全部を人間に上げない**のがこのモードの目的で、上げるほど並行で走らせた利点が消えていく。

| 層 | 誰が決めるか | 該当するもの |
|---|---|---|
| 即決 | 親が調べて即答 | 命名、既存パターンの踏襲、テストの置き場所、規約で決まること |
| 相談 | 親がサブエージェントに諮って決定 | 実装方針の分岐、トレードオフのある技術判断、ライブラリ選定、性能とのバランス |
| ゲート | 必ず人間 | 「ゲート: 代理承認しない停止」を参照 |

**即決**は先にコードを読む。子が「この命名でいいか」と聞いてきたなら、答えは既存ファイルの中にある。読まずに一般論で返すと、リポジトリの実態と食い違った回答が入る。

**相談**は Agent ツールで諮る。親のコンテキストは他の子の状況で埋まっていて、1 件の技術判断に要る深さで読み込む余裕がないことが多い。判断の種類で相手を選ぶ — アーキテクチャなら `feature-dev:code-architect`、技術選定なら `shared:tech-lead`、レビュー観点なら `shared:principal-engineer`。

回答の投入先は 4 通りに分かれる。

- **ゲートと判定したもの** → 何も送らない (「ゲート: 代理承認しない停止」)
- **`done` で Step 2 に宛名が確定しているもの** → その宛名で `SendMessage`
- **`blocked` でゲートでないと判定できたもの** → **ここで Step 2 の「宛先を確定する」2 段を通してから** `SendMessage`。`blocked` は Step 2 で全件が端末読みに回っているので、宛名はまだ無い
- **2 段を通らなかったもの** (手動タブ / `interactive` に出ない / 一致行が 2 行以上) → `herdr agent prompt <agent-name-or-pane-id> "<回答>"`

**一度確定した宛名は同じラウンド内で引き直さない。** 引き直すと `ListAgents` の一覧から名前で選ぶことになり、2 段確認を素通りする。禁じているのは重複取得であって、`blocked` に対する 2 段の検証そのものではない。

`SendMessage` を先に置くのは、`agent prompt` が入力欄へのキー入力だからで、連投すると 2 通が連結して未送信のまま残る (5-5 参照)。巡回では複数の子に続けて返すので踏みやすい。`SendMessage` は会話に直接入るのでこの事故が起きない。

回答には**決定と理由を必ず添える**。「A でお願いします」だけだと、子は次に似た判断が来たときにまた止まる。理由が入っていれば同じ系統の判断を自分でできる。

複数の子が同時に blocked のときは 1 件ずつ順に片付ける。まとめて質問を人間に投げると、どの回答がどの子のものか取り違える。

### Step 5: 決めたことを記録する

「相談」層で下した決定は、次の報告でまとめて見せる。無人で進めた判断こそ、後から見て間違いに気づける形にしておく必要がある。

```
自動で判断して進めたもの:
  ABC-123  再試行はキュー側で持つ方針にした (呼び出し側に散らすと重複制御が効かないため)
  ABC-125  日付整形は既存の date-fns に寄せた (新規に依存を増やさないため)

人間の承認待ち (ゲート):
  ABC-124  [GATE] 実装計画のレビュー待ち — docs/plans/... を確認してください
```

### 自動巡回 (automode)

毎回手で `patrol` を打つのは面倒なので、組み込みの `/loop` に乗せる。専用の仕組みは作らない。

```
/loop 15m /herdr-orchestrate patrol
```

間隔は子の作業の粒度で決める。実装が数十分かかる issue が並んでいるなら 15〜30 分。短くしても `working` を数えるだけの回が増える。止めるときは `/loop` を停止する。

**無人であることが前提になるので、有人時と振る舞いを変える。**

- 「相談」層は**そのまま実行してよい** — サブエージェントに諮って決定し、根拠を添えて子に返す。決めた内容は次の報告でまとめて見せる
- **ゲートは保留キューに積む** — 無人で承認しない。子は `blocked` のまま残す
- **人間に 1 問確認する動作は使えない** — 有人時なら聞いていた判断は、相談層に落とすか保留キューに積むかのどちらかにする。「聞けないから適当に決める」はしない
- **停止モード相当の操作はしない** — worktree を消すのは取り返しがつかない。無人で走る巡回に混ぜない

保留キューは会話の中で持てばよく、ファイルに書き出す必要はない。次に人間が現れたターンで「承認待ちが 2 件あります」と出せれば足りる。

1 周して `working` しか無ければ報告を省略してよい。15 分ごとに「変化なし」と出力されても読む価値がない。

---

## ゲート: 代理承認しない停止

親が代理で答えてはいけない停止がある。**`enhance-superpowers` 系のスキルが生成した設計文書・実装計画の承認**がこれ。人間がレビューして通す前提で置かれた関門なので、親が「良さそうなので進めて」と返すと関門ごと無効になる。自動で通した設計の誤りは、実装が全部終わるまで表面化しない。

### 見分け方

二重で判定する。マーカーだけだと子が付け忘れたときに素通りする。

1. **マーカー** — 子の出力の行頭に `[GATE]`。初回プロンプトで付けるよう指示してある
2. **内容の検出** — マーカーが無くても、次のどれかに当たればゲート扱いにする
   - 実装計画・設計文書・仕様書のレビューや承認を求めている
   - `docs/` 配下に新規生成した文書について可否を聞いている
   - 「この方針で進めていいですか」が、1 つの判断ではなく**計画全体**に向いている

判断に迷ったらゲート側に倒す。人間に 1 回多く聞く方が、通してはいけないものを通すより安い。

### ゲートに当たったときの扱い

- **子には何も返さない** — `blocked` のまま残す。中途半端に「確認中です」と投げると、子がそれを承認と読んで進むことがある
- **automode 中なら保留キューに積む** — 無人で承認はしない
- **報告では他と分けて出す** — 「自動で判断して進めたもの」と同じ並びに置かない。人間が見るべきものが埋もれる

ゲートかどうかの判断自体は親がしてよい。判断を保留するのは「承認」だけ。

---

## 停止モード

子を落とすとき、消える対象は 3 段階ある。どこまで消すかは中身を見ないと決められないので、確認してから聞く。

| 操作 | workspace / タブ | worktree の中身 | ブランチ |
|---|---|---|---|
| `herdr workspace close <ws>` | 消える | **残る** | 残る |
| `herdr worktree remove --workspace <ws>` | 消える | 消える | **残る** |
| 上 + `git branch -D <branch>` | 消える | 消える | 消える |

コミット済みの作業はブランチに残るので、`worktree remove` までなら失われない。失われるのは `git branch -D` を足したときだけ。

### Step 1: 対象を特定する

issue ID で言われることが多いので、タブ label から workspace を引く:

```sh
main_repo=$(dirname "$(git rev-parse --git-common-dir)")
herdr worktree list --cwd "$main_repo"  # 自リポの worktree と workspace_id
herdr tab list                          # label (issue ID) から workspace_id を照合
```

引いた workspace が「スコープ」の集合に無ければ、そこで止めて別リポである旨を伝える。

### Step 2: 失うものを数える

3 点を確認する。この確認を飛ばすと、聞くべきかどうかの判断自体ができない。

```sh
herdr agent get <agent-name>                    # agent_status が working なら実行中
git -C <checkout_path> status --short           # 未コミットの変更
git -C <checkout_path> log --oneline @{u}..HEAD # 未 push のコミット (upstream 未設定なら origin/main..HEAD)
```

### Step 3: 状態を見せて 1 問で聞く

```
ABC-123 を落とします。現在の状態:
  agent:     idle
  未コミット: 2 ファイル (src/foo.ts, src/bar.ts)
  未 push:   3 commits

どこまで消しますか?
  1. タブだけ閉じる (worktree もブランチも残る / 後で開き直せる)
  2. worktree も消す (ブランチは残るので未 push の 3 commits は無事)
  3. ブランチごと消す (未 push の 3 commits が失われる)
```

`working` 中なら「実行中ですが止めますか?」を先に確認する。途中で pane を閉じると、子が書きかけのファイルがそのまま残る。

未コミット変更がある状態で 2 か 3 を選ばれた場合、`herdr worktree remove` は `--force` なしだと `dirty_worktree_requires_force` で拒否する。この拒否は保護なので、`--force` を機械的に付けない。拒否されたら未コミットの中身 (`git -C <path> diff`) を見せて、捨ててよいか改めて確認してから `--force` を足す。

### Step 4: 実行して報告する

```sh
# 1 の場合
herdr workspace close <ws>

# 2 の場合
herdr worktree remove --workspace <ws> [--force]

# 3 の場合 (2 に続けて)
git -C <repo-root> branch -D <branch>
git -C <repo-root> worktree prune
```

`worktree prune` は、herdr が消した checkout の登録が git 側に残ることがあるので付けておく。

### Step 5: Linear をどうするか

落とし方で扱いが変わる。

- **完了して片付ける** → 触らない。PR マージ時に GitHub 連携が Done へ進める
- **中断・放棄する** → `linear issue update <ID> -s "Todo"` で戻す。In Progress のまま放置すると、Linear 上は着手中に見えて次に拾えなくなる

どちらか判断がつかないときは Step 3 の確認と一緒に聞く。落とす理由は本人しか知らない。

---

## やらないこと

本文で理由を書いた禁止は繰り返さない (スコープ / ゲート / automode / 停止モードの各節を参照)。ここには本文に出てこないものだけ置く。

- **自分が作っていない workspace / タブを閉じる、プロンプトを投げる** — 同じリポジトリでも手動で立てたタブが混在している。明示依頼がない限り触らない。`herdr agent prompt` だけでなく `SendMessage` も同じ扱い (経路が違うだけで、届く先は同じ人の作業中セッション)
- **`ListAgents` の一覧から宛先を選ぶ** — 名前が一致しても別リポジトリの別人の作業かもしれない。宛先は必ず `herdr agent list` 側で確定させる (「絞り込みの適用」を参照)
- **worktree を `wt` で作る** — herdr の workspace に紐づかず、`herdr worktree list` から辿れなくなる。手動作業の `wt` 運用とは分ける
- **`In Progress` より先の status に進める** — `In Review` / `Done` は PR 作成・マージ時に GitHub 連携が自動で進める。親が先回りすると実態とずれる
- **`--force` を確認なしに付ける** — あれは git の保護であって、邪魔な警告ではない

## 失敗時

- `agent start` が `invalid_agent_name` で落ちた → 大文字か記号が混じっている。小文字化して再実行する
- `worktree create` がブランチ重複で落ちた → 既に走っている可能性が高い。`herdr worktree list` と `herdr tab list` で既存タブを探し、あればそこに合流する案をユーザーに出す
- 子が意図と違うモデルで立った → `-- --model` を渡し忘れている (アカウント既定が出る)。`herdr agent read` で確認し、落として起動し直す。`[1m]` がクォートなしで glob 展開されて消えている場合も同じ症状になる
- `agent start` がタイムアウト → pane はできている。`herdr agent explain` で検出状態を見せ、手動で claude を起動する選択肢を出す
- 起動したのに子が何もしていない (`Ctx: 0` のまま) → 5-5 の確認を飛ばして 5-6 を撃ち、入力欄で 2 通が連結して未送信で残っている。`herdr agent send-keys <name> esc` で消し、タイトル確認を挟んで送り直す
- `gh` / `linear` が無い → 起動モードは成立しない。issue 内容を直接ユーザーから受け取る形に切り替えてよいか 1 問で確認する
- `SendMessage` が `is not an agent in this conversation` で落ちた → `[ref]` が要る。エラーに候補と ref が出るので、それをそのまま宛名にして送り直す
- `SendMessage` が `reply-only` で落ちた → その子は古いバイナリで起動されていて名前では届かない。`herdr agent read` に切り替える (子を起動し直すまで直らない)
