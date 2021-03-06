---
layout: main
title: ドメインモデル
category: About
menu: menu_ja
toc:
- title: ドメインモデル
  url: "#ドメインモデル"
  active: 'true'
- title: ソースコード
  url: "#ソースコード"
- title: ステップ
  url: "#ステップ"
- title: コンテナ
  url: "#コンテナ"
- title: ジョブ
  url: "#ジョブ"
- title: ビルド
  url: "#ビルド"
- title: イベント
  url: "#イベント"
- title: メタデータ
  url: "#メタデータ"
- title: Workflow
  url: "#workflow"
- title: パイプライン
  url: "#パイプライン"
---

## ドメインモデル

![Definition](../../../about/appendix/assets/definition-model.png)
![Runtime](../../../about/appendix/assets/runtime-model.png)

### ソースコード

ソースコードとは、ビルドやテスト、アプリケーションのパブリッシュに必要なコードと`screwdriver.yaml` を含む、指定されたSCMリポジトリとブランチのことです。

### ステップ

A step is a named action that needs to be performed, usually a single shell command. In essence, Screwdriver runs `/bin/sh` in your terminal then executes all the steps; in rare cases, different terminal/shell setups may have unexpected behavior. If the command finishes with a non-zero exit code, the step is considered a failure. Environment variables will be passed between steps, within the same job.

### コンテナ

