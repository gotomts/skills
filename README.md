# skills

自分が使っている Claude Code の自作 skill を置いている場所。
日々の開発で繰り返し踏む手順を、Claude 側に手順として持たせるために書いている。
自分用なので、配布やインストールを想定した作りにはしていない。

配布を前提にしたスキル + エージェント集は、plugin marketplace として [gotomts/claude-collections](https://github.com/gotomts/claude-collections) に置いている。

## 一覧

| skill | 何をするか | 起動のきっかけ | 引数 |
|---|---|---|---|
| [conflict-resolver](conflict-resolver/SKILL.md) | Git のコンフリクトを、戦略選定から解消・検証・レポート出力まで対話で進める。解消時に「ついでの修正」を混ぜないようスコープを縛る | 「コンフリクト解消して」「PR がコンフリクトしている」 | `<PR番号 or ブランチ名（省略可）>` |
| [herdr-orchestrate](herdr-orchestrate/SKILL.md) | issue ごとに worktree と子 Claude セッションを立ち上げ、巡回して詰まりを解消し、不要になった worktree を片付ける | 「ABC-123 立ち上げて」「巡回して」 | `[<issue-id> \| patrol [<group>] \| stop <issue-id>]` |
| [herdr-succession](herdr-succession/SKILL.md) | コンテキストが逼迫したセッションを、引き継ぎ文書の作成から後任の起動・状態照合まで通して渡し切る | 「引き継ぎたい」「コンテキストが限界」 | `[self \| child <tab-label>]` |
| [linear-next](linear-next/SKILL.md) | Linear の未完了 issue を、依存関係・現在のブランチ・中断メモと突き合わせて、次に着手すべきものを推奨順で出す | 「次に何やる？」「Linear 確認して」 | `[--epic <issue-id> \| --all]` |
| [wt-start](wt-start/SKILL.md) | issue ID・URL・自由テキストのいずれかから branch 名を提案し、承認後に worktree とブランチを同時に作る | 「worktree 切って」「issue を worktree で開始」 | `[<linear-id> \| <github-url> \| <free-text>]` |
