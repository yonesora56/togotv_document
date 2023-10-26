---
title: "CWLの作成環境をVScodeの｢Dev Containers｣の機能を使って構築する (2)"
emoji: "📑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["CWL", "bioinformatics"]
published: false
---

__※今回の記事で使用したCWLのファイルをおいているリポジトリは以下からアクセスすることができます｡__
https://github.com/yonezawa-sora/togotv_cwl_for_remote_container

:::message alert
この赤いフィールドは後ほど修正する点をメモしています
:::

# はじめに

:::message
__本記事の対象となる方__

■ 環境構築ができたので､もっとCWLをガシガシ書いていきたい人
■ CWLの文法を実際に書いてみて学びたい人

(1)の記事を見ていただいたあと､更にCWLを勉強したいという方向けの記事になります｡
:::

以前の記事では､環境構築や[zatsu-cwl-generator](https://github.com/tom-tan/zatsu-cwl-generator)の使い方をご紹介しました｡
この記事では､zatsu-cwl-generatorで生成されたcwlファイルを修正しながらワークフローを実際に記述する方法についてご紹介します｡ 
__実際に修正していく過程もこの記事では載せているので､ぜひ参考にしてください｡__

&nbsp;

# grepコマンドとwcコマンドをCWL化する

初めに(1)の記事で作ったgrepコマンドのcwlファイルに加え､wcコマンドの処理をCWLによって記述し､この2つを __連続して解析できるようにする__ 例を紹介します｡
実行するのは､`grep one mock.txt > grepout.txt` と `wc -l grepout.txt > wcout.txt` の2つです｡ 
この2つは､日本語ドキュメント(CWL Start Guide JP)[^1]でも説明されている例になります｡

```bash:example
grep one mock.txt > grepout.txt
wc -l grepout.txt > wcout.txt
```

&nbsp;

前回は環境構築と､zatsu-cwl-genratorを使ってCWLファイルの生成と実行を行いました｡
この記事では､更に手を動かす作業を行っていきます｡主役となるコマンドは`grep`と`wc`の2つです｡ 
今回記述する流れとしては以下の2ステップです｡
__Step1：コマンドの処理に関するcwlファイルを書く(今回は2つ)__
__Step2：ワークフロー全体を記述するcwlファイルを書く__

&nbsp;

## zatsu-cwl-generatorを使ってcwlファイルを書く

それでは実際に書いていきましょう｡ 
まず､前回書いていたgrepの処理に関するCWLファイルを以下に示します｡

https://github.com/yonezawa-sora/togotv_cwl_for_remote_container/blob/master/zatsu_generator/grep_zatsu.cwl

このファイルは､zatsu-cwl-generatorを使って生成したものを､少し修正しています｡
再度実行すると以下のようになります｡

:::details grep_zatsu.cwl実行結果
```bash:
cwltool ./zatsu_generator/grep_zatsu.cwl
INFO /usr/local/bin/cwltool 3.1.20231016170136
INFO Resolved './zatsu_generator/grep_zatsu.cwl' to 'file:///workspaces/togotv_shooting/zatsu_generator/grep_zatsu.cwl'
INFO [job grep_zatsu.cwl] /tmp/hyklfokj$ grep \
    one \
    /tmp/04eizij3/stg0ab8da43-ade2-4118-837c-e62d095ee2c3/mock.txt > /tmp/hyklfokj/grepout.txt
INFO [job grep_zatsu.cwl] completed success
{
    "all-for-debugging": [
        {
            "location": "file:///workspaces/togotv_shooting/grepout.txt",
            "basename": "grepout.txt",
            "class": "File",
            "checksum": "sha1$a972f6d93fec7529fd4af8344ca298eea43dfbc5",
            "size": 16,
            "path": "/workspaces/togotv_shooting/grepout.txt"
        }
    ],
    "out": {
        "location": "file:///workspaces/togotv_shooting/grepout.txt",
        "basename": "grepout.txt",
        "class": "File",
        "checksum": "sha1$a972f6d93fec7529fd4af8344ca298eea43dfbc5",
        "size": 16,
        "path": "/workspaces/togotv_shooting/grepout.txt"
    }
}INFO Final process status is success
```
:::

&nbsp;

以前の記事で紹介した過程と同様に､wcコマンドを使用したCWLファイルをzatsu-cwl-generatorで出力してみます｡

```bash:
zatsu-cwl-generator 'wc -l grepout.txt > wcout.txt' > wc_zatsu.cwl
```

出力された結果が以下になります｡
```yaml:
#!/usr/bin/env cwl-runner
# Generated from: wc -l grep_out.txt > wcout.txt
class: CommandLineTool
cwlVersion: v1.0
baseCommand: wc
arguments:
  - -l
  - $(inputs.l)
inputs:
  - id: l
    type: File
    default:
      class: File
      location: grep_out.txt
outputs:
  - id: all-for-debugging
    type:
      type: array
      items: [File, Directory]
    outputBinding:
      glob: "*"
  - id: out
    type: stdout
stdout: wcout.txt
```

こちらも同様に､実行の前に`--validate` オプションを使って評価してみます｡

```bash:
cwltool --validate wc_zatsu.cwl
INFO /usr/local/bin/cwltool 3.1.20231016170136
INFO Resolved 'wc_zatsu.cwl' to 'file:///workspaces/togotv_shooting/zatsu_generator/wc_zatsu.cwl'
wc_zatsu.cwl:14:7: Warning: Field 'location' contains undefined reference to
                   'file:///workspaces/togotv_shooting/zatsu_generator/grep_out.txt'
WARNING wc_zatsu.cwl:14:7: Warning: Field 'location' contains undefined reference to
                   'file:///workspaces/togotv_shooting/zatsu_generator/grep_out.txt'
wc_zatsu.cwl is valid CWL.
```

警告が出力されましたが､基本的な部分は大丈夫なようです｡
警告では､`location`フィールドが`grep_out.txt`というファイルを参照しているが､その記述形式が定義されていないというメッセージが出力されています｡
このまま実行する前に､CWLの勉強も兼ねて修正してみましょう(以下のトグルに修正の経緯を書いているのでお時間があれば参考にしてください)｡


:::details wc_zatsu.cwlの修正

警告にもあったように､`location`フィールドでは､ファイルをただ書くのではなく､`file://`からはじまる書き方(実行者側のファイルシステムのパス)にしたほうが良いようです[^2] [^3]｡
そこで以下のように修正しました｡

```yaml:
#!/usr/bin/env cwl-runner
# Generated from: wc -l grep_out.txt > wc_out.txt
class: CommandLineTool
cwlVersion: v1.0
baseCommand: wc
arguments:
  - -l
  - $(inputs.l)
inputs:
  - id: l
    type: File
    default:
      class: File
      location: file:///workspaces/togotv_shooting/zatsu_generator/grepout.txt # ここを修正
outputs:
  - id: all-for-debugging
    type:
      type: array
      items: [File, Directory]
    outputBinding:
      glob: "*"
  - id: out
    type: stdout
stdout: wc_out.txt
```

再度､`--validate`オプションを使って評価してみます｡

```bash:
cwltool --validate wc_zatsu.cwl
INFO /usr/local/bin/cwltool 3.1.20231016170136
INFO Resolved 'wc_zatsu.cwl' to 'file:///workspaces/togotv_shooting/zatsu_generator/wc_zatsu.cwl'
wc_zatsu.cwl is valid CWL.
```
無事､修正されました｡
このように､`--validate`オプションはCWLファイルの記述が正しいかどうかを確認するのに便利です｡
:::

警告された部分を修正したので､次に実際に実行してみましょう｡ (`--debug`オプションをつけて行いました)

:::details cwltool実行結果
```bash:
cwltool --debug wc_zatsu.cwl
INFO /usr/local/bin/cwltool 3.1.20231016170136
INFO Resolved 'wc_zatsu.cwl' to 'file:///workspaces/togotv_shooting/zatsu_generator/wc_zatsu.cwl'
DEBUG Parsed job order from command line: {
    "__id": "wc_zatsu.cwl",
    "l": {
        "class": "File",
        "location": "file:///workspaces/togotv_shooting/zatsu_generator/grepout.txt"
    }
}
DEBUG [job wc_zatsu.cwl] initializing from file:///workspaces/togotv_shooting/zatsu_generator/wc_zatsu.cwl
DEBUG [job wc_zatsu.cwl] {
    "l": {
        "class": "File",
        "location": "file:///workspaces/togotv_shooting/zatsu_generator/grepout.txt",
        "size": 16,
        "basename": "grepout.txt",
        "nameroot": "grepout",
        "nameext": ".txt"
    }
}
DEBUG [job wc_zatsu.cwl] path mappings is {
    "file:///workspaces/togotv_shooting/zatsu_generator/grepout.txt": [
        "/workspaces/togotv_shooting/zatsu_generator/grepout.txt",
        "/tmp/307b8iig/stg0c1080cc-90eb-40a9-b697-064ab90b3855/grepout.txt",
        "File",
        true
    ]
}
DEBUG [job wc_zatsu.cwl] command line bindings is [
    {
        "position": [
            -1000000,
            0
        ],
        "datum": "wc"
    },
    {
        "position": [
            0,
            0
        ],
        "datum": "-l"
    },
    {
        "position": [
            0,
            1
        ],
        "valueFrom": "$(inputs.l)"
    }
]
DEBUG [job wc_zatsu.cwl] initial work dir {}
INFO [job wc_zatsu.cwl] /tmp/s1835wqr$ wc \
    -l \
    /tmp/307b8iig/stg0c1080cc-90eb-40a9-b697-064ab90b3855/grepout.txt > /tmp/s1835wqr/wcout.txt
DEBUG Could not collect memory usage, job ended before monitoring began.
INFO [job wc_zatsu.cwl] completed success
DEBUG [job wc_zatsu.cwl] outputs {
    "all-for-debugging": [
        {
            "location": "file:///tmp/s1835wqr/wcout.txt",
            "basename": "wcout.txt",
            "nameroot": "wcout",
            "nameext": ".txt",
            "class": "File",
            "checksum": "sha1$bf7e1060bfb4a5e4659788eb512d14297d5cfd93",
            "size": 68,
            "http://commonwl.org/cwltool#generation": 0
        }
    ],
    "out": {
        "location": "file:///tmp/s1835wqr/wcout.txt",
        "basename": "wcout.txt",
        "nameroot": "wcout",
        "nameext": ".txt",
        "class": "File",
        "checksum": "sha1$bf7e1060bfb4a5e4659788eb512d14297d5cfd93",
        "size": 68,
        "http://commonwl.org/cwltool#generation": 0
    }
}
DEBUG [job wc_zatsu.cwl] Removing input staging directory /tmp/307b8iig
DEBUG [job wc_zatsu.cwl] Removing temporary directory /tmp/42prtgxi
DEBUG Moving /tmp/s1835wqr/wcout.txt to /workspaces/togotv_shooting/zatsu_generator/wcout.txt
DEBUG Removing intermediate output directory /tmp/s1835wqr
{
    "all-for-debugging": [
        {
            "location": "file:///workspaces/togotv_shooting/zatsu_generator/wcout.txt",
            "basename": "wcout.txt",
            "class": "File",
            "checksum": "sha1$bf7e1060bfb4a5e4659788eb512d14297d5cfd93",
            "size": 68,
            "path": "/workspaces/togotv_shooting/zatsu_generator/wcout.txt"
        }
    ],
    "out": {
        "location": "file:///workspaces/togotv_shooting/zatsu_generator/wcout.txt",
        "basename": "wcout.txt",
        "class": "File",
        "checksum": "sha1$bf7e1060bfb4a5e4659788eb512d14297d5cfd93",
        "size": 68,
        "path": "/workspaces/togotv_shooting/zatsu_generator/wcout.txt"
    }
}INFO Final process status is success
```
:::

実行が成功したようです｡
結果を見てみると､4としっかりカウントされていました｡

```txt:
4 /tmp/307b8iig/stg0c1080cc-90eb-40a9-b697-064ab90b3855/grepout.txt
```

これで2つのCWLファイルが揃いました｡次に､この2つを __一つのコマンドで実行できる__ ようにするファイルを用意します｡

&nbsp;

&nbsp;

# ワークフローの記述を行う

これまで作ってきたファイルは一つの処理を記述したものです｡
その場合は､`class`フィールドに`CommandLineTool`と記述します｡

https://github.com/yonezawa-sora/togotv_cwl_for_remote_container/blob/master/zatsu_generator/grep_zatsu.cwl#L1-L3

一方､これから作るファイルは､この`class`フィールドに`Workflow`と記述します｡
以下から`grep-and-count.cwl`というファイルを作成していきます｡このワークフローの記述は現在zatu-cwl-generatorではサポートされていないので､手動で記述していきます｡

:::message
詳細な説明はトグルに書いていますので､お時間があれば参考にしてください｡
:::

::::details grep-and-count.cwlの記述

まずはじめに､ワークフローの記述を行うことを明示するために､`class`フィールドに`Workflow`と記述します｡

```yaml:
#!/usr/bin/env cwl-runner
cwlVersion: v1.0
class: Workflow
```
&nbsp;

続いて`inputs`, `outputs`, そしてワークフローの手順を記述する`steps` について注目します｡ 

`inputs`フィールド：ここでは､ __｢ワークフロー｣__ として必要な入力のパラメータについて記述します｡

```yaml
#!/usr/bin/env cwl-runner
cwlVersion: v1.0
class: Workflow
inputs:
  grep_pattern:
    type: string
  target_file:
    type: File
```
今回は､最初のステップである｢grep｣コマンドの抜き出すパターンと､対象のファイルの2つのパラメータと､それぞれのパラメータのタイプを記述しています｡

&nbsp;

`outputs`フィールド：ここでは､｢wc｣コマンド出力のパラメータを記述します｡

```yaml:
#!/usr/bin/env cwl-runner
cwlVersion: v1.0
class: Workflow
inputs:
  grep_pattern:
    type: string
  target_file:
    type: File 
outputs:
  counts:
    type: File
    outputSource: 2_wc/out
```
`outputSource` は､ワークフロー全体の出力を明示的に指定するために使用されるフィールドであり､特定のステップの出力をワークフロー全体の出力としてマッピングします｡

&nbsp;

`steps`フィールド：ここでは､具体的なワークフローの手順を記述します｡ それぞれのステップでのインプット､アウトプットを"in"､"out"として記述します｡ 
`run`フィールドには先程作成したCommandLinetoolの処理を記述したファイルを記述しておきます｡ 
なお､`out`フィールドでは､リスト形式(\[ \] で囲む)で書くことで､複数の出力を書くことができます｡ 

::::

実際に書いてみたのが以下になります｡

```yaml:
#!/usr/bin/env cwl-runner
cwlVersion: v1.0
class: Workflow
inputs:
  grep_pattern:
    type: string
  target_file:
    type: File
outputs:
  counts:
    type: File
    outputSource: 2_wc/out
steps:
  1_grep:
    run: grep_zatsu.cwl
    in:
      one: grep_pattern
      mock_txt: target_file
    out: [out]
  2_wc:
    run: wc_zatsu.cwl
    in:
      grep_file: 1_grep/out
    out: [out] 
```

&nbsp;

## 実際にワークフローを実行してみる

それでは実際にこのワークフローを実行してみましょう｡ 
実行には今回､cwl-runnerを使用して実行します｡

```bash:
cwl-runner grep-and-count.cwl --grep_pattern one --target_file ./grepout.txt
INFO /usr/local/bin/cwl-runner 3.1.20231016170136
INFO Resolved 'grep-and-count.cwl' to 'file:///workspaces/togotv_shooting/zatsu_generator/grep-and-count.cwl'
INFO [workflow ] start
INFO [workflow ] starting step 1_grep
INFO [step 1_grep] start
INFO [job 1_grep] /tmp/51srtkt0$ grep \
    one \
    /tmp/k89vh6vf/stg4d1cfe72-87be-4e6a-8243-0bd82be27e8b/grepout.txt > /tmp/51srtkt0/grepout.txt
INFO [job 1_grep] completed success
INFO [step 1_grep] completed success
INFO [workflow ] starting step 2_wc
INFO [step 2_wc] start
INFO [job 2_wc] /tmp/bkrv17e7$ wc \
    -l \
    /tmp/_1sj1zbz/stg729c8fba-f536-4f74-a7fe-e9dc49ebd3e7/grepout.txt > /tmp/bkrv17e7/wc_out.txt
INFO [job 2_wc] completed success
INFO [step 2_wc] completed success
INFO [workflow ] completed success
{
    "counts": {
        "location": "file:///workspaces/togotv_shooting/zatsu_generator/wc_out.txt",
        "basename": "wc_out.txt",
        "class": "File",
        "checksum": "sha1$58b721e8d56d12e12a31f7ec435491bf3faa767f",
        "size": 68,
        "path": "/workspaces/togotv_shooting/zatsu_generator/wc_out.txt"
    }
}INFO Final process status is success

```

このように処理され､最終的にwcout.txtが出力されます｡

# 参考リンク集

[^1]: [CWL Start Guide JP: CWL で書いてみる: コマンドラインツール](https://github.com/pitagora-network/pitagora-cwl/wiki/CWL-Start-Guide-JP#cwl-%E3%81%A7%E6%9B%B8%E3%81%84%E3%81%A6%E3%81%BF%E3%82%8B-%E3%82%B3%E3%83%9E%E3%83%B3%E3%83%89%E3%83%A9%E3%82%A4%E3%83%B3%E3%83%84%E3%83%BC%E3%83%AB)
[^2]: [CWL実行時に、パラメータとしてファイルを渡したときの location や path](https://qiita.com/manabuishiirb/items/8e9747ce3d902f5fba0a)
[^3]: [Common Workflow Language (CWL) Command Line Tool Description, v1.0.2 5.1.5 File](https://www.commonwl.org/v1.0/CommandLineTool.html#File)