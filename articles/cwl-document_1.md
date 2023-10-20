---
title: "CWLの作成環境をVScodeの｢Dev Containers｣の機能を使って構築する (1)"
emoji: "📑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["CWL", "bioinformatics"]
published: false
---
:::message
今回の記事で使用したCWLのファイルをおいているリポジトリは以下からアクセスすることができます｡
#### [togotv_cwl_for_remote_container](https://github.com/yonezawa-sora/togotv_cwl_for_remote_container)
:::

:::message alert
この赤いフィールドは後ほど修正する点をメモしています
:::

# はじめに

皆さんは｢ワークフロー言語｣をご存知でしょうか? これらの言語は、一連の手順や操作を明示的に定義し、それらを連携させることで、より複雑な作業を効率的に行うことができます。
バイオインフォマティクスにおいて､ワークフロー言語は重要な役割を担っています｡ 

&nbsp;

:::message
__本記事の対象となる方__
■ ｢ワークフロー言語｣を知っているけど触ったことがない方
■ 動かしたいけど環境構築が難しそう...と考えている方
:::

&nbsp;

# なぜワークフロー言語を使用するのか

バイオインフォマティクスにおけるデータ解析では、一つのツールのみで解析が終了することは極めて稀です｡ 通常､複数のツールを組み合わせて、大量のデータに対して一連のプロセスを繰り返し実行する必要があります。これらの作業手順は、__ワークフロー(またはパイプライン)__ と呼ばれます。

&nbsp;

しかし、手動でこれらの手順を繰り返すと、ヒューマンエラーに加え､異なる実行環境による再現性の問題が発生することがあります。このような場合に、ワークフロー言語を用いることで、各ステップを自動化し、かつ実行環境に依存せず解析の再現性を向上させる事ができます｡

https://www.youtube.com/watch?v=yInRH3YK3Ik&list=PL0uaKHgcG00aJSa233gkhBA2HHe0-Ha-B

&nbsp;

Workflow systems turn raw data into scientific knowledge
https://doi.org/10.1038/d41586-019-02619-z

