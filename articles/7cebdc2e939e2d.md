---
title: "【セキュリティ対応】複数サブシステムで構成されるPythonプロジェクト(with AWS SAM)におけるパッケージ管理の改善"
emoji: "✒️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Python", "セキュリティ対策", "サプライチェーン攻撃"]
publication_name: "fusic"
published: true
---
昨今のサプライチェーン攻撃の多発を受けて、Pythonプロジェクトでパッケージ管理の見直しを行いました。
Pythonのパッケージ管理、ベスプラが分からない！と個人的に感じています。
Node.jsならpnpmが良いのかな、とかnpm/yarn/pnpm/bunの比較が分かりやすい…と思ったり。一方でPythonはpip、pipenv、poetry、uv、condaと選択肢が多い上にそれぞれの役割用途・できることが絶妙にまとまりがなくて併用もできてしまう。そしてPythonという言語を選択する理由がさまざまであるからこそ、これぞベスプラという情報が生まれにくいのかもしれません。
（勉強不足もありますが、それにしても奥の深さは間違いなくあるはず）

今回、AWS SAMで管理している複数サブシステムで構成されるプロジェクトにおいて、依存関係管理をuvベースに統一しました。
ただし「全部uvに置き換える」のではなく、**既存のSAMとの互換性を保ちながら、pyproject.tomlを起点にバージョンを固定する仕組み**を構築した内容を紹介します。
もっといい方法や気になる視点があればぜひコメントください！

## 最初にまとめ

- 運用の複雑性がもたらすリスクの軽減
  - **シンプルにすること**を第一にuvによるプロジェクトルートでのパッケージ管理 → 各サブシステムごとのrequirements.txtは手動編集せずに自動生成する仕組みを構築
- 依存関係のバージョン固定
  - uv.lockを生成して、**推移的依存も含めた全てのバージョンを固定** → uv exportでSAMビルド用のrequirements.txtを生成
- CIでの整合性チェック
  - 二重管理に変わりはないので、pyproject.toml → uv.lock → requirements.txtのチェーン全体を検証するCIを構築（**属人性の排除**）

## 前提
### AWS SAMのデフォルトのビルド方法

AWS SAMでPython Lambdaをビルドする際、`sam build`はLambda関数のソースディレクトリにあるrequirements.txtを自動検出し、そこに書かれたパッケージをpipでインストールしてLambdaパッケージに含める…というのがデフォルトの挙動のようです。
その気になれば任意のビルド方法を実現できそうですが、初期開発当時はそこまで気が回っておらずデフォルトで進めていました。
※とはいえ今検討し直しても、ビルド方法をカスタマイズするよりは、デフォルトの挙動に合わせてプロジェクト構成を整える方がシンプルで運用しやすいと判断するかもしれません。

