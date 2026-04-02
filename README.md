## 1. はじめに

特に、非エンジニアのPdMと、Devのチーム構成を想定し、専門知識がない方でも迷わずセットアップと開発フローを実践できるよう、具体的な手順にしました

## 2. チームの役割分担

AI-DLCでは、AIを主体としつつ、人間が重要な意思決定を担います。各役割の責任範囲を明確にしましょう。

| 役職 | 主な役割と責任 |
| --- | --- |
| PdM | - ビジネス要求の提示: プロダクトの目的や要求（Intent）をAIに伝える責任者。

- 仕様の承認: AIが生成した仕様書やユーザーストーリーをレビューし、ビジネス要求と合致しているか最終判断を下す。

- プロダクトの評価: AIが作成したプロトタイプを評価し、プロダクトの方向性を決定する。 |
| エンジニア | - 技術的レビュー: AIが生成した設計やコードを専門的な観点からレビューし、品質と実現可能性を担保する。

- 環境構築と維持: AIとチームが円滑に開発できる環境を構築し、メンテナンスする。

- 実装の主導: AIと協働しながら、技術的な実装をリードし、システムを完成させる。 |

## 3. 必要なツール

開発を始める前に、以下のツールを準備してください。

| ツール | 役割 | 担当者 |
| --- | --- | --- |
| Claude (Pro/API) | AIとの対話、コードやドキュメント生成の中心 | 全員 |
| Figma | デザイン、ワイヤーフレームの共有とレビュー | 全員 |
| Git & GitHub | ソースコードと生成物のバージョン管理 | エンジニア |
| Docker | 開発環境のコンテナ化（環境差異をなくすため） | エンジニア |

## 4.【エンジニア向け】開発環境構築手順

ここからは、エンジニア向けの具体的な構築手順です。2名のエンジニアが完全に同じ環境で作業できるよう、Dockerを使ったコンテナ化を前提とします。

**Step 1: プロジェクトの初期設定**

まず、すべてのソースコードとAI生成物を管理するための基盤をGitHub上に作成します。

```bash
# 1. プロジェクト用のディレクトリを作成
$ mkdir my-ai-app && cd my-ai-app

# 2. Gitリポジトリを初期化
$ git init

# 3. GitHubで新しいリポジトリを作成し、リモートリポジトリとして登録
# (GitHubの画面でリポジトリを作成後、以下のコマンドを実行)
$ git remote add origin <GitHubリポジトリのURL>

# 4. AI生成物を管理するディレクトリを作成
$ mkdir -p aidlc-docs/{inception,alpha-construction,inception-refining,beta-construction-design,beta-construction-implement}

# 5. このディレクトリ構造をGitにコミット
$ git add aidlc-docs
$ git commit -m "Initial commit for AI-DLC artifacts structure"
$ git push -u origin main
```

**Step 2: Slash Commandの定義**

次に、AI-DLCの各フェーズをClaudeに指示するための「Slash Command」を定義します。以下の内容で claude_slash_commands.json というファイルをプロジェクトのルートディレクトリに作成してください。

```json
{
  "/inception": {
    "description": "【PdM向け】ビジネス要求からIntentとユーザーストーリーを作成し、要件定義を開始します。",
    "prompt": "..."
  },
  "/alpha-construction": {
    "description": "【PdM/エンジニア向け】Inceptionの成果物を基に、動作するプロトタイプを作成します。",
    "prompt": "..."
  },
  "/inception-refining": {
    "description": "【PdM向け】プロトタイプのフィードバックを基に、Intentとユーザーストーリーを修正します。",
    "prompt": "..."
  },
  "/beta-construction-design": {
    "description": "【エンジニア向け】最終的な要件定義を基に、製品版の設計を行います。",
    "prompt": "..."
  },
  "/beta-construction-implement": {
    "description": "【エンジニア向け】設計書を基に、ReactとRuby on Railsで製品版のコードを実装します。",
    "prompt": "..."
  }
}
```

（注: promptの全文は長いため省略しています。完全なファイルは別途提供します）

このファイルをClaude Codeのワークスペース設定で読み込ませることで、チャット画面から / を入力するだけで各コマンドを呼び出せるようになります。

**Step 3: Dockerによる開発環境のコンテナ化**

環境をDockerで構築します。プロジェクトのルートに以下のファイルを作成してください。

```yaml
version: '3.8'
services:
  frontend:
    build:
      context: ./
      dockerfile: Dockerfile.frontend
    ports:
      - "5173:5173"
    volumes:
      - ./frontend:/app
    tty: true

  backend:
    build:
      context: ./
      dockerfile: Dockerfile.backend
    ports:
      - "3000:3000"
    volumes:
      - ./backend:/app
    tty: true

Dockerfile.frontend (React用)

Plain Text

FROM node:18
WORKDIR /app
RUN npm install -g vite
COPY frontend/package.json ./
RUN npm install
CMD ["npm", "run", "dev"]

Dockerfile.backend (Ruby on Rails用)

Plain Text

FROM ruby:3.1
WORKDIR /app
RUN gem install rails
COPY backend/Gemfile ./
RUN bundle install
CMD ["rails", "server", "-b", "0.0.0.0"]
```

セットアップコマンド:

```bash
# 1. フロントエンドとバックエンドのディレクトリを作成
$ mkdir frontend backend

# 2. Dockerコンテナをビルドして起動
$ docker-compose up -d --build
```

## 5.【PdM向け】開発の進め方

すべての作業はWebブラウザの中で完結します。

**利用ツール**

1.Claude Codeのチャット画面: エンジニアから共有されるURLにアクセス ここがAIとの対話のメインの場所です。

2.Figmaのプロジェクト画面: デザインの確認やフィードバックを行います。

**開発の始め方: Inceptionフェーズ**

プロダクト開発は、PdMがClaudeに「こんなものを作りたい」と伝えることから始まります。

1.ブラウザでClaude Codeのチャット画面を開きます。

2.チャット入力欄に / を入力すると、コマンドの候補が表示されます。

3.inception を選択し、Enterキーを押します。

4.AIが「このプロダクトのビジネス目標や、実現したい世界観を教えてください。」と質問してくるので、あなたの言葉で自由に答えてください。

あとはAIがファシリテーターとなって、対話しながら仕様書（Intent）やユーザーストーリーを作成してくれます。あなたは、その内容が作りたいものと合っているかを確認し、承認するだけです。

## 6. AI-DLC開発フローの実践

環境が整ったら、以下のフローに沿って開発を進めます。AIが各フェーズの進行役を務めます。
