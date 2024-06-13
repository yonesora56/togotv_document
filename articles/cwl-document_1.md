---
title: "CWLの作成環境をVScodeの｢Dev Containers｣の機能を使って構築する (1)"
emoji: "📑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["CWL", "bioinformatics"]
published: true
---
__今回の記事で使用したCWLのファイルをおいているリポジトリは以下からアクセスすることができます｡__
https://github.com/yonesora56/togotv_cwl_for_remote_container

:::message
この記事は第13回国内版バイオハッカソン22.9､および第14回国内版バイオハッカソン23.9にてアイデアをいただいて作成しています｡ 
CWLに関してアドバイスをしてくださった[石井さん](https://zenn.dev/manabuishii)､[丹生さん](https://zenn.dev/tom_tan)､山本さんにこの場をお借りして御礼申し上げます｡
:::

:::message
__本記事の対象となる方__
■ ｢ワークフロー言語｣を知っているけど触ったことがない方
■ 動かしたいけど環境構築が難しそう...と考えている方
:::

# はじめに

皆さんは｢ワークフロー言語｣をご存知でしょうか? これらの言語は、一連の手順や操作を明示的に定義し、それらを連携させることで、より複雑な作業を効率的に行うことができます。
バイオインフォマティクス分野において､ワークフロー言語は重要な役割を担っています｡ 

## なぜワークフロー言語を使用するのか

バイオインフォマティクスにおけるデータ解析では、一つのツールのみで解析が終了することは極めて稀です｡ 通常､複数のツールを組み合わせて、大量のデータに対して一連のプロセスを繰り返し実行する必要があります。
これらの作業手順は、__ワークフロー(またはパイプライン)__ と呼ばれます。
しかし、手動でこれらの手順を繰り返すと、ヒューマンエラーに加え､異なる実行環境による再現性の問題が発生することがあります。このような場合に、__ワークフロー言語__ を用いることで、各ステップを自動化し、かつ実行環境に依存せず解析の再現性を向上させる事ができます｡

@[card](https://www.youtube.com/watch?v=yInRH3YK3Ik&list=PL0uaKHgcG00aJSa233gkhBA2HHe0-Ha-B)

@[card](https://doi.org/10.1038/d41586-019-02619-z)

&nbsp;

現在､さまざまな種類のワークフロー言語(Snakemake, Nextflow, Workflow Description language(WDL)...)が存在していますが、このドキュメントでは **Common Workflow Language (CWL)** について環境構築とその実例をご紹介します。

https://www.commonwl.org/

https://www.youtube.com/watch?v=86eY8xs-Vo8

-----

# なぜCWLを使うのか?
では次に､数多くあるワークフロー言語の中でも､なぜCWLを使うのか?ということについてここで簡単に説明します｡

## 1\. 開発ツールが充実している
CWLでは､[Rabix benten](https://github.com/rabix/benten)や､作成したワークフローをウェブブラウザで可視化することが可能な[CWLviewer](https://github.com/common-workflow-language/cwlviewer?tab=readme-ov-file)など､CWLのユーザーをサポートしてくれるツールが豊富に開発されています｡

https://github.com/rabix/benten

https://view.commonwl.org/

使用できるツール一覧は以下のページから確認できます｡
https://www.commonwl.org/tools/

## 2\.自分の実行したい環境に合わせて最適な実行エンジンが選択できる

CWLでは､複数の実行エンジンで実行することができます｡
例えば､[cwltool](https://github.com/common-workflow-language/cwltool)に加え､ジョブスケジューラに対応している[Toil](https://github.com/DataBiosphere/toil)などが存在します｡
自分の実行したい環境に合わせて選択肢が多いことが特徴です｡

一覧は以下のページから確認できます｡
https://www.commonwl.org/implementations/#what-can-execute-cwl-descriptions

## 3\.様々なワークフローがCWLで記述されている

実際にCWLを使って記述された解析ワークフローは数多くあります｡ 
CWLの公式サイトにはワークフローリポジトリが紹介されていたり、User Galleryでは大規模な活用事例も見ることができます。

### リポジトリ一覧
https://www.commonwl.org/repos/#repositories-of-cwl-tools-and-workflows

### User Gallery
https://www.commonwl.org/gallery/

以下に具体的なワークフローの例をいくつかご紹介します｡
この記事でご紹介するのは __日本の研究者の方々によって__ 作成されたワークフローです｡

### 例1: ヒトゲノムバリアント検出ワークフロー ddbj/human-reseq

このワークフローは､DDBJ(DNA Data Bank of Japan)で開発されたヒトゲノムバリアント検出ワークフローです｡

https://github.com/ddbj/human-reseq

fastqファイルを入力として､BAMファイルに変換する `fastqPE2bam.multisamples.cwl`などが含まれています｡

:::details CWL viewerで見てみよう
先程紹介したCWLviewerを使って､このワークフローを可視化してみましょう｡
CWLviewerにアクセスし､githubのリンクを入力することで可視化ができます｡

https://view.commonwl.org/workflows/github.com/ddbj/human-reseq/blob/master/Workflows/fastqPE2bam.multisamples.cwl

このように簡単に可視化ができます!
![CWLviewer 1](https://storage.googleapis.com/zenn-user-upload/0a287d465d8e-20240612.png)
:::

### 例2: DRY解析教本に掲載されているワークフローをCWLで記述 

次世代シーケンサー(NGS)によって生成されるデータの解析手法を解説している ｢次世代シーケンサーDRY解析教本｣という本があります｡
ここには､トランスクリプトームアセンブリなど､初学者に向けて様々な解析手法が丁寧に解説されています｡

https://gakken-mesh.jp/book/detail/9784780909838.html

この本に掲載されている一連の解析手法についてもCWLで記述されています｡

https://github.com/pitagora-network/DAT2-cwl/tree/main

ここでは紹介しきれないほどのワークフローがCWLで記述されています! 
皆さんもぜひ興味があるワークフローを [Repositories](https://www.commonwl.org/repos/#repositories-of-cwl-tools-and-workflows)から探してみてください!

## 4. ドキュメントが非常に充実している

CWLは様々なドキュメントが充実しています。例えば､公式ドキュメントからユーザーガイドを閲覧することができます｡
また､このユーザーガイドは日本語に翻訳されており､非常に参考にしやすくなっています｡

### Common Workflow Language User Guide

https://www.commonwl.org/user_guide/#common-workflow-language-user-guide

### Common Workflow Language User Guide (日本語版)

https://www.commonwl.org/user_guide/ja/

また､下記に示すような他にも日本の研究者による解説も多くあります｡ぜひ参考にしてみてください｡

https://github.com/pitagora-network/pitagora-cwl/wiki/CWL-Start-Guide-JP

https://oumpy.github.io/blog/2018/12/cwl.html

&nbsp;

# さあ､CWLを始めよう! でも環境構築が難しい...

以上を踏まえて、CWLを記述することのメリットがわかったと思います。
また、上記で述べたようにCWLには充実したドキュメントが用意されています。
特に日本語のドキュメントも豊富にあるため、日本語で学ぶことができるのは大きな利点です。
CWLについてより深く理解したい方は、ぜひこれらのドキュメントを参考にしてみてください。

しかしながら、「よし、CWLを書くぞ！」と意気込んでも、__環境構築という壁が立ちふさがると思います。__
CWLを始めるにあたって、開発環境をセットアップすることは重要なステップですが、時に困難を伴うこともあるはずです(自分も環境構築で挫折しそうになることがよくあります)。

そこでこのドキュメントでは、作業するマシンの環境に依存せず、CWLの開発環境を作成し、実際にCWLファイルを実行する方法を紹介します。
次の「環境構築編」では、Visual Studio CodeとDockerを使った開発環境の構築方法を説明します。
CWLを始めたいと思っているそこのあなた! ぜひ一緒に環境を整えて素晴らしいCWL lifeを過ごしましょう。

&nbsp;

# 環境構築編

## 【STEP1-1】VScodeのインストール

はじめに、コードエディターであるVisual Studio Code (VScode)のインストール方法を説明します。
特徴としては拡張機能が豊富に存在している点が挙げられ､この記事でも拡張機能を活用しながら環境構築を進めていきます｡

ダウンロードは以下のページにアクセスしておこないます｡ 皆さんも自分のコンピュータの環境に合わせて選んでください｡

https://code.visualstudio.com/download

:::message
この記事では､macOSの｢Apple silicon｣をクリックしてインストールして執筆しています｡
:::

## 【STEP1-2】拡張機能を導入する

先程述べたように､VSCodeには豊富な拡張機能が存在します｡ 
サイドバー(ここでは左)の拡張機能のボタン(四角が4つあつまっている部分)を押すと､様々な拡張機能がMarketplaceから検索できます｡
ここで､｢[Dev Containers](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers)｣ と検索してください｡ 
この拡張機能が必要なのでインストールします(以前は｢Remote Container extension｣という名前だったようですが､どうやら変わったようです)｡

![Dev-container](https://storage.googleapis.com/zenn-user-upload/e5d7dc91c2e3-20240527.png)

https://code.visualstudio.com/docs/devcontainers/containers

&nbsp;

インストール後､VScodeの画面をよくみると....

![VScode-1](https://storage.googleapis.com/zenn-user-upload/ec4060167e94-20240527.png)

![VScode-2](https://storage.googleapis.com/zenn-user-upload/680a6dfcc6ec-20240527.png)
*左下に｢><｣マークが出現*

左下に｢ >< ｣というマークが出てきます｡ このボタンを押すことで次の作業に移ることができます｡これでVScodeは一旦準備完了です｡

:::message
この｢Dev Containers｣だけでなく､他の拡張機能を含んだ拡張機能パック[Remote Development](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.vscode-remote-extensionpack)をインストールする方法もあるようです｡

![remote development](https://t907947.p.clickup-attachments.com/t907947/c73afc83-d397-4826-b62c-de24f1d30ea1/image.png)
:::

Dev containersに関する情報は以下の記事が参考になります｡ こちらも合わせてご覧ください｡

https://qiita.com/yoshii0110/items/c480e98cfe981e36dd56

https://gist.github.com/heronshoes/4e707bbc92ceee60d71fc09007e01d02#%E9%96%8B%E7%99%BA%E3%82%B3%E3%83%B3%E3%83%86%E3%83%8A%E3%81%A8%E3%81%AF%E4%BD%95%E3%81%8B

この拡張機能を使って､新しく環境構築などを行う(VScodeから自分にあった環境をつくる､など)ことも可能ですが､今回はGitHubにすでに用意されているテンプレートを使って簡単に実行できる方法をご紹介します(【STEP3】に記載しています)｡

&nbsp;

## 【STEP2】Docker Desktopのインストール

次にDocker Desktopをインストールします｡ ダウンロードは下のリンクにアクセスすることで可能です｡ 
今回はMac(Apple chip)版をインストールします｡

https://www.docker.com/products/docker-desktop

&nbsp;

インストール後､ダッシュボードを開くと以下のようになっています｡

![docker-dashboard](https://storage.googleapis.com/zenn-user-upload/39e82f2f5450-20240527.png)

:::message
### Docker Desktop 以外の手段
Docker desktopを使用する以外にも以下のような代替手段があります｡ 

1. lima + docker (macOS)
2. Rancher desktop ([https://rancherdesktop.io/](https://rancherdesktop.io/)) (macOS, moby) ※ただし今回使用しているようなRemote containerだと安定しないとのことです
3. orbstack ([https://orbstack.dev/](https://orbstack.dev/)) (macOS)
4. Docker on WSL (Windows)
5. Podman Desktop (https://podman-desktop.io/) 
:::

&nbsp;

## 【STEP3】GitHubからテンプレートを取得する

ここまででVScodeとdockerのインストールが完了しました｡ 次にCWLを実行する環境のテンプレートをGitHubから取得します｡

まず､以下に示しているリポジトリ([tom-tan/cwl-for-remote-container-template](https://github.com/tom-tan/cwl-for-remote-container-template))にアクセスしてください｡
今回はこのリポジトリをテンプレートにして環境を作っていきます｡ 
このテンプレートでは､既にシンタックスハイライトの機能があるCWL(Rabix/benten)などの拡張機能が使用できるように準備されており､更には実行エンジンであるcwltoolなども含まれているので自ら`pip install`する必要はありません｡

https://github.com/tom-tan/cwl-for-remote-container-template

ページ内の ｢Use this template｣(緑のボタン)をクリックし､｢Create a new repository｣を選択すると､自分のアカウントで新規リポジトリを作成することができます｡ 

:::message
__なお､GitHubのアカウントを持っていない場合はこのステップを飛ばしてください｡__
:::

![template](https://storage.googleapis.com/zenn-user-upload/7a7a0a256bad-20240527.png)

次に `git clone` を行います(GitHubのアカウントがない場合は､`tom-tan/cwl-for-remote-container-template`を､アカウントがある場合は､`your_account/cwl-for-remote-container-template` ということになります)｡

```bash
#アカウントが無い場合
git clone https://github.com/tom-tan/cwl-for-remote-container-template 

#アカウントがある場合(Use this template後の状態  your_accountは自分のアカウント名に置き換えてください)
git clone https://github.com/your_account/cwl-for-remote-container-template
```

:::message
__この記事ではリポジトリの名前を`togotv_cwl_for_remote_container`に変更しています｡__
:::

この作業が終了したら､つづいてVSCodeを開きます｡ VSCode画面左下の緑の｢ >< ｣マークを押すと､ 検索窓に以下のようなオプションが出てくるので､｢コンテナーでフォルダーを開く｣を選択し､先程`git clone`したローカルリポジトリを開きます｡

![VScode-3](https://storage.googleapis.com/zenn-user-upload/14917eb2b27b-20240527.png)

最初の立ち上げ時には､5分程度かかりました｡
ログを見ていると環境構築のために色々されていることがわかります｡
![VScode-4](https://storage.googleapis.com/zenn-user-upload/da6eb7c3671d-20240527.png)

ターミナルを開いてみると､以下のように
`/workspaces/togotv_cwl_for_remote_container(repository_name)`となっています｡

![VScode-5](https://storage.googleapis.com/zenn-user-upload/bddab18fa4b7-20240527.png)

&nbsp;

実際にCWL関連のツールは使えるようになっているのか見てみましょう｡
`cwl` と入力してtabを2回ほど押してみると...

![VScode-6](https://storage.googleapis.com/zenn-user-upload/899cc2808da0-20240527.png)

このように､cwltoolなど実行に必要なツールが導入されています｡
では､Dockerのコンテナが起動しているかどうかdocker desktopで確認してみると...

![docker_dashboard2](https://storage.googleapis.com/zenn-user-upload/5c5d14a8f9df-20240527.png)

上記のように､立ち上がっているのがわかります｡
これで一旦環境構築が完了しました!

&nbsp;

## 【番外編】GitHub Codespacesで実行環境を作って作業する

ここまでは､ローカルの自分のマシンで行うことを前提に色々準備してきました｡ 
しかしながら､__｢もっと楽に環境構築して動かしてみたい!!｣__ という方もいらっしゃるかと思います｡ 
そこで活用できるのが｢GitHub Codespaces｣というクラウドでホストされている開発環境です｡ その概要は以下の日本語ドキュメントをご覧ください｡

https://docs.github.com/ja/codespaces/overview#what-is-a-codespace

先程テンプレートを取得する段階で､｢Use this template｣を押す時に気になった方がいらっしゃるかもしれませんが､この時､｢Create a new repository｣ と｢Open in a codespace｣と2つの選択肢があったかと思います｡ 
このとき｢Open in a codespace｣を選べばブラウザでこの開発環境が立ち上げることができます｡

また､VScode経由でも開くことが可能です｡ 
そのためには [｢GitHub codespaces｣](https://marketplace.visualstudio.com/items?itemName=GitHub.codespaces) という拡張機能をインストールしてください｡
その後､左下の｢><｣ボタンを押してください｡

![VScode-7](https://storage.googleapis.com/zenn-user-upload/e94f99c773f2-20240527.png)

次に､｢Create New Codespace｣をクリックして､立ち上げたいリポジトリを選択します｡
Create New Codespaceをクリックすると以下のような表示が出てきます｡

![VScode-8](https://storage.googleapis.com/zenn-user-upload/58e04cac7aa6-20240527.png)

ここから`Use this template`で取得したリポジトリを選択して...
![VScode-9](https://storage.googleapis.com/zenn-user-upload/968a26a0596e-20240527.png)

そうすると､自動的に環境が構築されていきます｡
今回はすでに環境を構築しているものを使用します｡なお､Codespaceで作成した環境は､自分のGitHubのページ(Your Codespaces)から確認できます｡最初の環境の立ち上げには同様に5分程度時間がかかります｡

:::message alert
1ヶ月で使用できる時間には限りがあるようです｡
自分の場合では､Codespacesの使用時間が(1ヶ月で割り当てられている)全体の90%を超えると､警告のメールが届きました｡
:::

:::message
このCodespacesを使った環境構築では､実は`Use this template`をしなくても立ち上げることができます｡
まず､[cwl-for-remote-container-template](https://github.com/tom-tan/cwl-for-remote-container-template)にアクセスします｡

`Use this template`ボタンの横に`V`があると思いますが､これを押すと､Open in a codespaceという選択肢が出てきます｡
![VScode-10](https://storage.googleapis.com/zenn-user-upload/3a5650a912bb-20240527.png)

こちらを選択すると､自動的にCodespacesが立ち上がります｡
![VScode-11](https://storage.googleapis.com/zenn-user-upload/9ae5837c0737-20240527.png)

より簡単に環境構築を行うことができますが､Codespacesの使用時間には注意が必要です｡
:::

&nbsp;

---

# zatsu-cwl-generatorを使ってCWLファイルを作成する

ここまではCWLに関する説明､および環境導入を行いました｡この項目では実際にCWLファイルの記述､実行を行っていきます｡
CWLファイルは記述する内容をYAMLかJSONの形式で記述し、｢.cwl｣という拡張子でファイルに保存します。そして実行時にこのCWLファイルを実行エンジンに入力すると、ワークフローが実行される､という流れになっています｡

```bash
cwltool hoge.cwl # 例
```
しかしながら､いきなり書きはじめるというのはとても難しいと思います｡
そこでCWLファイルを簡単に生成できるツール､[zatsu-cwl-generator](https://github.com/tom-tan/zatsu-cwl-generator)を使ってCWLファイルを出力してみましょう!
先程作成した環境にはすでにインストールされているため､インストールする必要はありません｡

https://github.com/tom-tan/zatsu-cwl-generator

https://qiita.com/tm_tn/items/2c789c5b3c28e3eb3c9a

&nbsp;

## `grep`コマンドのCWLファイルを作成する

それでは､実際に生成してみましょう｡
この記事では､grepコマンドによる処理をCWLによって記述する例を紹介します｡

実行するのは､`grep one mock.txt > grep_out.txt`というプロセスです｡ 
以下のようなmock.txtを作成し､このファイルに対して､`one`という文字列をgrepで検索し､その結果をgrep_out.txtに出力します｡

https://github.com/yonesora56/togotv_cwl_for_remote_container/blob/master/zatsu_cwl/mock.txt

```bash
grep one mock.txt > grep_out.txt
```

&nbsp;

まず､ターミナルを開きます｡
今回は`zatsu_cwl`というディレクトリを作成し､その中で作業を行います｡

```bash
mkdir zatsu_cwl
cd zatsu_cwl
```

次に､ターミナルで`zatsu-cwl-generator`と入力します｡
次に､cwlファイルとして書きたい処理を '' で囲んで記入します｡ 
今回は`grep`コマンドの処理を例に実行してみます｡

```bash
zatsu-cwl-generator 'grep one ./data/mock.txt > grepout.txt'
```

実行すると､標準出力に以下のようにcwlファイルが出力されます｡

```yaml:generated by zatsu-cwl-generator
#!/usr/bin/env cwl-runner
# Generated from: grep one ./data/mock.txt > grepout.txt
class: CommandLineTool
cwlVersion: v1.0
baseCommand: grep
arguments:
  - $(inputs.one)
  - $(inputs.mock_txt)
inputs:
  - id: one
    type: Any
    default: one
  - id: mock_txt
    type: File
    default:
      class: File
      location: ./data/mock.txt
outputs:
  - id: all-for-debugging
    type:
      type: array
      items: [File, Directory]
    outputBinding:
      glob: "*"
  - id: out
    type: stdout
stdout: grepout.txt

```

この出力される形式はYAML形式になっています｡
shebang(`#!/usr/bin/env cwl-runner`)以下には､今回CWLを実行するためのフィールドとして､上から順に
`class` ､ `cwlversion`､`baseCommand`､ `arguments`､ `inputs` ､ `outputs`､ `stdout` が出力されています｡

この出力をリダイレクトしてファイルとして保存します｡
今回は､`zatsu-cwl`ディレクトリに保存します｡

```bash:
zatsu-cwl-generator 'grep one mock.txt > grepout.txt' > grep_zatsu.cwl
```

:::message
### cwlversionはどれを使う?
出力されるフィールドの中に､`cwlVersion` というのがあります｡
これはCWLを動かすエンジンであるcwltoolのバージョンを記述する場所になります｡
現在､v1.0, v1.1, v1.2など複数のバージョンに対応しています｡
なお､この記事ではv1.0で書いていますが､基本的な書き方はどのバージョンでも大きく変わりません｡
![cwl_version_check](https://storage.googleapis.com/zenn-user-upload/76a8a6e82763-20240527.png)
:::

&nbsp;

## 記述が正しいかチェックする

zatsu-cwl-genratorで出力されたファイルに対し､実際の実行前に記述が本当に正しいか確認することができます｡
`cwltool –-validate` コマンドを実行すると､記述したCWLファイルを評価することができます｡ 
以下のようにoptionを追加します｡

```bash:
cwltool --validate grep_zatsu.cwl
```

すると以下のように出力されました｡
```bash:
INFO /usr/local/bin/cwltool 3.1.20240508115724
INFO Resolved 'grep_zatsu.cwl' to 'file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl/grep_zatsu.cwl'
grep_zatsu.cwl is valid CWL.
```
今回の場合はエラーは確認されず､記述としては正しいようです｡ 
このように､記述がおかしい場合はコマンドで確認できる他､スクリプトを編集している際に､補助的に赤線で明示されますので参考にしてください｡

&nbsp;

## 実際に実行する

ファイルの記載が正しいことを確認できたので､次に実際に`cwltool`というコマンドで試してみます(以降の操作はzatsu_generatorディレクトリでの作業です)｡ 

```bash:
cwltool grep_zatsu.cwl 
```

実行してみると､以下のように解析が行われます｡
```bash:
INFO /usr/local/bin/cwltool 3.1.20240508115724
INFO Resolved 'grep_zatsu.cwl' to 'file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl/grep_zatsu.cwl'
INFO [job grep_zatsu.cwl] /tmp/i5d745lc$ grep \
    one \
    /tmp/vd9_s1sr/stg873258e0-d845-4666-98b9-2c121d5f8bbe/mock.txt > /tmp/i5d745lc/grepout.txt
INFO [job grep_zatsu.cwl] completed success
{
    "all-for-debugging": [
        {
            "location": "file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl/grepout.txt",
            "basename": "grepout.txt",
            "class": "File",
            "checksum": "sha1$a972f6d93fec7529fd4af8344ca298eea43dfbc5",
            "size": 16,
            "path": "/workspaces/togotv_cwl_for_remote_container/zatsu_cwl/grepout.txt"
        }
    ],
    "out": {
        "location": "file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl/grepout.txt",
        "basename": "grepout.txt",
        "class": "File",
        "checksum": "sha1$a972f6d93fec7529fd4af8344ca298eea43dfbc5",
        "size": 16,
        "path": "/workspaces/togotv_cwl_for_remote_container/zatsu_cwl/grepout.txt"
    }
}INFO Final process status is success
```
無事ワークフローが成功し､`zatsu_cwl`ディレクトリ内に `grepout.txt` が出力されました｡

https://github.com/yonesora56/togotv_cwl_for_remote_container/blob/master/zatsu_cwl/grepout.txt

このようにzatsu-cwl-generatorを使うことで､簡単にCWLファイルを作成することができます｡

&nbsp;

:::message
### エラーが発生した時
もしこの実行でエラーが発生した場合､ `--debug` オプションを追加してより詳しいエラーの結果を取得することができます｡
はじめての処理を実行する際には､エラーの修正が容易にするためこのオプションをつけて実行することをおすすめします｡

```bash:
cwltool --debug grep_zatsu.cwl 
```
:::

&nbsp;

## 【発展】 自分で修正してみよう
上記のように実際に実行することができることを確認しました｡
しかし､このファイルを修正することでより良い解析を実行することができます｡
実は､先程作成したgrep処理のファイル(`grep_zatsu.cwl`)の編集時には､以下の部分に赤線が示されていました｡

https://github.com/yonesora56/togotv_cwl_for_remote_container/blob/master/zatsu_cwl/grep_zatsu.cwl#L9-L12

エラーメッセージを見ると､ `Expecting one of: ['Directory', 'File', 'boolean', 'double', 'float', 'int', 'long', 'null', 'stderr', 'stdout', 'string']`という表示が出ていました｡
ここでは`one`という文字列(string)を入力するので､以下のように修正できます｡

https://github.com/yonesora56/togotv_cwl_for_remote_container/blob/master/zatsu_cwl/grep_zatsu_v2.cwl#L9-L12

このように､zatsu-cwl-generatorで生成されたファイルを修正しながら､CWLの文法を勉強していくということが可能です｡

&nbsp;

## 【発展】 `--help`オプションを活用しよう

`cwltool grep_zatsu.cwl --help`のように､CWLファイルの次に`-h`, `--help`オプションをつけると､その __CWLファイル自体のヘルプを見ることができます__｡ (`cwltool --help`ではありません!)
どういうことか実際にやってみましょう｡
先程のセクションで修正した`grep_zatsu_v2.cwl`で実行してみました｡

```bash
cwltool grep_zatsu_v2.cwl --help
INFO /usr/local/bin/cwltool 3.1.20240508115724
INFO Resolved 'grep_zatsu_v2.cwl' to 'file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl/grep_zatsu_v2.cwl'
usage: grep_zatsu_v2.cwl [-h] [--one ONE] [--mock_txt MOCK_TXT] [job_order]

positional arguments:
  job_order            Job input json file

options:
  -h, --help           show this help message and exit
  --one ONE
  --mock_txt MOCK_TXT
```
このように､指定したパラメータ(今回は`--one`と`--mock_txt`)に関する情報が取得できます｡
ちなみに`cwltool --help grep_zatsu_v2.cwl`とやると`cwltool`に関するhelpが出力されるので､順序には注意が必要です｡

:::details cwltool --helpの場合
```bash:
cwltool --help grep_zatsu_v2.cwl
usage: cwltool [-h] [--basedir BASEDIR] [--outdir OUTDIR] [--log-dir LOG_DIR] [--parallel] [--preserve-environment ENVVAR | --preserve-entire-environment] [--rm-container | --leave-container] [--cidfile-dir CIDFILE_DIR] [--cidfile-prefix CIDFILE_PREFIX]
               [--tmpdir-prefix TMPDIR_PREFIX] [--tmp-outdir-prefix TMP_OUTDIR_PREFIX | --cachedir CACHEDIR] [--rm-tmpdir | --leave-tmpdir] [--move-outputs | --leave-outputs | --copy-outputs] [--enable-pull | --disable-pull]
               [--rdf-serializer RDF_SERIALIZER] [--eval-timeout EVAL_TIMEOUT] [--provenance PROVENANCE] [--enable-user-provenance] [--disable-user-provenance] [--enable-host-provenance] [--disable-host-provenance] [--orcid ORCID]
               [--full-name CWL_FULL_NAME] [--print-rdf | --print-dot | --print-pre | --print-deps | --print-input-deps | --pack | --version | --validate | --print-supported-versions | --print-subgraph | --print-targets | --make-template]
               [--strict | --non-strict] [--skip-schemas] [--no-doc-cache | --doc-cache] [--verbose | --no-warnings | --quiet | --debug] [--write-summary WRITE_SUMMARY] [--strict-memory-limit] [--strict-cpu-limit] [--timestamps] [--js-console]
               [--disable-js-validation] [--js-hint-options-file JS_HINT_OPTIONS_FILE] [--user-space-docker-cmd CMD | --udocker | --singularity | --podman | --no-container] [--tool-help] [--relative-deps {primary,cwd}] [--enable-dev] [--enable-ext]
               [--enable-color | --disable-color] [--default-container DEFAULT_CONTAINER] [--no-match-user] [--custom-net CUSTOM_NET] [--enable-ga4gh-tool-registry | --disable-ga4gh-tool-registry] [--add-ga4gh-tool-registry GA4GH_TOOL_REGISTRIES]
               [--on-error {stop,continue}] [--compute-checksum | --no-compute-checksum] [--relax-path-checks] [--force-docker-pull] [--no-read-only] [--overrides OVERRIDES] [--target TARGET | --single-step SINGLE_STEP | --single-process SINGLE_PROCESS]
               [--mpi-config-file MPI_CONFIG_FILE]
               [cwl_document] ...

Reference executor for Common Workflow Language standards. Not for production use.

positional arguments:
  cwl_document          path or URL to a CWL Workflow, CommandLineTool, or ExpressionTool. If the `inputs_object` has a `cwl:tool` field indicating the path or URL to the cwl_document, then the `cwl_document` argument is optional.
  inputs_object         path or URL to a YAML or JSON formatted description of the required input values for the given `cwl_document`.

options:
  -h, --help            show this help message and exit
  --basedir BASEDIR
  --outdir OUTDIR       Output directory. The default is the current directory.
  --log-dir LOG_DIR     Log your tools stdout/stderr to this location outside of container This will only log stdout/stderr if you specify stdout/stderr in their respective fields or capture it as an output
  --parallel            Run jobs in parallel.
  --preserve-environment ENVVAR
                        Preserve specific environment variable when running CommandLineTools. May be provided multiple times. By default PATH is preserved when not running in a container.
  --preserve-entire-environment
                        Preserve all environment variables when running CommandLineTools without a software container.
  --rm-container        Delete Docker container used by jobs after they exit (default)
  --leave-container     Do not delete Docker container used by jobs after they exit
  --tmpdir-prefix TMPDIR_PREFIX
                        Path prefix for temporary directories. If --tmpdir-prefix is not provided, then the prefix for temporary directories is influenced by the value of the TMPDIR, TEMP, or TMP environment variables. Taking those into consideration, the
                        current default is /tmp/.
  --tmp-outdir-prefix TMP_OUTDIR_PREFIX
                        Path prefix for intermediate output directories. Defaults to the value of --tmpdir-prefix.
  --cachedir CACHEDIR   Directory to cache intermediate workflow outputs to avoid recomputing steps. Can be very helpful in the development and troubleshooting of CWL documents.
  --rm-tmpdir           Delete intermediate temporary directories (default)
  --leave-tmpdir        Do not delete intermediate temporary directories
  --move-outputs        Move output files to the workflow output directory and delete intermediate output directories (default).
  --leave-outputs       Leave output files in intermediate output directories.
  --copy-outputs        Copy output files to the workflow output directory and don't delete intermediate output directories.
  --enable-pull         Try to pull Docker images
  --disable-pull        Do not try to pull Docker images
  --rdf-serializer RDF_SERIALIZER
                        Output RDF serialization format used by --print-rdf (one of turtle (default), n3, nt, xml)
  --eval-timeout EVAL_TIMEOUT
                        Time to wait for a Javascript expression to evaluate before giving an error, default 60s.
  --print-rdf           Print corresponding RDF graph for workflow and exit
  --print-dot           Print workflow visualization in graphviz format and exit
  --print-pre           Print CWL document after preprocessing.
  --print-deps          Print CWL document dependencies.
  --print-input-deps    Print input object document dependencies.
  --pack                Combine components into single document and print.
  --version             Print version and exit
  --validate            Validate CWL document only.
  --print-supported-versions
                        Print supported CWL specs.
  --print-subgraph      Print workflow subgraph that will execute. Can combined with --target or --single-step
  --print-targets       Print targets (output parameters)
  --make-template       Generate a template input object
  --strict              Strict validation (unrecognized or out of place fields are error)
  --non-strict          Lenient validation (ignore unrecognized fields)
  --skip-schemas        Skip loading of schemas
  --no-doc-cache        Disable disk cache for documents loaded over HTTP
  --doc-cache           Enable disk cache for documents loaded over HTTP
  --verbose             Default logging
  --no-warnings         Only print errors.
  --quiet               Only print warnings and errors.
  --debug               Print even more logging
  --write-summary WRITE_SUMMARY, -w WRITE_SUMMARY
                        Path to write the final output JSON object to. Default is stdout.
  --strict-memory-limit
                        When running with software containers and the Docker engine, pass either the calculated memory allocation from ResourceRequirements or the default of 1 gigabyte to Docker's --memory option.
  --strict-cpu-limit    When running with software containers and the Docker engine, pass either the calculated cpu allocation from ResourceRequirements or the default of 1 core to Docker's --cpu option. Requires docker version >= v1.13.
  --timestamps          Add timestamps to the errors, warnings, and notifications.
  --js-console          Enable javascript console output
  --disable-js-validation
                        Disable javascript validation.
  --js-hint-options-file JS_HINT_OPTIONS_FILE
                        File of options to pass to jshint. This includes the added option "includewarnings".
  --user-space-docker-cmd CMD
                        (Linux/OS X only) Specify the path to udocker. Implies --udocker
  --udocker             (Linux/OS X only) Use the udocker runtime for running containers (equivalent to --user-space-docker-cmd=udocker).
  --singularity         Use Singularity or Apptainer runtime for running containers. Requires Singularity v2.6.1+ and Linux with kernel version v3.18+ or with overlayfs support backported.
  --podman              Use Podman runtime for running containers.
  --no-container        Do not execute jobs in a Docker container, even when `DockerRequirement` is specified under `hints`.
  --tool-help           Print command line help for tool
  --relative-deps {primary,cwd}
                        When using --print-deps, print paths relative to primary file or current working directory.
  --enable-dev          Enable loading and running unofficial development versions of the CWL standards.
  --enable-ext          Enable loading and running 'cwltool:' extensions to the CWL standards.
  --enable-color        Enable logging color (default enabled)
  --disable-color       Disable colored logging (default false)
  --default-container DEFAULT_CONTAINER
                        Specify a default software container to use for any CommandLineTool without a DockerRequirement.
  --no-match-user       Disable passing the current uid to `docker run --user`
  --custom-net CUSTOM_NET
                        Passed to `docker run` as the '--net' parameter when NetworkAccess is true, which is its default setting.
  --enable-ga4gh-tool-registry
                        Enable tool resolution using GA4GH tool registry API
  --disable-ga4gh-tool-registry
                        Disable tool resolution using GA4GH tool registry API
  --add-ga4gh-tool-registry GA4GH_TOOL_REGISTRIES
                        Add a GA4GH tool registry endpoint to use for resolution, default ['https://dockstore.org/api']
  --on-error {stop,continue}
                        Desired workflow behavior when a step fails. One of 'stop' (do not submit any more steps) or 'continue' (may submit other steps that are not downstream from the error). Default is 'stop'.
  --compute-checksum    Compute checksum of contents while collecting outputs
  --no-compute-checksum
                        Do not compute checksum of contents while collecting outputs
  --relax-path-checks   Relax requirements on path names to permit spaces and hash characters.
  --force-docker-pull   Pull latest software container image even if it is locally present
  --no-read-only        Do not set root directory in the container as read-only
  --overrides OVERRIDES
                        Read process requirement overrides from file.
  --target TARGET, -t TARGET
                        Only execute steps that contribute to listed targets (can be provided more than once).
  --single-step SINGLE_STEP
                        Only executes a single step in a workflow. The input object must match that step's inputs. Can be combined with --print-subgraph.
  --single-process SINGLE_PROCESS
                        Only executes the underlying Process (CommandLineTool, ExpressionTool, or sub-Workflow) for the given step in a workflow. This will not include any step-level processing: 'scatter', 'when'; and there will be no processing of step-
                        level 'default', or 'valueFrom' input modifiers. However, requirements/hints from the step or parent workflow(s) will be inherited as usual.The input object must match that Process's inputs.
  --mpi-config-file MPI_CONFIG_FILE
                        Platform specific configuration for MPI (parallel launcher, its flag etc). See README section 'Running MPI-based tools' for details of the format.

Options for recording the Docker container identifier into a file.:
  --cidfile-dir CIDFILE_DIR
                        Store the Docker container ID into a file in the specified directory.
  --cidfile-prefix CIDFILE_PREFIX
                        Specify a prefix to the container ID filename. Final file name will be followed by a timestamp. The default is no prefix.

Options for recording provenance information of the execution:
  --provenance PROVENANCE
                        Save provenance to specified folder as a Research Object that captures and aggregates workflow execution and data products.
  --enable-user-provenance
                        Record user account info as part of provenance.
  --disable-user-provenance
                        Do not record user account info in provenance.
  --enable-host-provenance
                        Record host info as part of provenance.
  --disable-host-provenance
                        Do not record host info in provenance.
  --orcid ORCID         Record user ORCID identifier as part of provenance, e.g. https://orcid.org/0000-0002-1825-0097 or 0000-0002-1825-0097. Alternatively the environment variable ORCID may be set.
  --full-name CWL_FULL_NAME
                        Record full name of user as part of provenance, e.g. Josiah Carberry. You may need to use shell quotes to preserve spaces. Alternatively the environment variable CWL_FULL_NAME may be set.
```
:::
例として､grep_zatsu_v3.cwlに`doc`フィールドを以下のように書き加えてみました｡

https://github.com/yonesora56/togotv_cwl_for_remote_container/blob/master/zatsu_cwl/grep_zatsu_v3.cwl#L9-L19

再度実行してみると以下のようになります｡

```bash:
cwltool grep_zatsu_v3.cwl -h
INFO /usr/local/bin/cwltool 3.1.20240508115724
INFO Resolved 'grep_zatsu_v3.cwl' to 'file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl/grep_zatsu_v3.cwl'
usage: grep_zatsu_v3.cwl [-h] [--one ONE] [--mock_txt MOCK_TXT] [job_order]

positional arguments:
  job_order            Job input json file

options:
  -h, --help           show this help message and exit
  --one ONE            please input grep pattern Default: one
  --mock_txt MOCK_TXT  please input text file Default: ./mock.txt
```
このように`doc`フィールドを書いておくことでより再現性を高めることが可能です! 

&nbsp;

# 終わりに

この記事では､環境構築と簡単なCWLファイルを作成するところまでご紹介しました｡
しかしながら､CWLを更に楽しむために､引き続きこの環境でどんどんCWLファイルを作っていきます!
自分で修正しながら実際に __ワークフローを作っていく__ 例については次の [記事](https://zenn.dev/sorayone/articles/cwl-document_2)で紹介しています｡
また､3番目の [記事](https://zenn.dev/sorayone/articles/cwl-document_3)では､`zatsu-cwl-generator`をフルに使ってバイオインフォマティクスの解析のワークフローを記述していきます｡

https://zenn.dev/sorayone/articles/cwl-document_2

https://zenn.dev/sorayone/articles/cwl-document_3

&nbsp;

__今回の記事で使用したCWLのファイルをおいているリポジトリは以下からアクセスすることができます｡__
https://github.com/yonesora56/togotv_cwl_for_remote_container