https://docs.aws.amazon.com/ja_jp/serverless-application-model/latest/developerguide/serverless-sam-cli-using-build.html
> アプリケーションの依存関係は、マニフェストファイル (requirements.txt (Python) または package.json (Node.js) など) で指定するか、関数リソースの Layers プロパティを使用して指定します。Layers プロパティには、Lambda 関数が依存する [AWS Lambda レイヤー](https://docs.aws.amazon.com/lambda/latest/dg/configuration-layers.html)リソースのリストが含まれています。

### 多数のLambda関数群で構成するプロジェクト固有の運用

当該プロジェクトでは、役割の異なるサブシステムの中に複数のLambda関数handlerがある構成で、それぞれ必要なパッケージが異なります。
各Lambda関数群に必要最小限のパッケージだけを含めたいこともあり、各サブシステムのディレクトリ直下にrequirements.txtを置きたいという需要がありました。
一方で、ローカル開発環境においてはプロジェクトルートでパッケージを一元管理した方が圧倒的に楽です。いちいち各関数群ディレクトリに移動して依存関係をinstallして・・・は手間だし、複雑。

というわけで元々は各サブシステム直下に必要最小限のrequirements.txt + ルートに開発に必要なライブラリも含めたrequirements.txtを置いてpip × venvで運用していました。
この時点でいろいろと分かりづらかったりプロジェクトが進むにつれて別の需要が生まれたりして、なんやかんやあってローカル開発環境・CI環境の一部をuvに移行したという経緯があります。
ここでルートのパッケージ管理はpyproject.tomlに統一されていきます。（ややこしい話、pyproject.tomlを置かざるを得ない状況でrequirements.txtとpyproject.tomlがルートに共存していました）

※ 詳細は省略します。

諸々省いていますが、イメージ図はこんな感じ。
```
my-project/
├── pyproject.toml
├── requirements.txt
├── hoge/
│   ├── requirements.txt
│   ├── function-a.py
│   └── function-b.py
└── fuga/
    ├── requirements.txt
    ├── function-a.py
    └── function-b.py
```

## 起きていた問題
uvに移行したことで開発環境に関してはだいぶシンプルになってきました。
とはいえセキュリティ観点・運用面ではまだまだ問題は残ったままでした。

### 1. 推移的依存のバージョンが固定されていなかった

※ここで記述しているバージョンはあくまで例示のため、最新性は考慮していません。

既存のrequirements.txtには直接依存のみが書かれていました。

```
aws_lambda_powertools == 2.41.0
beautifulsoup4 == 4.12.3
```

直接依存のバージョンは固定されていますが、aws-lambda-powertoolsが内部で依存している`jmespath`や`typing-extensions`などの**推移的依存はバージョン指定がありません**。つまり`sam build`を実行するタイミングによって、推移的依存のバージョンが変わる可能性がありました。

これは昨今話題のセキュリティリスクでもあります。ある日突然、推移的依存のパッケージに脆弱性のあるバージョンが入る、あるいはサプライチェーン攻撃で改竄されたパッケージが混入する、といったシナリオが考えられるのです。

### 2. pyproject.tomlとrequirements.txtの乖離

ローカル開発ではuvとpyproject.tomlを使い、デプロイ時はrequirements.txtを使う、という二重管理になっていました。パッチ適用管理ができておらず、当然ながらバージョンの乖離が発生していました。

```
# pyproject.toml
"aws-lambda-powertools>=2.41.0"

# hoge/requirements.txt（古いまま）
aws_lambda_powertools == 2.26.0
```

### 3. パッケージ追加時の手順が不明確

上記の問題に追従して、パッケージを追加・更新したい場合に「pyproject.tomlを直すのか、requirements.txtを直すのか、両方直すのか」が曖昧で、どちらかの更新漏れが起きやすい状態でした。

## 解決策：pyproject.toml → uv.lock → requirements.txt

### 基本方針

**pyproject.tomlをsingle source of truth（唯一の信頼の源）にする。** という案です。
pyproject.tomlを起点にして全てをコントロールできるようにしたい、と考えて以下の構成を練りました。

```
pyproject.toml          ← 開発者が編集する唯一のファイル
    │
    ▼  uv lock
uv.lock                 ← 推移的依存を含む全バージョンの完全固定
    │
    ▼  uv export
requirements.txt × N    ← sam build用（Lambda関数ごとに生成）
    │
    ▼  sam build
Lambdaパッケージ         ← 固定されたバージョンでビルド
```

requirements.txtは手動編集せず、uv.lockから自動生成する仕組みです。
### pyproject.tomlの定義方法：dependency-groupsによるLambda関数群ごとの依存分離

Lambda関数群ごとに必要最小限のパッケージを使いたいという需要に対して、この方法を採用しました。
pyproject.tomlの`[dependency-groups]`を使い、Lambda関数群ごとに依存を定義します。

```toml
[project]
dependencies = []

[dependency-groups]
# hogeサブシステムLambda群の依存
hoge = [
    "aws-lambda-powertools>=2.41.0",
    "pydantic>=2.9.2",
]
# fugaサブシステムLambda群の依存
fuga = [
    "aws-lambda-powertools>=2.41.0",
    "beautifulsoup4>=4.12.3",
    "lxml>=5.2.2",
]
# 開発・テスト用
dev = [
    "boto3>=1.34.131",
    "pytest>=8.0.0",
    # ...
]

[tool.uv]
default-groups = ["hoge", "fuga", "dev"]
```
※ boto3はLambdaのPythonランタイムでは標準で含められているので、ローカル開発環境のみの利用として定義しています。

`[project] dependencies`は空にし、全パッケージをグループに分類しました。`default-groups`を設定することで、`uv sync`実行時に全グループがインストールされ、ローカル開発の利便性は維持されます。

### uv exportによるrequirements.txt生成

`uv export --only-group <group>`で、指定グループの依存だけを推移的依存含めて出力できます。

```bash
# hoge系Lambdaの依存だけをrequirements.txtとして出力
uv export --frozen --no-hashes --only-group hoge --no-emit-project \
  -o path/to/hoge/requirements.txt
```
※`--no-hashes`は模索中です。sam build時の不整合等を防ぐため、いったんパッチバージョン固定までにしています。

生成されるrequirements.txt：

```
annotated-types==0.7.0
    # via pydantic
aws-lambda-powertools==3.23.0
aws-xray-sdk==2.15.0
botocore==1.42.16
    # via aws-xray-sdk
certifi==2025.11.12
    # via requests
jmespath==1.0.1
    # via aws-lambda-powertools, botocore
pydantic==2.12.5
# ...（推移的依存が全てバージョン固定で列挙される）
```

以前は直接依存の4行だけだったものが、推移的依存を含むものになりました。いつ`sam build`を実行しても、同じパッケージセットがインストールされます。

### 生成スクリプト

uv exportを手動で実施するのは面倒なので、全グループ分のrequirements.txtを一括生成するスクリプトを用意しました。pyproject.tomlの変更からrequirements.txtの生成までこのスクリプト1つで完結します。

```bash
#!/bin/bash
# export_requirements.sh

uv lock  # pyproject.tomlからuv.lockを更新

uv export --frozen --no-hashes --only-group hoge --no-emit-project --quiet \
  -o path/to/hoge/requirements.txt

uv export --frozen --no-hashes --only-group fuga --no-emit-project --quiet \
  -o path/to/fuga/requirements.txt

# ... 他のグループも同様
```

開発者の手順は以下の3ステップに簡略化されました。

1. `pyproject.toml`の該当グループを編集
2. スクリプトを実行
3. （必要に応じて）コミットしてPR作成

## CIでの整合性担保

ここまでやっても、スクリプト実行はやはり手動です。まだまだ属人性が高い。
そこでCIです。必要な手順の実行し忘れや手動編集による不整合を防ぐため、GitHub Actionsで整合性チェックを行います。

```yaml
# .github/workflows/check-requirements.yml
steps:
  - name: Check uv.lock is up to date with pyproject.toml
    run: |
      if ! uv lock --check; then
        echo "::error::uv.lock is out of sync with pyproject.toml."
        exit 1
      fi

  - name: Regenerate requirements.txt
    run: |
      uv export --frozen --no-hashes --only-group hoge ...
      uv export --frozen --no-hashes --only-group fuga ...
      # ... 各グループを出力

  - name: Check for diff
    run: |
      if ! git diff --exit-code -- '**/requirements.txt'; then
        echo "::error::requirements.txt is out of sync with uv.lock."
        exit 1
      fi
```

これはチェーン全体を検証するものです。
[概要図]
```
pyproject.toml → uv.lock → requirements.txt
                    ↑              ↑
              uv lock --check    git diff --exit-code
```

- `uv lock --check`: pyproject.tomlの内容に対してuv.lockが最新かを検証
- `git diff --exit-code`: CIで再生成したrequirements.txtとコミット済みのものに差分がないかを検証

どちらかに不整合があればCIがfailし、`scripts/export_requirements.sh`の実行を促すエラーメッセージが表示されます。CIは検証のみで修正コミットは行わないので、開発者が差分を確認してからコミットする安全なフローになっています。（修正も自動化できたら良いなと思いますが、悩ましい。。）

## まとめ

今回の対応でパッケージ管理・運用をシンプルにまとめて、セキュリティ観点での懸念を減らすことができました。複雑性そのものがリスクになりうるので、そこを改善できたのは良かったと思います。さらに推移依存までバージョン固定するなどの対応もできたので。
どのツールが良いか、結局何がベストプラクティスか、というのはまだまだ検討の余地があるかもしれません。しかし、現状のセキュリティリスクを減らすための改善策(複雑性の排除, 依存関係のバージョン固定)としては、今回の対応は十分に意味のあるものになったと感じています。
