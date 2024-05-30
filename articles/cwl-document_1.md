---
title: "CWLの作成環境をVScodeの｢Dev Containers｣の機能を使って構築する (1)"
emoji: "📑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["CWL", "bioinformatics"]
published: true
---

__※今回の記事で使用したCWLのファイルをおいているリポジトリは以下からアクセスすることができます｡__
https://github.com/yonesora56/togotv_cwl_for_remote_container

&nbsp;

:::message
公開している記事のバージョンは2024年5月時点のものです｡
今後修正を重ねていく予定なので､コメントをいただけると幸いです｡
:::

# はじめに

皆さんは｢ワークフロー言語｣をご存知でしょうか? これらの言語は、一連の手順や操作を明示的に定義し、それらを連携させることで、より複雑な作業を効率的に行うことができます。
バイオインフォマティクス分野において､ワークフロー言語は重要な役割を担っています｡ 

:::message
__本記事の対象となる方__
■ ｢ワークフロー言語｣を知っているけど触ったことがない方
■ 動かしたいけど環境構築が難しそう...と考えている方
:::

# なぜワークフロー言語を使用するのか

バイオインフォマティクスにおけるデータ解析では、一つのツールのみで解析が終了することは極めて稀です｡ 
通常､複数のツールを組み合わせて、大量のデータに対して一連のプロセスを繰り返し実行する必要があります。
これらの作業手順は、__ワークフロー(またはパイプライン)__ と呼ばれます。
しかし、手動でこれらの手順を繰り返すと、ヒューマンエラーに加え､異なる実行環境による再現性の問題が発生することがあります。このような場合に、__ワークフロー言語__ を用いることで、各ステップを自動化し、かつ実行環境に依存せず解析の再現性を向上させる事ができます｡

