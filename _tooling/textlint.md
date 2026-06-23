---
title: textintを翻訳の校正に使う
toc: true
excerpt: "`textlint`は、モノリンガルの校正ツールです。この記事では、`textlint`を翻訳作業で活用する方法を紹介します。"
---


## textlintの概要

`textlint`は、様々な言語をサポートし、校正のフレームワークを提供するFOSSのツ
ールです。校正ルールは別途用意されており、各ルールを組み合わせて校正できます。
校正に使うルールには、ユーザ定義の辞書で拡張できるものもあります。
[textlintのWikiページ](https://github.com/textlint/textlint/wiki/Collection-of-textlint-rule)
には、日本語を含む各言語に対応したルールの一覧があります。

一般的なCATで使える校正ルールは、言語ごとの事情が考慮されていない、デフォルト設
定がターゲット言語に適切ではない、そもそもルールが貧弱、などの欠点があります。
また、翻訳が終わったあとにまとめてチェックするという思想なので、翻訳者の作業を
助けるという観点では、あまり親切とはいえません。翻訳者としては、訳文を入力した
時点で問題を指摘してほしいのです。こうした欠点を補うために`textlint`が使えます。

`textlint`の特徴は、文単位ではなく、文書の文脈によって校正できる点です。例えば、
[本文では敬体を使い、見出しや箇条書きでは常体を使う](https://github.com/textlint-ja/textlint-rule-no-mix-dearu-desumasu)、
[見出しの構造をチェックする](https://github.com/matsu-chara/textlint-rule-incremental-headers)
といったチェックができます。

もう一つの特徴は、正規表現だけでなく、形態素解析によるチェックもサポートしてい
ます。形態素解析を使ったルールには、漢字の読みや文字から受ける印象を考慮し一部
をひらがなで表記するための
[textlint-rule-ja-hiraku](https://github.com/akiomik/textlint-rule-ja-hiraku)
などがあります。日本語の場合は、形態素解析がないと書くのが実質的に無理なルール
もありますので、一般的な正規表現ベースの校正ツールではできないチェックも可能で
す。JavaScriptの形態素解析には、
[kuromoji.js](https://github.com/takuyaa/kuromoji.js/)
を使うのが主流のようです。

ちなみにkuromoji.jsでは、Pythonの
[Spacy](https://spacy.io/)
のように、係り受けの関係は解析しないようです(ある名詞を修飾する形容詞といった関係性はわからない)。
{: .notice--info }

FOSSのライセンス(MITライセンス)なので、ソースコードも公開されていますし、自由に
使えます。メンテナンス状況も良好で、バグレポートや、使用しているライブラリの管
理もきちんとされています。

VS Codeやvimといった、主要なエディタにも対応しています。`textlint`の公式サイ
トには明記されていませんが、
[秀丸のtextlintプラグイン](https://hide.maruo.co.jp/lib/macro/textlint001.html)
もあるようです。基本的にはCLIのコマンドですので、入力中に随時チェックさせるので
はないのなら、ほとんどのエディタで使えるでしょう。

JavaScriptでルールを作成できますから、開発環境の構築も比較的容易です。必要なの
は、JavaScriptの実行環境とパッケージを管理するコマンドで、Node.jsと`npm`があれ
ば十分です。辞書をユーザ定義の外部ファイルから参照できるので、ルールを作成した
あとでユーザが独自に辞書を維持拡張できますから、メンテナンスコストも低く抑えら
れます。

使用するルールの定義や、ルールごとの設定は`.textlintrc`というファイルで管理しま
す。このファイルはプロジェクトごとに変えられるので、プロジェクトに応じてルール
を定義できます。

`textlint`はモノリンガルですので、原文を考慮した校正はできません。原文が必要に
なるチェックは、CATのプラグインなどを利用するしかありません。

そもそも原文に文脈がない場合(`.po`ファイルなど)や、文脈が利用できない場合(CATと
連携する場合など)は、こうした文脈によるルールは効果を発揮できません。

対応している入力文書のファイル形式は、シンプルなテキスト形式(`.md`や`.rst`など)
しかありません。Wordの`.docx`やExcelの`.xlsx`形式の場合は、こうしたファイルに対
応するCATと連携させる、もしくはファイルから文章を抜き出す必要があります。

## `textlint`の基本的な使い方


`textlint`による校正に最低限必要な構成は以下のとおりです。

* JavaScriptの実行環境(Node.jsなど)
* JavaScriptのパッケージマネージャー(`npm`など)
* 原文のテキスト


Node.jsと`npm`のインストールは、公式の文書(
[Downloading and installing Node.js and npm](https://docs.npmjs.com/downloading-and-installing-node-js-and-npm)
)を参照します。どちらのインストールも完了しているものと想定して解説します。

まず、作業用のディレクトリを作成し、カレントディレクトリをそのディレクトリに変更します。

```console
mkdir textlint-demo
```

```console
cd textlint-demo
```

以降の作業は、このディレクトリを起点に作業します。

### `textlint`とルールのインストール

`textlint`とルールを`npm`でインストールします。ここでは
[textlint-rule-preset-JTF-style](https://github.com/textlint-ja/textlint-rule-preset-jtf-style)
というルール(正確には、ルールの集合体であるpresetルール）を使います。

```console
npm install textlint textlint-rule-preset-jtf-style
```

このコマンドを実行するとパッケージのインストールが始まり、各パッケージは、作業
ディレクトリの直下にできる`node_modules`というディレクトリにインストールされま
す。

作業ディレクトリには、`package.json`というファイルが作成されます。これは、
`npm`でインストールしたパッケージを管理するファイルです。このファイルには、プロ
ジェクトで使うJavaScriptのパッケージを指定します。また、`package-lock.json`とい
うファイルも作成されます。一見`package.json`と似ていますが、パッケージの依存関
係を正確に指定するファイルで、通常は手動で変更したりはしません。

### `textlint`の動作確認

`node_modules`には、`.bin`というディレクトリが作成され、そこに`textlint`の実行
ファイルがあります。次のコマンドを実行すると、ヘルプが表示されます。動作確認を
してみましょう。

```console
node_modules/.bin/textlint --help
```

ですが、いちいち相対パス名を入力するのは手間なので、`npx`を使います。


```console
npx textlint --help
```

`npx`を使うと、`node_modules/.bin`を自動的に補完してくれます。

### `textlint --init`でルールを設定

`textlint`の動作確認ができたので、ルールを設定します。ルールの設定には
`.textlintrc.json`というファイルを使います。`textlint --init`を実行すると、イン
ストール済みのルールを自動的に設定してくれます。


```console
npx textlint --init
```

ルールをたくさん使う場合は、`npm install`を実行して、まずルールをインストールしてしまうと、`.textlintrc.json`を一から書く必要がなくなるので、おすすめです。
{: .notice--info }

`textlintrc.json`の内容は以下のようになっているはずですので、確認してください。

```json
{
  "plugins": {},
  "filters": {},
  "rules": {
    "preset-jtf-style": true
}
```

**注意** ルールの命名規則として、ルールのパッケージ名は`textlint-rule-`で始まり
ます。しかし、`.textlintrc.json`でルールを指定する場合、`textlint-rule-`は省略
しないといけないので、注意しましょう。
{: .notice--warning }

`package.json`と`.textlintrc.json`は、別のプロジェクトで使い回すこともできます。
{: .notice--info }

### チェックする文書の作成

設定ファイルができたので、テストする文書を作成します。ファイル名はなんでもいいで
すが、`foo.md`にしましょう。`.md`という拡張子は、Markdownという記法のファイルで
す。

Markdownについては、[Markdown Guide](https://www.markdownguide.org/)の
[Getting Started](https://www.markdownguide.org/getting-started/)が参考になります。
[Markdown Cheat Sheet](https://www.markdownguide.org/cheat-sheet/)もあります。
{: .notice--info }

テキストエディタで`foo.md`を作成します。内容は以下のとおりです。

```markdown
この文章はエラーになります.

この文章はエラーになりません。
```

1行目は半角のピリオドで文章が終わっています。これがエラーになるはずです。

### `textlint`で校正

ファイルを保存したら、`textlint`でチェックしましょう。次のコマンドは、指定した
ルールで`foo.md`をチェックします。

```console
> npx textlint foo.md
/path/to/foo.md
 1:14  ✓ error  句読点には全角の「、」と「。」を使います。和文の句読点としてピリオド(.)とカンマ(,)を使用しません。  jtf-style/1.2.1.句点(。)と読点(、)
 1:14  ✓ error  和文の句読点としてはピリオドを使用しません。                                                        jtf-style/4.1.3.ピリオド(.)、カンマ(,)

✖ 2 problems (2 errors, 0 warnings, 0 infos)
✓ 2 fixable problems.
Try to run: $ textlint --fix [file]
```
{: .no-copy}

ルールには、自動修正に対応しているものもあります。`--fix`オプションを使うと、文書を自動的に修正します。

**注意** バックアップなどは作成されず、ファイルを直接修正します。
{: .notice--warning }


```console
npx textlint --fix foo.md
```

修正されたファイルを確認しましょう。以下のようになっているはずです。

```markdown
この文章はエラーになります。

この文章はエラーになりません。
```
### 特定の単語を無視

Markdownには、特殊な記法で拡張されているものもあります。例えば、
[GitHub Flavored Markdown](https://github.github.com/gfm/)や
[Kramdown](https://kramdown.gettalong.org/)
などがあります。こうした実装には拡張された記法があります。


```markdown
**注意** Kramdownでは、CSSのクラスを指定できます。
{: .notice--warning }
```

こうした拡張記法は、通常のMarkdownパーサーでは解釈できないので、本文として扱わ
れてエラーになることがあります。こうしたエラーを無視するには、
[`textlint-filter-rule-allowlist`](https://github.com/textlint/textlint-filter-rule-allowlist)
というフィルターを使います。まず、`npm`でフィルターをインストールします。

```console
npm install textlint-filter-rule-allowlist
```

次に、インストールしたフィルターを`.textlintrc.json`に追加します。これはフィル
ターなので、`filters`のキーで登録します。

```json
{
  "plugins": {},
  "filters": {
    "allowlist": {
      "allow": [
        "/{: [^}]+}/"
      ]
    }
  },
  "rules": {
    "preset-jtf-style": true
}
```

**注意** ルールと同様に、`textlint-filter-rule-`が`.textlintrc.json`では省略されるので、
注意してください。
{: .notice--warning }


`allow`というキーに、配列として無視する文字列、もしくは正規表現を指定します。こ
れで、マッチした文字列はエラーにならなくなります。

## 独自の校正ルール

プロジェクトごと、チームごと、もしくは自分だけでのルールを作成できます。ただし、
ハードルはやや高いです。

公式サイトの
[Creating Rules](https://textlint.org/docs/rule)
というページでルールの作成方法が説明されています。ただし、ある程度の前提知識を
持っていても、この文書だけですと隅々までの理解はなかなか難しいと思います。ソー
スコードにあたったり、他のルールのソースコードを参照するといいでしょう。

ルールには、ユーザが独自にルールの辞書を拡張できる場合と、ルールがプログラムに
ハードコードされている場合があります。ハードコードされている場合は、プログラム
に手を入れなければなりません。そのルールプロジェクトをforkして、独自にメンテナ
ンスするのはコストが高いので、あまり望ましくありません。

単純なルールであれば、
[`textlint-rule-prh`](https://github.com/textlint-rule/textlint-rule-prh)
がおすすめです。正規表現ベースのルールですが、辞書はYAMLで管理できて、辞書の定
義も簡単です。誤字や脱字、固有名詞の大文字小文字の間違いなどを防止できます。あ
らかじめ文書の固有名詞を抜き出し、そのまま辞書に登録すると、`MacOS`(正しくは、
macOS)や`Javascript`と(正しくは、JavaScript)といった見つけにくい表記間違いをチ
ェックできます。次の例は、`node.js`、`NODE.js`という間違いを検出します。

```yaml
---
version: 1
rules:
  - expected: Node.js
```

## 校正ルールを使い回す

別のプロジェクトで校正ルールを使い回すには、`package.json`と
`.textlintrc.json`を、そのプロジェクトのプロジェクトディレクトリにコピーします。
そして、そのプロジェクトディレクトリで、`npm install`を実行します。

```console
cp package.json .textlintrc.json /path/to/another/project/
cd /path/to/another/project
npm install
```

## `textlint`の応用

Markdownは、テキストのファイル形式としてデファクトスタンダードになった感があり
ます。Markdownを使っているのであれば、応用できる分野は多いでしょう。

チェック結果の出力フォーマットには様々なオプションが用意されているので、他のア
プリケーションとの連携が容易です。例えば、GitHub Actionsと連携させて、CMSの記事
やマニュアルの原稿を自動でチェックといった自動化は、すぐに実装できます。MCPサー
バも実装されているので、流行のLLMとの連携もできます。

CATとの連携は、マクロによるバッチ処理であれば、外部コマンドの実行をサポートする
仕組みがあればいいだけなのですが、CATに統合しつつ、CATのチェック機能に組み込む
用途だと、`textlint`を組み込んだAPIサーバを書く必要があるでしょう(コマンドの起
動コストが高いため)。

## OmegaTと連携する`rokujo-textlint-server`のご紹介

FOSSのCAT、
[OmegaT](https://omegat.org/)
に`textlint`をネイティブのチェック機能に追加する
[rokujo-textlint-server](https://github.com/trombik/rokujo-textlint-server)
というものを作ってみました。仕組み自体は簡単で、単純なHTTPによるAPIサーバです。
付属の`groovy`スクリプトを使って、チェック項目としてOmegaTに統合しています。

OmegaTスクリプトですが、ネイティブのissue providerを登録するので、ショートカッ
トキーですぐに起動できますし、結果もOmegaTのUIに統合されて見やすくなっています。

使い方は付属のREAMEをご覧ください。