コンテナは隔離された環境で[ステップ](#%E3%82%B9%E3%83%86%E3%83%83%E3%83%97)を実行します。異なる環境・バージョンで動作しているコードの互換性について、同時に実行されている他の[ビルド](#%E3%83%93%E3%83%AB%E3%83%89)に影響を与えずにテストするために行われます。これはDockerコンテナを利用して実装されています。

### ジョブ

ジョブとは、順番が設定された複数の[ステップ](#%E3%82%B9%E3%83%86%E3%83%83%E3%83%97)を指定された[コンテナ](#%E3%82%B3%E3%83%B3%E3%83%86%E3%83%8A)で実行することです。一連のステップのうちいずれかが失敗すると、ジョブ全体は失敗されたとみなされ、以降のステップはスキップされます。（そのように設定されていない場合を除く）

実際のジョブでは[ソースコード](#%E3%82%BD%E3%83%BC%E3%82%B9%E3%82%B3%E3%83%BC%E3%83%89) の特定のコミットをチェックアウトし、求められた環境変数を設定し、指定された[ステップ](#%E3%82%B9%E3%83%86%E3%83%83%E3%83%97)を実行するという処理が行われます。

ジョブの実行中、実行される[ステップ](#%E3%82%B9%E3%83%86%E3%83%83%E3%83%97)では次の3つのコンテキストが共有されます。

- ファイルシステム
- [コンテナ](#コンテナ)
- [メタデータ](#メタデータ)

ジョブは[ソースコード](#%E3%82%BD%E3%83%BC%E3%82%B9%E3%82%B3%E3%83%BC%E3%83%89)への変更や[ワークフロー](#workflow)からのトリガーで自動的に開始します。UIから手動で開始することも可能です。

#### Pull Requests

Pull requests are run separately from existing pipeline jobs. They will only execute steps from the `main` job in the Screwdriver configuration.

#### 並列実行

環境変数のマトリックスを定義することでジョブの並列実行が可能です。通常これは複数の[コンテナ](#%E3%82%B3%E3%83%B3%E3%83%86%E3%83%8A)やテストの種類が必要な際に利用されます。

このジョブの例では4つの[ビルド](#%E3%83%93%E3%83%AB%E3%83%89)が並列に実行されます。

```yaml
image: node:{{NODE_VERSION}}
steps:
    test: npm run test-${TEST_TYPE}
matrix:
    NODE_VERSION:
        - 4
        - 6
    TEST_TYPE:
        - unit
        - functional
```

- `NODE_VERSION=4` and `TEST_TYPE=unit`
- `NODE_VERSION=4` and `TEST_TYPE=functional`
- `NODE_VERSION=6` and `TEST_TYPE=unit`
- `NODE_VERSION=6` and `TEST_TYPE=functional`

### ビルド

ビルドは実行中の[ジョブ](#%E3%82%B8%E3%83%A7%E3%83%96)ジョブのインスタンスのことを指します。すべてのビルドはユニークなビルド番号が振られています。また、各ビルドは[イベント](#%E3%82%A4%E3%83%99%E3%83%B3%E3%83%88)イベントに紐づいています。基本的なジョブ設定の場合、ジョブに対し一度に一つのビルドが実行されます。[ジョブマトリクス](#%E4%B8%A6%E5%88%97%E5%AE%9F%E8%A1%8C)が設定されていれば複数のビルドが並列に実行されます。

ビルドは次の5つのうちいずれかの状態を持ちます。

- `QUEUED` - ビルドはリソースの空きを待っています
- `RUNNING` - ビルドはexecutorで実行されています
- `SUCCESS` - 全てのステップが成功しました
- `FAILURE` - いずれかのステップが失敗しました
- `ABORTED` - ユーザーが実行中のビルドをキャンセルしました

### イベント

イベントはコミットや[パイプライン](#%E3%83%91%E3%82%A4%E3%83%97%E3%83%A9%E3%82%A4%E3%83%B3)の手動リスタートを表します。イベントには下記の2種類があります。

- `pipeline`: - パイプラインを手動で開始したりpull requestをマージしたりした場合に作られるイベントです。この種類のイベントは、パイプラインにおけるワークフローとしてのジョブと同じ順序でトリガーされます。(例: `['main', 'publish', 'deploy']`)
- `pr`:  - pull requestの作成や更新により作られるイベントです。この種類のイベントは`main`ジョブのみトリガーします。

### メタデータ

Metadata is a structured key/value storage of relevant information about a [build](#%E3%83%93%E3%83%AB%E3%83%89). Metadata will be shared with subsequent builds in the same [workflow](#workflow). It can be updated or retrieved throughout the build by using the built-in CLI ([meta](https://github.com/screwdriver-cd/meta-cli)) in the [steps](#%E3%82%B9%E3%83%86%E3%83%83%E3%83%97).

Example:

```bash
$ meta set example.coverage 99.95
$ meta get example.coverage
99.95
$ meta get example
{"coverage":99.95}
```

### Workflow

ワークフローとは、デフォルトブランチの`main`ジョブの[ビルド](#%E3%83%93%E3%83%AB%E3%83%89)成功後に実行される[ジョブ](#%E3%82%B8%E3%83%A7%E3%83%96)の順番のことです。ジョブは並列や逐次、またはその組み合わせで実行することができます。ワークフローにはパイプライン内に定義されたジョブがすべて含まれている必要があります。

ワークフロー内で実行されるジョブは次の内容を共有します。

- 同じgitコミットからチェックアウトされたソースコード
- Access to [metadata](#%E3%83%A1%E3%82%BF%E3%83%87%E3%83%BC%E3%82%BF) from a `main` build that triggered or was selected for this job's build

下記のworkflowセクションの例ではこのようなフローになっていて

```yaml
workflow:
    - publish
    - parallel:
        - series:
            - deploy-west
            - validate-west
        - series:
            - deploy-east
            - validate-east
```

pull-requestがmasterにマージされると次のように動作します。

- `main`が実行され、`publish`をトリガー
- `publish`は`deploy-west`と`deploy-east`を並列でトリガー
- `deploy-west`は`validate-west`をトリガー
- `deploy-east`は`validate-east`をトリガー

### パイプライン

パイプラインとは同じ[ソースコード](#%E3%82%BD%E3%83%BC%E3%82%B9%E3%82%B3%E3%83%BC%E3%83%89)を共有する[ジョブ](#%E3%82%B8%E3%83%A7%E3%83%96)ジョブの集合を表します。これらのジョブは[ワークフロー](#workflow)で定義された順で実行されます。

`main`ジョブはソースコードへの各種変更をビルドするものなので、全てのパイプラインに定義されている必要があります。
