# claude-workbench — Claude Code を Docker / tmux で運用

**MacBook → SSH → Ubuntu (ホスト tmux 1つ) → Docker Compose → Claude Code**

`claude-personal` は独立した Git リポジトリです。
**サブスクと Bedrock の認証・履歴・共通設定を分けません。** 切替は `.env` の `CC_PROVIDER` だけです。

## ディレクトリ構成

```text
~/claude/
├── claude-workbench/           ← 実行環境 (Git repo #1)
│   ├── .env                    ← 接続先を切り替える設定（Git管理外）
│   ├── .env.example
│   ├── compose.yaml
│   ├── .devcontainer/
│   │   ├── Dockerfile          ← Python/uv・Node・Go はコメントインで追加
│   │   └── devcontainer.json
│   ├── bin/cc                  ← Ubuntu ホスト側のコマンド
│   └── scripts/cc-entry        ← コンテナ内のコマンド
├── claude-personal/            ← カスタマイズ (Git repo #2)
│   ├── CLAUDE.md
│   ├── settings.json           ← サブスク／Bedrock 共通
│   ├── skills/
│   ├── agents/
│   └── rules/
├── START_HERE.md
└── CHANGELOG.md
~/workspaces/my-project/        ← 開発対象 (任意の Git repo)
```

## 1. 初期セットアップ（Ubuntu）

ZIP ファイルを **ホームディレクトリに展開**します。

```bash
cd ~
unzip claude-code-workbench-nested.zip -d ~
```

展開後のパスは `~/claude/claude-workbench` と `~/claude/claude-personal` です。
`~/claude` 自体は Git リポジトリにはせず、2つの子ディレクトリを個別に Git 管理します。

Docker Engine + Compose Plugin、Git、tmux、unzip をインストールします。
必要であれば既存プロジェクトを `~/workspaces/my-project` に clone します。

```bash
cd ~/claude/claude-workbench
cp .env.example .env
```

`.env` を開き、実際の**絶対パス**、`USER_UID=$(id -u)` と `USER_GID=$(id -g)` の数値を設定します。初期例:

```dotenv
COMPOSE_PROJECT_NAME=claude-myproject
PROJECT_DIR=/home/YOUR_USER/workspaces/my-project
PERSONAL_DIR=/home/YOUR_USER/claude/claude-personal
USER_UID=1000
USER_GID=1000
CC_PROVIDER=subscription
CC_MODEL=sonnet
AWS_PROFILE=bedrock
AWS_REGION=ap-northeast-1
```

```bash
docker compose config
docker compose up -d --build
docker compose ps
```

Docker の `vscode` ユーザーの UID/GID は、ホストのユーザーの数値と合わせます。Dockerfile はベースイメージの現在の ID を先に調べ、**既に一致する場合は変更しません**。古い Ubuntu 24.04 イメージなどで `ubuntu` ユーザーの UID/GID が希望値と重なる場合のみ、`ubuntu` の数値 ID を空き番号に移動して `vscode` に割り当てます。`ubuntu` ユーザー自体は削除しません。ほかのユーザーと衝突した場合は勝手に変更せず、理由を表示してビルドを停止します。

VS Code が別の UID/GID へ再変更しないよう、`devcontainer.json` に `"updateRemoteUserUID": false` を設定しています。ユーザー ID は `.env` の `USER_UID` / `USER_GID` に統一してください。

ベースイメージ内のユーザーを事前に確認するコマンド：

```bash
docker run --rm mcr.microsoft.com/devcontainers/base:ubuntu24.04 \
  sh -c 'id vscode; id ubuntu || true'
```

既に旧バージョンのコンテナを使用中の場合、ID を変更するビルドによって既存 Volume のファイル所有権が変わる可能性があります。**認証・会話履歴をバックアップし、現在のコンテナ内の `id vscode` と `.env` の UID/GID を確認してから更新**してください。

## 2. 切替方法（重要）

**`claude-workbench/.env` の1行だけを変更します。**

```dotenv
CC_PROVIDER=subscription
```

または:

```dotenv
CC_PROVIDER=bedrock
```

次回 `bin/cc` を実行した時点から反映されます。**`docker compose up` や再ビルドは不要**です。
コンテナには workbench ディレクトリを読み取り専用マウントしており、`cc-entry` が毎回 `.env` を読み直します。
起動中の Claude Code は切り替わりません。いったん終了して新しく起動してください。

```bash
cd ~/claude/claude-workbench
./bin/cc                  # .env の接続先＋モデル
./bin/cc --model opus     # モデルだけ今回上書き
./bin/cc --resume         # 保存済みセッションを選んで再開
```

`CC_MODEL` も `.env` にあります。`--model` は起動時だけの指定です。

