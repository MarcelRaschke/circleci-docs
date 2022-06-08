---
layout: classic-docs
title: "Docker 実行環境の使用"
description: "Docker 実行環境で実行するジョブの設定方法を説明します。"
version:
  - クラウド
  - Server v3.x
  - Server v2.x
---

[custom-images]: {{ site.baseurl }}/ja/2.0/custom-images/
[building-docker-images]: {{ site.baseurl }}/ja/2.0/building-docker-images/
[server-gpu]: {{ site.baseurl }}/ja/2.0/gpu/


**プレフィックスが「 circleci/ 」のレガシーイメージは、 2021 年 12 月 31 日に[サポートが終了](https://discuss.circleci.com/t/legacy-convenience-image-deprecation/41034)**しています。 ビルドを高速化するには、[次世代の CircleCI イメージ](https://circleci.com/blog/announcing-our-next-generation-convenience-images-smaller-faster-more-deterministic/)を使ってプロジェクトをアップグレードしてください。
{: class="alert alert-warning"}

`docker` キーは、Docker コンテナを使用してジョブを実行するために、基盤テクノロジーとして Docker を定義します。 コンテナは、ユーザーが指定した Docker イメージのインスタンスです。設定ファイルで最初にリストされているイメージがプライマリコンテナ イメージとなり、そこですべてのステップが実行されます。 Docker を初めて使用する場合は、[Docker の概要](https://docs.docker.com/engine/docker-overview/)についてのドキュメントを確認してください。

Docker は、アプリケーションに必要なものだけをビルドすることで、パフォーマンスを向上させます。すべてのステップが実行されるプライマリコンテナを生成する Docker イメージは [`.circleci/config.yml`]({{ site.baseurl }}/ja/2.0/configuration-reference/) ファイルで指定します。

```yaml
jobs:
  build:
    docker:
      - image: cimg/node:lts
```

この例で、すべてのステップは、`build` ジョブの下に最初にリストされているイメージによって作成されるコンテナで実行されます。

スムーズに移行できるように、CircleCI は一般的な言語用のコンビニエンス イメージを Docker Hub で提供しています。 名前とタグの一覧については、「[CircleCI のビルド済み Docker イメージ]({{ site.baseurl }}/2.0/circleci-images/)」を参照してください。 Docker をインストールした Git を含む Docker イメージが必要な場合は、`cimg/base:current` をご使用ください。

### Docker イメージのベストプラクティス
{: #docker-image-best-practices }

- レジストリ プロバイダーのレート制限によって問題が発生した場合は、[認証済みの Docker プルを使用する]({{ site.baseurl }}/ja/2.0/private-images/)ことで解決できる可能性があります。

- CircleCI は Docker との提携により、ユーザーの皆さまが今後もレート制限なしで Docker Hub にアクセスできるようにしています。 2020 年 11 月 1 日時点では、いくつかの例外を除き、CircleCI を通じて Docker Hub からイメージをプルする際に、レート制限の影響を受けることはありません。 ただし、今後 CircleCI ユーザーにもレート制限が適用される可能性があります。 将来的にレート制限の影響を受けることのないよう、お使いの CircleCI 設定ファイルに [Docker Hub 認証を追加する]({{ site.baseurl }}/ja/2.0/private-images/)と共に、必要に応じてご利用の Docker Hub プランのアップグレードを検討することをお勧めします。

- `config.yml` のなかでは、イメージのバージョンを示す `latest` や `1` といった、いずれ変わる可能性の高いタグはできるだけ使わないでください。 例で示している通り、`redis:3.2.7` や `redis@sha256:95f0c9434f37db0a4f...` のように正確にバージョンとハッシュ値を使うのが適切です。 指し示すものが変わりやすいタグは、ジョブの実行環境において予想できない結果になる場合があります。  そういった変化しやすいタグがそのイメージの最新版を指し示すかどうかについて、CircleCI は検知できません。 `alpine:latest` と指定すると、1 カ月前の古いキャッシュを取得することもありえます。

- 実行中に追加ツールをインストールするために実行時間が長くなる場合は、これらのツールが事前にインストールされているカスタムビルドイメージの作成および使用をお勧めします。 詳細については、[カスタムビルドの Docker イメージの使用]({{site.baseurl}}/ja/2.0/custom-images/)を参照してください。

- [AWS ECR]({{ site.baseurl }}/ja/2.0/private-images/#aws-ecr) イメージを使用する場合は、`us-east-1` リージョンを使用することをお勧めします。 CircleCI のジョブ実行インフラストラクチャは `us-east-1` リージョンにあるので、同じリージョンにイメージを配置すると、イメージのダウンロードにかかる時間が短縮されます。

- プロジェクトをほとんどあるいはまったく変更していないのにパイプラインが失敗した場合は、Docker イメージが使用されているアップストリームで問題が生じていないか確認してみることをお勧めします。

Docker Executor の詳細については、[CircleCI を設定する]({{ site.baseurl }}/ja/2.0/configuration-reference/)を参照してください。

### 複数の Docker イメージを使用する
{: #using-multiple-docker-images }

ジョブのなかでは複数のイメージを指定することが可能です。 テストにデータベースを使う必要があったり、それ以外にも何らかのサービスが必要になったりする場合に、複数イメージの指定が役に立ちます。 **複数のイメージを指定して設定されたジョブでは、最初にリストしたイメージによって作成されるコンテナで、すべてのステップが実行されます**。 全てのコンテナが共通ネットワーク上で実行され、開放されるポートはいずれも[プライマリコンテナ]({{ site.baseurl }}/2.0/glossary/#primary-container)の `localhost` 上で利用できます。

```yaml
jobs:
  build:
    docker:
    # Primary container image where all steps run.
     - image: cimg/base:current
    # Secondary container image on common network.
     - image: cimg/mariadb:10.6
       command: [mongod, --smallfiles]

    steps:
      # command will execute in an Ubuntu-based container
      # and can access MariaDB on localhost
      - run: sleep 5 && nc -vz localhost 3306
```
Docker イメージは以下の方法で指定することができます。

- Docker Hub 上での イメージ名やバージョンタグ
- レジストリのイメージへの URL を使用

下記の例により、様々なソースからパブリックイメージを使用する方法を紹介します。

#### Docker Hub 上のパブリック CircleCI イメージ
{: #public-convenience-images-on-docker-hub }

  - `name:tag`
    - `cimg/node:14.17-browsers`
  - `name@digest`
    - `cimg/node@sha256:aa6d08a04d13dd8a...`

#### Docker Hub 上のパブリックイメージ
{: #public-images-on-docker-hub }

  - `name:tag`
    - `alpine:3.13`
  - `name@digest`
    - `alpine@sha256:e15947432b813e8f...`

#### Docker レジストリ上のパブリックイメージ
{: #public-docker-registries }

  - `image_full_url:tag`
    - `gcr.io/google-containers/busybox:1.24`
  - `image_full_url@digest`
    - `gcr.io/google-containers/busybox@sha256:4bdd623e848417d9612...`

`config.yml` ファイルで `docker:` キーを指定すると、デフォルトで Docker Hub と Docker レジストリ上のほぼすべてのパブリックイメージがサポートされます。 プライベートのイメージまたはレジストリを操作する場合は、[Docker の認証付きプルの使用]({{ site.baseurl }}/2.0/private-images/)」を参照してください。

### RAM ディスク
{: #ram-disks }

RAM ディスクは `/mnt/ramdisk` に配置され、`/dev/shm` を使用する場合と同様に[一時ファイル用ファイルシステム](https://ja.wikipedia.org/wiki/Tmpfs)として利用できます。 使用する `resource_class` でプロジェクトのコンテンツすべて (Git からチェックアウトされたすべてのファイル、依存関係、生成されたアセットなど) をまかなえるだけのメモリを確保できている場合、RAM ディスクを使用することでビルドを高速化できます。

RAM ディスクの最もシンプルな使用方法は、ジョブの `working_directory` を `/mnt/ramdisk` に設定することです。

```yaml
jobs:
  build:
    docker:
     - image: alpine

    working_directory: /mnt/ramdisk

    steps:
      - run: |
          echo '#!/bin/sh' > run.sh
          echo 'echo Hello world!' >> run.sh
          chmod +x run.sh
      - run: ./run.sh
```

### Docker のメリットと制限事項
{: #docker-benefits-and-limitations }

Docker にはもともとイメージのキャッシュ機能があり、[リモート Docker][building-docker-images] を介した Docker イメージのビルド、実行、パブリッシュを可能にしています。 開発しているアプリケーションで Docker を利用する必要があるかどうか、再確認してください。 アプリケーションが下記内容に合致するなら、Docker を使うと良いでしょう。

- 自己完結型のアプリケーションである.
- テストのために他のサービスが必要なアプリケーションである.
- アプリケーションが Docker イメージとして配布される ([リモート Docker]({{ site.baseurl }}/ja/2.0/building-docker-images/) の使用が必要)。
- `docker-compose` を使用したい ([リモート Docker]({{ site.baseurl }}/ja/2.0/building-docker-images/) の使用が必要)。

Docker を使うと、Docker コンテナのなかで可能な範囲の機能に実行が制限されることになります (CircleCI における \[リモート Docker\]\[building-docker-images\] の機能も同様です)。 たとえば、ネットワークへの低レベルアクセスが必要な場合や、外部ボリュームをマウントする必要がある場合は、`machine` の使用を検討してください。

コンテナ環境として `docker` イメージを使用する場合と、Ubuntu ベースの `machine` イメージを使用する場合では、下表のような違いがあります。

| 機能                                                                                    | `docker`         | `machine` |
| ------------------------------------------------------------------------------------- | ---------------- | --------- |
| 起動時間                                                                                  | 即時               | 30 ～ 60 秒 |
| クリーン環境                                                                                | はい               | はい        |
| カスタム イメージ                                                                             | はい <sup>(1)</sup> | いいえ       |
| Docker イメージのビルド                                                                       | はい <sup>(2)</sup> | はい        |
| ジョブ環境の完全な制御                                                                           | いいえ              | はい        |
| 完全なルート アクセス                                                                           | いいえ              | はい        |
| 複数データベースの実行                                                                           | はい <sup>(3)</sup>  | はい        |
| 同じソフトウェアの複数バージョンの実行                                                                   | いいえ              | はい        |
| [Docker レイヤーキャッシュ]({{ site.baseurl }}/2.0/docker-layer-caching/)                      | はい               | はい        |
| 特権コンテナの実行                                                                             | いいえ              | はい        |
| Docker Compose とボリュームの使用                                                              | いいえ              | はい        |
| [構成可能なリソース (CPU/RAM)]({{ site.baseurl }}/2.0/configuration-reference/#resource_class) | はい               | はい        |
{: class="table table-striped"}

<sup>(1)</sup> [カスタム Docker イメージの使用][custom-images] を参照してください。

<sup>(2)</sup> [リモート Docker][building-docker-images] を使用する必要があります。

<sup>(3)</sup> Docker で複数のデータベースを実行することもできますが、その場合、すべてのイメージ (プライマリおよびセカンダリ) の間で、基になるリソース制限が共有されます。 このときのパフォーマンスは、ご契約のコンテナ プランで利用できるコンピューティング能力に左右されます。

`machine` の詳細については、次のセクションを参照してください。

### Docker イメージのキャッシュ
{: #caching-docker-images }

ここでは、Docker 実行環境のスピンアップに使用する Docker イメージのキャッシュについて説明します。 これは、プロジェクトにおける新しい Docker イメージのビルドを高速化するために使用する機能である [Docker レイヤーキャッシュ]({{site.baseurl}}/ja/2.0/docker-layer-caching)には適用されません。
{: class="alert alert-info" }


Docker コンテナのスピンアップからジョブの実行までに要する時間は、複数の要因により変わることがあります。要因としては、イメージのサイズのほか、レイヤーの一部または全部が基盤となる Docker ホストマシンに既にキャッシュされているかどうかも影響します。

CircleCI イメージなどのより広く利用されているイメージほど、多くのレイヤーがキャッシュでヒットする可能性が高くなります。 よく使われる CircleCI イメージの多くで、同じ基本イメージが使用されています。 各イメージ間で大部分の基本レイヤーが同じなため、キャッシュがヒットする確率が高くなっています。

環境のスピンアップは新しいジョブごとに必要です (新規ジョブが同じワークフロー内にある場合でも、ジョブの再実行や 2 回目以降の実行の場合でも)。 CircleCI ではセキュリティ上の理由から、コンテナを再利用することはありません。 ジョブが終了すると、コンテナは破棄されます。 たとえ同じワークフロー内にある場合でも、ジョブが同じ Docker ホストマシンで実行される保証はありません。　 これは、キャッシュステータスが異なる可能性があることを意味します。

いかなる場合でも、キャッシュのヒットは保証されるものではなく、ヒットすればラッキーな景品のようなものと言えるでしょう。 そのため、すべてのジョブでキャッシュがまったくヒットしないケースも想定しておいてください。

つまり、キャッシュのヒット率は設定や構成で制御することはできません。[CircleCI イメージ](https://circleci.com/developer/ja/images)など、広く利用されているイメージを選択すれば、"環境のスピンアップ" ステップでレイヤーがキャッシュでヒットする可能性が高まるでしょう。

### 使用可能な Docker リソース クラス
{: #available-docker-resource-classes }

[`resource_class`]({{ site.baseurl }}/2.0/configuration-reference/#resource_class) キーを使用すると、ジョブごとに CPU と RAM のリソース量を設定できます。 Docker では、次のリソース クラスを使用できます。

| クラス      | vCPU | RAM   |
| -------- | ---- | ----- |
| small    | 1    | 2 GB  |
| medium   | 2    | 4 GB  |
| medium+  | 3    | 6 GB  |
| large    | 4    | 8 GB  |
| xlarge   | 8    | 16 GB |
| 2xlarge  | 16   | 32GB  |
| 2xlarge+ | 20   | 40 GB |
{: class="table table-striped"}

たとえば次のように設定します。

```yaml
jobs:
  build:
    docker:
      - image: cimg/base:current
    resource_class: xlarge
    steps:
    #  ...  other config
```