[<img src="https://hackmd.io/_uploads/rJSw1mzg6.png" width="500" height="">](https://doi.org/10.1038/d41586-019-02619-z)

&nbsp;

現在､さまざまな種類のワークフロー言語(Snakemake, Nextflow, Workflow Description language(WDL)...)が存在していますが、このドキュメントと統合TVの動画では**Common Workflow Language (CWL)** について環境構築と実例をご紹介します。

&nbsp;

:::success
### なぜCWLを使うのか?

では次に､数多くあるワークフロー言語の中でも､なぜCWLなのか?ということについてここで説明します｡

#### 1\. bentenなどの開発ツールが充実している

CWLでは､[Rabix benten](https://github.com/rabix/benten)や､VScodeの拡張機能 [vscode-cwl](https://marketplace.visualstudio.com/items?itemName=manabuishii.vscode-cwl)など､CWLのユーザーをサポートしてくれるツールが豊富に開発されています｡

#### 参考：[CWL公式サイト Development Tools](https://www.commonwl.org/tools/)

#### 2\.自分の実行したい環境に合わせて最適なものが選択できる

CWLでは､複数のツールで実行することができます｡例えば､[cwltool](https://github.com/common-workflow-language/cwltool)に加え､ジョブスケジューラに対応している[Toil](https://github.com/DataBiosphere/toil)などが存在します｡自分の実行したい環境に合わせて選択肢が多いことが特徴です｡

#### 参考：[CWL公式サイト Implementations](https://www.commonwl.org/implementations/)
:::

実際にCWLを使って記述されたバイオインフォマティクスにおける解析ワークフローは数多くあります｡ 例えば､ヒトゲノムバリアント検出ワークフローである｢ddbj/human-reseq｣が挙げられます｡

#### 参考：[ddbj/human-reseq](https://github.com/ddbj/human-reseq)

[<img src="https://hackmd.io/_uploads/B1kuG7zla.png" width="500" height="200">](https://github.com/ddbj/human-reseq) 

このように､バイオインフォマティクスに関するワークフローは多くのツールで導入されています｡

CWLについてより詳しく知りたい方は、下記に示している日本語のドキュメントや書籍も多くあるので、ぜひ参考にしてください。

#### 参考：[CWL日本語公式ドキュメント](https://github.com/pitagora-network/pitagora-cwl/wiki/CWL-Start-Guide-JP)

[<img src="https://hackmd.io/_uploads/BJtNmQzx6.png" width="500" height="">](https://github.com/pitagora-network/pitagora-cwl/wiki/CWL-Start-Guide-JP) 

#### 参考：[Common Workflow Language入門](https://oumpy.github.io/blog/2018/12/cwl.html)

[<img src="https://hackmd.io/_uploads/HkUjQQzla.png" width="500" height="">](https://oumpy.github.io/blog/2018/12/cwl.html)

&nbsp; 

CWLの実行を行うエンジンであるcwltoolなどは、pipコマンドやcondaを使ってインストールすることが可能です。
しかしながら､環境構築は大変な場合が多々あるかと思います。そこでこのドキュメントでは､作業するコンピュータの環境に依存せず、CWLの開発環境を作成して実行する方法を紹介します。

&nbsp;

:::success
なお､このドキュメントと統合TVの動画は､第13回国内版バイオハッカソン22.9､および第14回国内版バイオハッカソン23.9にてアイデアをいただいて作成しています｡ 主な内容はQiitaの以下の記事がベースになっています｡ CWLに関してアドバイスをしてくださった石井さん､丹生さん､山本さんにこの場をお借りして御礼申し上げます｡

[指先一つで立ち上げる CWL ツール・ワークフロー作成環境](https://qiita.com/tm_tn/items/3fafe22e2c4a92a7f597)
:::

&nbsp;

&nbsp;

# 準備しよう

## 【STEP1-1】VScodeのインストール

はじめに、Visual Studio Code (VScode)のインストール方法を説明します。VScodeoといえばもはや業界で標準的なコードエディタになりつつあります｡ 特徴としては拡張機能が豊富に存在している部分であり､この記事でもこの部分を活用します｡

  

ダウンロードは以下のページにアクセスしておこないます｡ 皆さんも自分のコンピュータの環境に合わせて選んでください｡

#### 参考： [Download Visual Studio Code](https://code.visualstudio.com/download)

<img src="https://hackmd.io/_uploads/H1FCNmGeT.png" width="500" height="">

この記事では､macOSの｢Apple silicon｣をクリックしてインストールして執筆しています｡

&nbsp;

## 【STEP1-2】拡張機能を導入する
環境構築の準備については､動画で見てもらったほうがわかりやすいかと思いますので､ここでは補足的な情報を主に記述しています｡

先程述べたように､VSCodeには豊富な拡張機能が存在します｡ サイドバー(ここでは左)の拡張機能のボタン(四角が4つあつまっている部分)を押すと､様々な拡張機能がMarketplaceで検索できます｡

ここで､｢[Dev Containers](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers)｣ と検索してください｡ こちらが必要なのでインストールします(以前は｢Remote -Container extension｣という名前だったようですが､どうやら変わったようです)｡

<img src="https://t907947.p.clickup-attachments.com/t907947/8dede111-052d-4ea4-ba7c-007ed30335ed/image.png" width="600" height="">

&nbsp;

#### 参考：[Developing inside a Container](https://code.visualstudio.com/docs/devcontainers/containers)
<img src="https://hackmd.io/_uploads/B13kL7Gea.png" width="500" height="">

&nbsp;

インストール後､VScodeの画面をよくみると....

<img src="https://t907947.p.clickup-attachments.com/t907947/a0ec0686-3952-4149-b1a7-6c0245a2f93b/image.png" width="600" height="">

&nbsp;

<img src="https://hackmd.io/_uploads/BJZmvmfg6.png" width="300" height="">

左下に｢ >< ｣というマークが出てきます｡ このボタンを押すことで次の作業に移ることができます｡これでVScodeは一旦､準備完了です｡

&nbsp;

:::success
この｢Dev Containers｣だけでなく､他の拡張機能を含んだ拡張機能パック｢[Remote Development](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.vscode-remote-extensionpack)｣をインストールする方法もあるようです｡

![](https://t907947.p.clickup-attachments.com/t907947/c73afc83-d397-4826-b62c-de24f1d30ea1/image.png)
:::

&nbsp;

Dev containersに関する情報は以下の記事が参考になります｡ 
開発環境を揃える､という点でこの機能は非常に重要なことがわかります｡

#### 参考：[Devcontainer(Remote Container) いいぞという話 開発環境を整える](https://qiita.com/yoshii0110/items/c480e98cfe981e36dd56)

<img src="https://hackmd.io/_uploads/r1sDumfla.png" width="500" height="">
  

#### 参考：[開発コンテナ(Development Containers)を使おう](https://gist.github.com/heronshoes/4e707bbc92ceee60d71fc09007e01d02#%E9%96%8B%E7%99%BA%E3%82%B3%E3%83%B3%E3%83%86%E3%83%8A%E3%81%A8%E3%81%AF%E4%BD%95%E3%81%8B)

<img src="https://hackmd.io/_uploads/HkpVFXzga.png" width="400" height="">

&nbsp;
 
:::success
なお､自分は日本語で表示されるように拡張機能である｢[Japanese Language Pack for Visual Studio Code](https://marketplace.visualstudio.com/items?itemName=MS-CEINTL.vscode-language-pack-ja)｣もインストールしています｡ 

![](https://t907947.p.clickup-attachments.com/t907947/02f51be2-60bf-4e65-8518-2e9217d776d5/image.png)
:::

この拡張機能を使って､新しく環境構築などを行う(VScodeから自分にあった環境をつくる､など) ことも可能ですが､今回はGitHubにすでに用意されているテンプレートを使って簡単に実行できる方法をご紹介します(【STEP3】に記載)｡

&nbsp;

&nbsp;

## 【STEP2】Docker Desktopのインストール

次にDocker Desktopをインストールします｡ 下のリンクから可能です｡ 今回はMac(Apple chip)版をインストールします｡

#### [Docker公式サイト](https://www.docker.com/products/docker-desktop/)

<img src="https://hackmd.io/_uploads/Bkw-cXMla.png" width="500" height="">

&nbsp;

ダッシュボードを開くと以下のようになっています｡

<img src="https://t907947.p.clickup-attachments.com/t907947/3d83459a-08c3-47b3-bb92-0e89060865a6/image.png" width="550" height="">

&nbsp;

:::success
### Docker Desktop 以外の手段

Docker desktopを使用する以外にも代替手段があります｡ こちらは国内版バイオハッカソン23.9にて丹生さんより情報提供いただきました｡

1. lima + docker (macOS)
2. Rancher desktop ([https://rancherdesktop.io/](https://rancherdesktop.io/)) (macOS, moby): ただしRemote containerだと安定しないとのことです
3. orbstack ([https://orbstack.dev/](https://orbstack.dev/)) (macOS)
4. Docker on WSL (Windows)

:::

&nbsp;

&nbsp;

## 【STEP3】GitHubからテンプレートを取得する

ここまででVScodeとdockerのインストールが完了しました｡ 次にCWLを実行する環境のテンプレートをGitHubから取得します｡

以下のリポジトリ[tom-tan/cwl-for-remote-container-template](https://github.com/tom-tan/cwl-for-remote-container-template)にアクセスしてください｡今回はこのリポジトリをテンプレートにして環境を作っていきます｡ このテンプレートでは既にシンタックスハイライトの機能があるCWL(Rabix/benten)などの拡張機能が使用できるように準備されています｡

&nbsp;

#### [tom-tan/cwl-for-remote-container-template](https://github.com/tom-tan/cwl-for-remote-container-template)

<img src="https://hackmd.io/_uploads/Hy8-sXGgT.png" width="550" height="">

&nbsp;

ページ内の ｢Use this template｣(緑のボタン)をクリックし､｢Create a new repository｣を選択すると､自分のアカウントで新規リポジトリを作成することができます｡ 
__なお､GitHubのアカウントを持っていない場合はこのステップを飛ばしてください｡__

<img src="https://t907947.p.clickup-attachments.com/t907947/b7265827-aa40-451d-b251-472e6abec596/image.png" width="550" height="">

&nbsp;

次に `git clone` を行います(GitHubのアカウントがない場合は､tom-tan/cwl-for-remote-container-templateを､アカウントがある場合は､your_account/cwl-for-remote-container-template ということになります)｡

```bash=
# アカウントが無い場合
git clone https://github.com/tom-tan/cwl-for-remote-container-template 

#アカウントがある場合
git clone https://github.com/your_account/cwl-for-remote-container-template
```

&nbsp;

この作業が終了したら､つづいてVSCodeを開きます｡ VSCode画面左下の緑の｢ >< ｣マークを押すと､ 検索窓に以下のようなオプションが出てくるので､｢コンテナーでフォルダーを開く｣を選択し､先程`git clone`したローカルリポジトリを開きます｡

<img src="https://t907947.p.clickup-attachments.com/t907947/3e86b0e8-d5b9-42aa-8934-caefbd58b73c/image.png" width="550" height="">

最初の立ち上げ時には､5分程度かかりました｡
ログを見ていると環境構築のために色々されていることがわかります｡

<img src="https://t907947.p.clickup-attachments.com/t907947/f4bac0c6-adaa-4ea4-a794-734ab630d40d/image.png" width="550" height="">

&nbsp;

ターミナルを開いてみると､以下のように`/workspaces/togotv_shooting(repository_name)`となっています｡

<img src="https://t907947.p.clickup-attachments.com/t907947/ea721043-8c13-4415-be0f-9fa971a1342c/image.png" width="550" height="">

実際にCWL関連のツールは使えるようになっているのか見てみましょう｡
`cwl` と入力してtabを2回押すと...

<img src="https://hackmd.io/_uploads/SyI63Qze6.png" width="650" height="">

このように､cwltoolなど実行に必要なツールが導入されています｡

では､Dockerのコンテナが起動しているかどうかdocker desktopで確認してみると...

<img src="https://t907947.p.clickup-attachments.com/t907947/40fb1465-c5af-4bdf-abea-fc9cfb44a81d/image.png" width="550" height="">

上記のように､立ち上がっているのがわかります｡
これで環境構築が完了です｡

&nbsp;

## 【番外編】GitHub Codespacesで実行環境を作って作業する

ここまでは､ローカルの自分のマシンで行うことを前提に色々準備してきました｡ しかしながら､｢もっと楽に環境構築して動かしてみたい!!｣ という方もいらっしゃるかと思います｡ そこで活用できるのが｢GitHub Codespaces｣というクラウドでホストされている開発環境です｡ その概要は以下の日本語ドキュメントをご覧ください｡


#### 参考： [GitHub Codespaces の概要](https://docs.github.com/ja/codespaces/overview)

<img src="https://hackmd.io/_uploads/HJvDa7GgT.png" width="550" height="">

&nbsp;

先程テンプレートを取得する段階で､｢Use this template｣を押す時に気になった方がいらっしゃるかもしれませんが､この時､｢Create a new repository｣ と｢Open in a codespace｣と2つの選択肢があったかと思います｡ このとき｢Open in a codespace｣を選べばなんとブラウザでこの開発環境が立ち上がります! 

また､VScode経由でも開くことが可能です｡ ｢GitHub codespaces｣という拡張機能をインストールしてください｡その後｡左下の｢><｣ボタンを押してください｡

<img src="https://t907947.p.clickup-attachments.com/t907947/760c595c-7e08-47dd-b5c7-a862c2f71194/image.png" width="100" height="">

&nbsp;

次に､｢Create New Codespace｣をクリックして､立ち上げたいリポジトリを選択します｡

Create New Codespaceをクリックすると

<img src="https://t907947.p.clickup-attachments.com/t907947/b0aa9b0e-6370-4c91-b666-937df1847ee5/image.png" width="500" height="">

&nbsp;

リポジトリを選択して...

<img src="https://t907947.p.clickup-attachments.com/t907947/633cc294-92f3-477b-a9d6-bdfd6f142bf9/image.png" width="500" height="">

そうすると､自動的に環境が構築されていきます｡
今回はすでに環境を構築しているものを使用します｡なお､Codespaceで作成した環境は､自分のGitHubのページ(Your Codespaces)から確認できます｡最初の環境の立ち上げには同様に5分程度時間がかかります｡

:::warning
1ヶ月で使用できる時間には限りがあるようです｡
使用時間が全体の90%を超えると､警告のメールが届きました｡
:::

&nbsp;

:::danger
__修正1：Use this templateしなくてもcodespacesを開く例をサラッと説明__
__修正2：山本さんが送ってくれたスクショとかも貼る(ユーザー名隠す)__
:::

&nbsp;

&nbsp;

# zatsu-cwl-generatorを使ってcwlファイルを生成する

ここまではCWLに関する説明､および環境導入を行いました｡ 
ここからは､CWLファイルの記述､実行を行っていきます... ということですが､

&nbsp;

初めにgrepコマンドとwcコマンドを使ったワークフローをCWLによって記述する例を紹介します｡

実行するのは､`grep one mock.txt > grep_out.txt`です｡ 

```bash=
grep one mock.txt > grep_out.txt
```

#### 参考：[CWL Start Guide JP: CWL で書いてみる: コマンドラインツール](https://github.com/pitagora-network/pitagora-cwl/wiki/CWL-Start-Guide-JP#cwl-%E3%81%A7%E6%9B%B8%E3%81%84%E3%81%A6%E3%81%BF%E3%82%8B-%E3%82%B3%E3%83%9E%E3%83%B3%E3%83%89%E3%83%A9%E3%82%A4%E3%83%B3%E3%83%84%E3%83%BC%E3%83%AB)

&nbsp;

CWLファイルは記述する内容を YAMLかJSON の形式で記述し、｢.cwl ｣という拡張子でファイルに保存します。
実行時にこの CWL ファイルを実行エンジンに入力すると、ワークフローが実行される､という流れになっています｡まずはじめにスクリプトの最初の処理である `grep one mock.txt > grepout.txt` の処理をCWLファイルとして記述していきます｡

:::danger

__追記：https://view.commonwl.org/workflows?search=__
:::

&nbsp;

## zatsu-cwl-generatorを使おう

今回は､コマンドラインツールであるzatsu-cwl-genratorを使ってファイルを __出力__ してみます｡

参考：[雑に始めるCWL！をもっと雑に始めたい](https://qiita.com/tm_tn/items/2c789c5b3c28e3eb3c9a)

&nbsp;

まず､ターミナルを開いて､zatsu-cwl-generatorと入力します｡
次に､cwlファイルとして書きたい処理を '' で囲んで記入します｡ 今回は`grep`コマンドの処理を例に実行してみます｡

```bash=
zatsu-cwl-generator 'grep one ./data/mock.txt > grepout.txt'
```

&nbsp;

実行すると､標準出力に以下のようにcwlファイルが出力されます｡

```yaml=
#!/usr/bin/env cwl-runner
# Generated from: grep one mock.txt > grepout.txt
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
      location: mock.txt
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

この出力される形式はYAML形式です｡
shebang以下には､今回CWLを実行するためのフィールドとして､上から順に
`class` ､ `cwlversion`､`baseCommand`､ `arguments`､ `inputs` ､ `outputs`､ `stdout` が出力されています｡

この出力をリダイレクトしてファイルとして保存します｡
今回は､zatsu-generatorディレクトリに保存します｡

```bash=
zatsu-cwl-generator 'grep one mock.txt > grepout.txt' > ./zatsu_generator/grep_zatsu.cwl
```

:::success
### cwlversionはどれを使う?

出力されるフィールドで`cwlVersion` があります｡
ここには､cwltoolの準拠しているバージョンを書きます｡
現在､v1.0, v1.1, v1.2など複数のバージョンに対応しています｡
なお､この記事ではv1.0で書いていますが､基本的な書き方はどのバージョンでも大きく変わりません｡

<img src="https://t907947.p.clickup-attachments.com/t907947/cf8cf2c7-57b7-4173-bdee-21c202175c66/image.png" width="500" height="">

:::

&nbsp; 

### 記述が正しいか確認する

zatsu-cwl-genratorで出力されたファイルに対し､実際の実行前に記述が本当に正しいか確認することができます｡
`cwltool –-validate` コマンドを実行すると､記述したCWLファイルを評価することができます｡ 実際にやってみましょう｡
以下のようにコマンドを書きます｡

```bash=
cwltool --validate grep_zatsu.cwl
```

すると以下のように出力されました｡

```bash=
INFO /usr/local/bin/cwltool 3.1.20231016170136
INFO Resolved './zatsu_generator/grep_zatsu.cwl' to 'file:///workspaces/togotv_shooting/zatsu_generator/grep_zatsu.cwl'
./zatsu_generator/grep_zatsu.cwl is valid CWL.
```

今回の場合はエラーは確認されず､記述としては正しいようです｡ 
このように､記述がおかしい場合はコマンドで確認できる他､スクリプトを編集している際に､補助的に赤線で明示されますので参考にしてください｡

&nbsp;

### 実際に試してみる

ファイルの記載が正しいことを確認できたので､次に実際に`cwltool`というコマンドで試してみます(以降の操作はzatsu_generatorディレクトリでの作業です)｡ 

```bash=
cwltool grep_zatsu.cwl 
```

実行してみたのが以下になります｡

```bash=
INFO /usr/local/bin/cwltool 3.1.20231016170136
INFO Resolved 'grep_zatsu.cwl' to 'file:///workspaces/togotv_shooting/zatsu_generator/grep_zatsu.cwl'
INFO [job grep_zatsu.cwl] /tmp/zlsch9g1$ grep \
    one \
    /tmp/ppel7ebs/stgdd954ba7-9e15-45a6-8d5f-bfad5dff22b2/mock.txt > /tmp/zlsch9g1/grepout.txt
INFO [job grep_zatsu.cwl] completed success
{
    "all-for-debugging": [
        {
            "location": "file:///workspaces/togotv_shooting/zatsu_generator/grepout.txt",
            "basename": "grepout.txt",
            "class": "File",
            "checksum": "sha1$a972f6d93fec7529fd4af8344ca298eea43dfbc5",
            "size": 16,
            "path": "/workspaces/togotv_shooting/zatsu_generator/grepout.txt"
        }
    ],
    "out": {
        "location": "file:///workspaces/togotv_shooting/zatsu_generator/grepout.txt",
        "basename": "grepout.txt",
        "class": "File",
        "checksum": "sha1$a972f6d93fec7529fd4af8344ca298eea43dfbc5",
        "size": 16,
        "path": "/workspaces/togotv_shooting/zatsu_generator/grepout.txt"
    }
}INFO Final process status is success

```

無事ワークフローが成功し､ `grep_out.txt` が出力されました｡
このようにzatsu-cwl-generatorを使うことで､簡単にcwlファイルを作成することができます
 
&nbsp;

:::success
### エラーが発生した時

もしこの実行でエラーが発生した場合､ `--debug` オプションを追加してより詳しいエラーの結果を見ることができます｡

```bash=
cwltool --debug grep.cwl 
```
:::

&nbsp;

:::success
### `--help`オプションを活用しよう


`cwltool grep_zatsu.cwl --help`のように､cwlファイルの次に`--help`オプションをつけると､その __cwlファイル自体のヘルプを見ることができます__｡
どういうことか実際にやってみましょう｡

```bash=
cwltool grep_zatsu.cwl --help
INFO /usr/local/bin/cwltool 3.1.20231016170136
INFO Resolved 'grep_zatsu.cwl' to 'file:///workspaces/togotv_shooting/zatsu_generator/grep_zatsu.cwl'
usage: grep_zatsu.cwl [-h] [--mock_txt MOCK_TXT] [job_order]

positional arguments:
  job_order            Job input json file

options:
  -h, --help           show this help message and exit
  --mock_txt MOCK_TXT
```
このように､指定したパラメータ(今回は`--mock_txt`)に関する情報が取得できます｡

大文字の`MOCK_TXT`のあとの部分に具体的に説明を付け加える際には､__`doc`フィールドを追加することで可能になります｡__
例として､grep_zatsu.cwlに`doc`フィールドを以下のように書き加えました｡

```yaml=
#!/usr/bin/env cwl-runner
# Generated from: grep one mock.txt > grepout.txt
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
    doc: please input text file #記載を追加
    default:
      class: File
      location: mock.txt
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

再度実行すると以下のようになります｡
```bash=
cwltool grep_zatsu.cwl --help
INFO /usr/local/bin/cwltool 3.1.20231016170136
INFO Resolved 'grep_zatsu.cwl' to 'file:///workspaces/togotv_shooting/zatsu_generator/grep_zatsu.cwl'
usage: grep_zatsu.cwl [-h] [--mock_txt MOCK_TXT] [job_order]

positional arguments:
  job_order            Job input json file

options:
  -h, --help           show this help message and exit
  --mock_txt MOCK_TXT  please input text file
```

&nbsp;

参考：[user_guide 2.16 best-practices](https://www.commonwl.org/user_guide/topics/best-practices.html)

:::

&nbsp;

:::warning

## (発展編) 自分で修正する

上記のようにこのままでのファイルでも実際に実行することができます｡
しかし､このファイルを修正することでよりよい記述をすることができます｡
作成したgrep処理のファイルには､以下の部分に赤線が示されます｡

```yaml=
inputs:
  - id: one
    type: Any #赤線で示された部分
    default: one
```

エラーメッセージを見ると､ `Expecting one of: ['Directory', 'File', 'boolean', 'double', 'float', 'int', 'long', 'null', 'stderr', 'stdout', 'string']`という表示が出ています｡
ここでは文字列を入力するので､以下のように修正できます｡

```yaml=
inputs:
  - id: one
    type: string #stringに変更
    default: one
```

そうするとエラーメッセージが消え､`ー-help`オプションを使うと表示されるようになります｡

```bash=
cwltool grep_zatsu.cwl --help
INFO /usr/local/bin/cwltool 3.1.20231016170136
INFO Resolved 'grep_zatsu.cwl' to 'file:///workspaces/togotv_shooting/zatsu_generator/grep_zatsu.cwl'
usage: grep_zatsu.cwl [-h] [--one ONE] [--mock_txt MOCK_TXT] [job_order]

positional arguments:
  job_order            Job input json file

options:
  -h, --help           show this help message and exit
  --one ONE #新しく表示される部分
  --mock_txt MOCK_TXT  please input text file
```

このように､zatsu-cwl-generatorで生成されたファイルを修正しながら､CWLの文法を勉強していくということが可能です｡
このように､自分で修正しながら実際に __ワークフローを作っていく__ 例については次の記事で紹介しています｡

:::

:::info

## 参考リンク集

- 
:::