**すべてのプロバイダーで共通**:
- `/home/vscode/.claude` (Claude Code の認証・会話履歴・状態) : Docker Volume `claude-state`
- `/opt/claude-personal` (Skills・Agents・Rules・settings.json) : Git 管理ディレクトリ
- `/workspace` (プロジェクト) : Ubuntu ホストから bind mount
- `/home/vscode/.aws` (AWS CLI・SSO) : Docker Volume `aws-state`

元のような `profiles/sub`、`profiles/bedrock`、`.claude-bedrock` はありません。
Claude Code の会話履歴は共有されますが、プロバイダーを変えた後の `--resume` が完全に同じ挙動を保証するものではありません。

## 3. サブスクリプションの認証

`.env` を `CC_PROVIDER=subscription` にし、`./bin/cc` を起動して Claude のサブスクアカウントにログインします。
**`ANTHROPIC_API_KEY` があると API 利用料金が優先される場合があります。**
`cc-entry` はこれを起動プロセスから解除しています。

## 4. Bedrock の認証（SSO の例）

AWS 側で使用するモデルへのアクセス権限と IAM 権限が必要です。

```bash
cd ~/claude/claude-workbench
docker compose exec workspace aws configure sso --profile bedrock
docker compose exec workspace aws sso login --profile bedrock --use-device-code
docker compose exec workspace aws sts get-caller-identity --profile bedrock
```

`.env` の `AWS_PROFILE` / `AWS_REGION` を設定し、`CC_PROVIDER=bedrock` に変更して `./bin/cc` を起動します。
SSO が切れたときは `aws sso login` を再実行します。AWS プロファイルの名前は例です。

**注意:** `/setup-bedrock` ウィザードは `settings.json` に接続先設定を保存するため使わず、上記の AWS CLI と `CC_PROVIDER` で切り替えます。また、`claude-personal/settings.json` やプロジェクトの `.claude/settings.json` に `CLAUDE_CODE_USE_BEDROCK` を入れないでください。

## 5. 個人設定と Git

コンテナ起動時に、ひとつの `/home/vscode/.claude` 内へ以下をリンクします。

```text
CLAUDE.md    -> /opt/claude-personal/CLAUDE.md
settings.json -> /opt/claude-personal/settings.json
skills/       -> /opt/claude-personal/skills/
agents/       -> /opt/claude-personal/agents/
rules/        -> /opt/claude-personal/rules/
```

Git 管理は `~/claude/claude-personal` 側で行ってください。既に Volume 内に同名の実ファイルが存在する場合、**自動上書きせずエラー**にします。必ずバックアップし、必要な差分を取り込んでから、既存の実ファイルを退避して再実行してください。

## 6. tmux と VS Code

```bash
ssh ubuntu-host
tmux new -A -s myproject
cd ~/claude/claude-workbench
./bin/cc
```

`Ctrl+b` → `d` で切断。ホスト上の tmux だけで十分です。
VS Code の Remote - SSH で Ubuntu に接続後、Dev Containers でこの workbench を開けます。
VS Code の統合ターミナルから起動するときは、`/usr/local/bin/cc-entry` を使用すると `.env` の設定が反映されます。
**VS Code の Claude Code GUI 拡張のプロバイダー選択には、このラッパーは自動適用されません。**

## 7. 更新・バックアップ・移行

Dockerfile の変更時は、セッションを終了してから:

```bash
cd ~/claude/claude-workbench
docker compose up -d --build
```

**`docker compose down -v` は認証・会話履歴を削除するため使わないでください。**

### 以前の2 Volume 方式から移行する場合

以前の ZIP を実際に動かしていた場合、新しい `claude-state` は**新規 Volume**です。
`claude-sub-state` の認証・履歴は自動移行されません。移行する場合はまず旧 Volume を必ずバックアップし、**旧コンテナを停止してから**新 Volume にデータをコピーしてください。
従来の `.claude` に `settings.json` / `CLAUDE.md` / `skills` などのリンクがある場合は、旧プロファイル特有の `settings.json` のリンクを取り込まず、共通 `claude-personal/settings.json` と差し替えてください。
旧 Bedrock 側の履歴も一つに統合する場合は、同名ファイル衝突の検査が必要です。無条件の `cp -a` で上書きしないでください。

### セキュリティ・制約

- `.env` はシェルとして読み込むため、**自分で作成した信頼できるファイルだけ**を使用し、秘密情報は記載しないでください。
- 個人設定の Git リポジトリはコンテナへ RW マウントされています。必要なら `compose.yaml` で RO にしてください。
- `/workspace` の編集は Ubuntu ホスト側にも残ります。Git worktree を使うと作業の分離ができます。
- ホストの `~/.ssh` や `~/.aws` を無条件にマウントしないでください。
- Docker コンテナの停止・再生成では、対話中の Claude Code プロセスが終了します。保存した履歴は `./bin/cc --resume` で再開を試せます。
- このテンプレートは Docker Engine 上のビルドおよび実際の AWS 認証まで検証していません。