@[card](https://www.youtube.com/watch?v=yInRH3YK3Ik&list=PL0uaKHgcG00aJSa233gkhBA2HHe0-Ha-B)

@[card](https://doi.org/10.1038/d41586-019-02619-z)

&nbsp;

現在､さまざまな種類のワークフロー言語(Snakemake, Nextflow, Workflow Description language(WDL)...)が存在していますが、このドキュメントでは **Common Workflow Language (CWL)** について環境構築とその実例をご紹介します。

https://www.commonwl.org/

&nbsp;

# なぜCWLを使うのか?
では次に､数多くあるワークフロー言語の中でも､なぜCWLなのか?ということについてここで説明します｡

## 1\. bentenなどの開発ツールが充実している
CWLでは､[Rabix benten](https://github.com/rabix/benten)や､VScodeの拡張機能 [vscode-cwl](https://marketplace.visualstudio.com/items?itemName=manabuishii.vscode-cwl)など､CWLのユーザーをサポートしてくれるツールが豊富に開発されています[^3]｡

https://www.commonwl.org/tools/

## 2\.自分の実行したい環境に合わせて最適なものが選択できる
CWLでは､複数のツールで実行することができます｡例えば､[cwltool](https://github.com/common-workflow-language/cwltool)に加え､ジョブスケジューラに対応している[Toil](https://github.com/DataBiosphere/toil)などが存在します｡自分の実行したい環境に合わせて選択肢が多いことが特徴です[^4]｡

https://www.commonwl.org/implementations/#what-can-execute-cwl-descriptions

&nbsp;

実際にCWLを使って記述されたバイオインフォマティクスにおける解析ワークフローは数多くあります｡ 例えば､ヒトゲノムバリアント検出ワークフローである｢ [ddbj/human-reseq](https://github.com/ddbj/human-reseq) ｣が挙げられます｡
https://github.com/ddbj/human-reseq

このように､バイオインフォマティクスに関するワークフローは多くのツールで導入されています｡
CWLについてより詳しく知りたい方は、下記に示している日本語のドキュメント[^5] [^6]や書籍も多くあるので、ぜひ参考にしてください。

https://github.com/pitagora-network/pitagora-cwl/wiki/CWL-Start-Guide-JP

https://oumpy.github.io/blog/2018/12/cwl.html

CWLの実行を行うエンジンであるcwltoolなどは、pipコマンドやcondaを使ってインストールすることが可能です。
しかしながら､環境構築は大変な場合があるかと思います。そこでこのドキュメントでは､作業するコンピュータの環境に依存せず、CWLの開発環境を作成して実行する方法を紹介します。

:::message
※なお､このドキュメントと統合TVの動画は､第13回国内版バイオハッカソン22.9､および第14回国内版バイオハッカソン23.9にてアイデアをいただいて作成しています｡ 主な内容はQiitaの記事[^7]がベースになっています｡ CWLに関してアドバイスをしてくださった石井さん､丹生さん､山本さんにこの場をお借りして御礼申し上げます｡

__(※ 制作が大変遅くなってしまい申し訳ありません!!!)__
:::

&nbsp;

# 環境構築編

## 【STEP1-1】VScodeのインストール

はじめに、コードエディターであるVisual Studio Code (VScode)のインストール方法を説明します。
特徴としては拡張機能が豊富に存在している部分であり､この記事でもこの部分を活用します｡

ダウンロードは以下のページにアクセスしておこないます｡ 皆さんも自分のコンピュータの環境に合わせて選んでください｡

https://code.visualstudio.com/download

この記事では､macOSの｢Apple silicon｣をクリックしてインストールして執筆しています｡

## 【STEP1-2】拡張機能を導入する

先程述べたように､VSCodeには豊富な拡張機能が存在します｡ 
サイドバー(ここでは左)の拡張機能のボタン(四角が4つあつまっている部分)を押すと､様々な拡張機能がMarketplaceで検索できます｡
ここで､｢[Dev Containers](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers)｣ と検索してください｡ 
こちらが必要なのでインストールします[^8](以前は｢Remote Container extension｣という名前だったようですが､どうやら変わったようです)｡

![Dev-container](https://storage.googleapis.com/zenn-user-upload/e5d7dc91c2e3-20240527.png)

https://code.visualstudio.com/docs/devcontainers/containers

&nbsp;

インストール後､VScodeの画面をよくみると....

![VScode-1](https://storage.googleapis.com/zenn-user-upload/ec4060167e94-20240527.png)

![VScode-2](https://storage.googleapis.com/zenn-user-upload/680a6dfcc6ec-20240527.png)
*左下に｢><｣マークが出現*

左下に｢ >< ｣というマークが出てきます｡ このボタンを押すことで次の作業に移ることができます｡これでVScodeは一旦､準備完了です｡

:::message
この｢Dev Containers｣だけでなく､他の拡張機能を含んだ拡張機能パック[Remote Development](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.vscode-remote-extensionpack)をインストールする方法もあるようです｡

![remote development](https://t907947.p.clickup-attachments.com/t907947/c73afc83-d397-4826-b62c-de24f1d30ea1/image.png)
:::

Dev containersに関する情報は以下の記事が参考になります[^9] [^10]｡ こちらも合わせてご覧ください｡

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

※ こちらは国内版バイオハッカソン23.9にて丹生さんより情報提供いただきました｡
:::

&nbsp;

## 【STEP3】GitHubからテンプレートを取得する

ここまででVScodeとdockerのインストールが完了しました｡ 次にCWLを実行する環境のテンプレートをGitHubから取得します｡

まず､以下に示しているリポジトリにアクセスしてください｡
今回はこのリポジトリ([tom-tan/cwl-for-remote-container-template](https://github.com/tom-tan/cwl-for-remote-container-template))をテンプレートにして環境を作っていきます｡ 
このテンプレートでは既にシンタックスハイライトの機能があるCWL(Rabix/benten)などの拡張機能が使用できるように準備されています｡

https://github.com/tom-tan/cwl-for-remote-container-template

ページ内の ｢Use this template｣(緑のボタン)をクリックし､｢Create a new repository｣を選択すると､自分のアカウントで新規リポジトリを作成することができます｡ 

:::message
__GitHubのアカウントを持っていない場合はこのステップを飛ばしてください｡__
:::

![template](https://storage.googleapis.com/zenn-user-upload/7a7a0a256bad-20240527.png)

次に `git clone` を行います(GitHubのアカウントがない場合は､tom-tan/cwl-for-remote-container-templateを､アカウントがある場合は､your_account/cwl-for-remote-container-template ということになります)｡

```bash
#アカウントが無い場合
git clone https://github.com/tom-tan/cwl-for-remote-container-template 

#アカウントがある場合(Use this template後の状態  your_accountは自分のアカウント名に置き換えてください)
git clone https://github.com/your_account/cwl-for-remote-container-template
```

:::message
この記事ではリポジトリの名前を`togotv_cwl_for_remote_container`に変更しています｡
:::

この作業が終了したら､つづいてVSCodeを開きます｡ VSCode画面左下の緑の｢ >< ｣マークを押すと､ 検索窓に以下のようなオプションが出てくるので､｢コンテナーでフォルダーを開く｣を選択し､先程`git clone`したローカルリポジトリを開きます｡

![VScode-3](https://storage.googleapis.com/zenn-user-upload/14917eb2b27b-20240527.png)

最初の立ち上げ時には､5分程度かかりました｡
ログを見ていると環境構築のために色々されていることがわかります｡
![VScode-4](https://storage.googleapis.com/zenn-user-upload/da6eb7c3671d-20240527.png)

ターミナルを開いてみると､以下のように`/workspaces/togotv_cwl_for_remote_container(repository_name)`となっています｡

![VScode-5](https://storage.googleapis.com/zenn-user-upload/bddab18fa4b7-20240527.png)

&nbsp;

実際にCWL関連のツールは使えるようになっているのか見てみましょう｡
`cwl` と入力してtabを2回押すと...

![VScode-6](https://storage.googleapis.com/zenn-user-upload/899cc2808da0-20240527.png)

このように､cwltoolなど実行に必要なツールが導入されています｡
では､Dockerのコンテナが起動しているかどうかdocker desktopで確認してみると...

![docker_dashboard2](https://storage.googleapis.com/zenn-user-upload/5c5d14a8f9df-20240527.png)

上記のように､立ち上がっているのがわかります｡
これで一旦環境構築が完了です｡

&nbsp;

## 【番外編】GitHub Codespacesで実行環境を作って作業する

ここまでは､ローカルの自分のマシンで行うことを前提に色々準備してきました｡ しかしながら､__｢もっと楽に環境構築して動かしてみたい!!｣__ という方もいらっしゃるかと思います｡ そこで活用できるのが｢GitHub Codespaces｣というクラウドでホストされている開発環境です｡ その概要は以下の日本語ドキュメントをご覧ください[^11]｡

https://docs.github.com/ja/codespaces/overview

先程テンプレートを取得する段階で､｢Use this template｣を押す時に気になった方がいらっしゃるかもしれませんが､この時､｢Create a new repository｣ と｢Open in a codespace｣と2つの選択肢があったかと思います｡ このとき｢Open in a codespace｣を選べばブラウザでこの開発環境が立ち上げることができます｡

また､VScode経由でも開くことが可能です｡ 
そのためには [｢GitHub codespaces｣](https://marketplace.visualstudio.com/items?itemName=GitHub.codespaces) という拡張機能をインストールしてください｡
その後､左下の｢><｣ボタンを押してください｡

![VScode-7](https://storage.googleapis.com/zenn-user-upload/e94f99c773f2-20240527.png)

次に､｢Create New Codespace｣をクリックして､立ち上げたいリポジトリを選択します｡
Create New Codespaceをクリックすると以下のような表示が出てきます｡

![VScode-8](https://storage.googleapis.com/zenn-user-upload/58e04cac7aa6-20240527.png)

ここから`Use this template`で取得したリポジトリを選択して... ()
![VScode-9](https://storage.googleapis.com/zenn-user-upload/968a26a0596e-20240527.png)

そうすると､自動的に環境が構築されていきます｡
今回はすでに環境を構築しているものを使用します｡なお､Codespaceで作成した環境は､自分のGitHubのページ(Your Codespaces)から確認できます｡最初の環境の立ち上げには同様に5分程度時間がかかります｡

:::message alert
1ヶ月で使用できる時間には限りがあるようです｡
自分の場合では､Codespacesの使用時間が(1ヶ月で割り当てられている)全体の90%を超えると､警告のメールが届きました｡
:::

:::message alert
このCodespacesを使った環境構築では､実は`Use this template`をしなくても立ち上げることができます｡
まず､[cwl-for-remote-container-template](https://github.com/tom-tan/cwl-for-remote-container-template)にアクセスします｡

`Use this template`ボタンの横に`V`があると思いますが､これを押すと､Open in a codespaceという選択肢が出てきます｡
![VScode-10](https://storage.googleapis.com/zenn-user-upload/3a5650a912bb-20240527.png)

こちらを選択すると､自動的にCodespacesが立ち上がります｡
![VScode-11](https://storage.googleapis.com/zenn-user-upload/9ae5837c0737-20240527.png)

より簡単に環境構築を行うことができますが､Codespacesの使用時間には注意が必要です｡
:::

&nbsp;

# zatsu-cwl-generatorを使ってcwlファイルを作る

ここまではCWLに関する説明､および環境導入を行いました｡ 
ここからは､CWLファイルの記述､実行を行っていきます... ということですが､このCWLファイルを簡単に生成できるツールを使って

初めにgrepコマンドとwcコマンドを使ったワークフローをCWLによって記述する例を紹介します｡

実行するのは､`grep one mock.txt > grep_out.txt`というプロセスです｡ 
以下のようなmock.txtを作成し､このファイルに対して､`one`という文字列をgrepで検索し､その結果をgrep_out.txtに出力します｡

https://github.com/yonesora56/togotv_cwl_for_remote_container/blob/master/zatsu_cwl/mock.txt

```bash
grep one mock.txt > grep_out.txt
```


CWLファイルは記述する内容を YAMLかJSON の形式で記述し、｢.cwl ｣という拡張子でファイルに保存します。
実行時にこのCWLファイルを実行エンジンに入力すると、ワークフローが実行される､という流れになっています｡
まずはじめにスクリプトの最初の処理である `grep one mock.txt > grepout.txt` の処理をCWLファイルとして記述していきます｡

今回は､上記の処理をCWLファイルとして生成することが可能なコマンドラインツールである[zatsu-cwl-genrator](https://github.com/tom-tan/zatsu-cwl-generator)[^12]を使ってファイルを出力してみます｡

https://github.com/tom-tan/zatsu-cwl-generator

https://qiita.com/tm_tn/items/2c789c5b3c28e3eb3c9a

&nbsp;

まず､ターミナルを開きます｡
今回は､`zatsu_cwl`というディレクトリを作成し､その中で作業を行います｡

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
今回は､zatsu-generatorディレクトリに保存します｡

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

### 記述が正しいか確認する

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

### 実際に実行してみる

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

このようにzatsu-cwl-generatorを使うことで､簡単にcwlファイルを作成することができます｡

&nbsp;

:::message
### エラーが発生した時
もしこの実行でエラーが発生した場合､ `--debug` オプションを追加してより詳しいエラーの結果を取得することができます｡
はじめての処理を実行する際には､このオプションをつけて実行することをおすすめします｡

```bash:
cwltool --debug grep_zatsu.cwl 
```
:::

&nbsp;

:::message
### `--help`オプションを活用しよう

`cwltool grep_zatsu.cwl --help`のように､cwlファイルの次に`--help`オプションをつけると､その __cwlファイル自体のヘルプを見ることができます [^13]__｡
どういうことか実際にやってみましょう｡

```bash
cwltool grep_zatsu.cwl --help
INFO /usr/local/bin/cwltool 3.1.20240508115724
INFO Resolved 'grep_zatsu.cwl' to 'file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl/grep_zatsu.cwl'
usage: grep_zatsu.cwl [-h] [--mock_txt MOCK_TXT] [job_order]

positional arguments:
  job_order            Job input json file

options:
  -h, --help           show this help message and exit
  --mock_txt MOCK_TXT
```
このように､指定したパラメータ(今回は`--mock_txt`)に関する情報が取得できます｡
ちなみに`cwltool --help grep_zatsu.cwl`とやるとcwltoolに関するhelpが出力されます｡

大文字の`MOCK_TXT`のあとの部分に具体的に説明を付け加える際には､__`doc`フィールドを追加することで可能になります｡__
例として､grep_zatsu.cwlに`doc`フィールドを以下のように書き加えました｡

https://github.com/yonesora56/togotv_cwl_for_remote_container/blob/master/zatsu_cwl/grep_zatsu_2.cwl

再度実行してみると以下のようになります｡

```bash:
cwltool grep_zatsu_2.cwl --help
INFO /usr/local/bin/cwltool 3.1.20240508115724
INFO Resolved 'grep_zatsu_2.cwl' to 'file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl/grep_zatsu_2.cwl'
usage: grep_zatsu_2.cwl [-h] [--mock_txt MOCK_TXT] [job_order]

positional arguments:
  job_order            Job input json file

options:
  -h, --help           show this help message and exit
  --mock_txt MOCK_TXT  please input text file
```
:::

&nbsp;

### (発展編) 自分で修正してみる
上記のように実際に実行することができることを確認しました｡
しかし､このファイルを修正することでより良い解析を実行することができます｡
実は､作成したgrep処理のファイル(`grep_zatsu.cwl`)には､以下の部分に赤線が示されていました｡

```yaml
inputs:
  - id: one
    type: Any #赤線で示されていた部分
    default: one
```
エラーメッセージを見ると､ `Expecting one of: ['Directory', 'File', 'boolean', 'double', 'float', 'int', 'long', 'null', 'stderr', 'stdout', 'string']`という表示が出ています｡
ここでは`one`という文字列(string)を入力するので､以下のように修正できます｡

https://github.com/yonesora56/togotv_cwl_for_remote_container/blob/master/zatsu_cwl/grep_zatsu_3.cwl#L9-L12

そうするとエラーメッセージが消え､`--help`オプションを使うとargumentsとして表示されるようになります｡

```bash
cwltool grep_zatsu_3.cwl --help
INFO /usr/local/bin/cwltool 3.1.20240508115724
INFO Resolved 'grep_zatsu_3.cwl' to 'file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl/grep_zatsu_3.cwl'
usage: grep_zatsu_3.cwl [-h] [--one ONE] [--mock_txt MOCK_TXT] [job_order]

positional arguments:
  job_order            Job input json file

options:
  -h, --help           show this help message and exit
  --one ONE # 新しく表示される部分
  --mock_txt MOCK_TXT  please input text file
```
このように､zatsu-cwl-generatorで生成されたファイルを修正しながら､CWLの文法を勉強していくということが可能です｡
自分で修正しながら実際に __ワークフローを作っていく__ 例については次の記事で紹介しています｡

&nbsp;

[^3]: [CWL公式サイト Development Tools](https://www.commonwl.org/tools/)
[^4]: [CWL公式サイト Implementations](https://www.commonwl.org/implementations/)
[^5]: [CWL日本語公式ドキュメント](https://github.com/pitagora-network/pitagora-cwl/wiki/CWL-Start-Guide-JP)
[^6]: [Common Workflow Language入門](https://oumpy.github.io/blog/2018/12/cwl.html)
[^7]: [指先一つで立ち上げる CWL ツール・ワークフロー作成環境](https://qiita.com/tm_tn/items/3fafe22e2c4a92a7f597)
[^8]: [Developing inside a Container](https://code.visualstudio.com/docs/devcontainers/containers)
[^9]: [Devcontainer(Remote Container) いいぞという話 開発環境を整える](https://qiita.com/yoshii0110/items/c480e98cfe981e36dd56)
[^10]: [開発コンテナ(Development Containers)を使おう](https://gist.github.com/heronshoes/4e707bbc92ceee60d71fc09007e01d02#%E9%96%8B%E7%99%BA%E3%82%B3%E3%83%B3%E3%83%86%E3%83%8A%E3%81%A8%E3%81%AF%E4%BD%95%E3%81%8B)
[^11]: [GitHub Codespaces の概要](https://docs.github.com/ja/codespaces/overview)
[^12]: [雑に始めるCWL！をもっと雑に始めたい](https://qiita.com/tm_tn/items/2c789c5b3c28e3eb3c9a)
[^13]: [user_guide 2.16 best-practices](https://www.commonwl.org/user_guide/topics/best-practices.html)