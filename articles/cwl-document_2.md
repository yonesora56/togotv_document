---
title: "CWLの作成環境をVScodeの｢Dev Containers｣の機能を使って構築する (2)"
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

前回の記事はこちらです
https://zenn.dev/sorayone/articles/cwl-document_1

# はじめに

:::message
__本記事の対象となる方__
■ 環境構築ができたので､次のステップに進みたい方
■ CWLの文法を実際に書いてみて学びたい方

(1)の記事を見ていただいたあと､更にCWLを勉強したいという方向けの記事になります｡
:::

前回の記事では､環境構築および[zatsu-cwl-generator](https://github.com/tom-tan/zatsu-cwl-generator)によるCWLファイルの生成し実行してみました｡
この記事では､zatsu-cwl-generatorで生成されたCWLファイルをもう一つ書き､更にこの2つのCWLファイルを実行するワークフローを記述する方法についてご紹介します｡ 
今回記述する流れとしては以下の2ステップです｡

__(1)：`wc`コマンドの処理に関するCWLファイルで記述する__
__(2)：以前作成した`grep`コマンドおよび`wc`コマンドのワークフローを実行するCWLファイルを書く__

:::message
実際に修正していく過程もこの記事では載せているので(トグルになっている部分です)､ぜひ参考にしてください｡
:::

&nbsp;

# `wc`コマンドによる処理をCWLで記述する

初めに前回の [記事](https://zenn.dev/sorayone/articles/cwl-document_1#zatsu-cwl-generator%E3%82%92%E4%BD%BF%E3%81%A3%E3%81%A6cwl%E3%83%95%E3%82%A1%E3%82%A4%E3%83%AB%E3%82%92%E4%BD%9C%E6%88%90%E3%81%99%E3%82%8B)で作成したgrepコマンドのCWLファイルに加え､wcコマンドの処理をCWLによって記述します｡

```bash:example
grep one mock.txt > grepout.txt
wc -l grepout.txt > wcout.txt # grepout.txtの行数をカウントする(この記事ではこの処理を記述)
```
grepout.txtの処理で得られた結果に対し､`wc`コマンドを使って行数をカウントする処理を行います｡

https://github.com/yonesora56/togotv_cwl_for_remote_container/blob/master/zatsu_cwl/grepout.txt

以前の記事で紹介したプロセスと同様に､`wc`コマンドを使用したCWLファイルをzatsu-cwl-generatorで出力してみましょう｡
簡単にまとめると以下の通りです｡

```text
STEP1: zatsu-cwl-generatorを使ってwcコマンドの処理のCWLファイルを生成する
STEP2: 生成したCWLファイルの評価を行う
STEP3: 生成したCWLファイルを実行する
```

## zatsu-cwl-generatorを使って`wc`コマンドのCWLファイルを生成する

それでは実際に書いていきましょう｡ 
まず､前回 zatsu-cwl-generatorを使って出力した`grep`の処理に関するCWLファイルを以下に示します｡

https://github.com/yonesora56/togotv_cwl_for_remote_container/blob/master/zatsu_cwl/grep_zatsu_v3.cwl

:::details grep_zatsu_v3.cwl実行結果
再度実行すると以下のようになります｡
```bash:
cwltool grep_zatsu_v3.cwl
INFO /usr/local/bin/cwltool 3.1.20240508115724
INFO Resolved 'grep_zatsu_v3.cwl' to 'file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl/grep_zatsu_v3.cwl'
INFO [job grep_zatsu_v3.cwl] /tmp/arfu8dvc$ grep \
    one \
    /tmp/gc3kb5xk/stg8b7a268e-1c35-4d52-9ebf-e4edc53819d5/mock.txt > /tmp/arfu8dvc/grepout.txt
INFO [job grep_zatsu_v3.cwl] completed success
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
:::

&nbsp;

同じように`wc`コマンドの処理を記述してみましょう｡
まずはじめに､以下のようにzatsu-cwl-generatorを使って`wc`コマンドのプロセスを生成します｡

```bash:
zatsu-cwl-generator 'wc -l grepout.txt > wcout.txt' > wc_zatsu.cwl
```

出力された結果が以下になります｡

https://github.com/yonesora56/togotv_cwl_for_remote_container/blob/master/zatsu_cwl/wc_zatsu.cwl

## 記述が正しいかチェックする

こちらも前回と同様に､実行の前に`cwltool --validate` を使って評価してみます｡

```bash:
cwltool --validate wc_zatsu.cwl
INFO /usr/local/bin/cwltool 3.1.20240508115724
INFO Resolved 'wc_zatsu.cwl' to 'file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl/wc_zatsu.cwl'
wc_zatsu.cwl is valid CWL.
```
生成されたファイルは問題ないようです｡

## 実際に実行する

それではこのファイルを用いて､実際に実行してみましょう｡ (`--debug`オプションをつけて行いました)

:::details cwltool実行結果
```bash:
cwltool --debug wc_zatsu.cwl
INFO /usr/local/bin/cwltool 3.1.20240508115724
INFO Resolved 'wc_zatsu.cwl' to 'file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl/wc_zatsu.cwl'
DEBUG Parsed job order from command line: {
    "__id": "wc_zatsu.cwl",
    "l": {
        "class": "File",
        "location": "file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl/grepout.txt"
    }
}
DEBUG [job wc_zatsu.cwl] initializing from file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl/wc_zatsu.cwl
DEBUG [job wc_zatsu.cwl] {
    "l": {
        "class": "File",
        "location": "file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl/grepout.txt",
        "size": 16,
        "basename": "grepout.txt",
        "nameroot": "grepout",
        "nameext": ".txt"
    }
}
DEBUG [job wc_zatsu.cwl] path mappings is {
    "file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl/grepout.txt": [
        "/workspaces/togotv_cwl_for_remote_container/zatsu_cwl/grepout.txt",
        "/tmp/7ik0tsqf/stg9ffecc84-0735-4aaf-a92c-63d6c8ce9339/grepout.txt",
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
INFO [job wc_zatsu.cwl] /tmp/zwoidn60$ wc \
    -l \
    /tmp/7ik0tsqf/stg9ffecc84-0735-4aaf-a92c-63d6c8ce9339/grepout.txt > /tmp/zwoidn60/wcout.txt
DEBUG Could not collect memory usage, job ended before monitoring began.
INFO [job wc_zatsu.cwl] completed success
DEBUG [job wc_zatsu.cwl] outputs {
    "all-for-debugging": [
        {
            "location": "file:///tmp/zwoidn60/wcout.txt",
            "basename": "wcout.txt",
            "nameroot": "wcout",
            "nameext": ".txt",
            "class": "File",
            "checksum": "sha1$57d64e066b00b4788870f2f701df156f2709687d",
            "size": 68,
            "http://commonwl.org/cwltool#generation": 0
        }
    ],
    "out": {
        "location": "file:///tmp/zwoidn60/wcout.txt",
        "basename": "wcout.txt",
        "nameroot": "wcout",
        "nameext": ".txt",
        "class": "File",
        "checksum": "sha1$57d64e066b00b4788870f2f701df156f2709687d",
        "size": 68,
        "http://commonwl.org/cwltool#generation": 0
    }
}
DEBUG [job wc_zatsu.cwl] Removing input staging directory /tmp/7ik0tsqf
DEBUG [job wc_zatsu.cwl] Removing temporary directory /tmp/uc8ubcn5
DEBUG Moving /tmp/zwoidn60/wcout.txt to /workspaces/togotv_cwl_for_remote_container/zatsu_cwl/wcout.txt
DEBUG Removing intermediate output directory /tmp/zwoidn60
{
    "all-for-debugging": [
        {
            "location": "file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl/wcout.txt",
            "basename": "wcout.txt",
            "class": "File",
            "checksum": "sha1$57d64e066b00b4788870f2f701df156f2709687d",
            "size": 68,
            "path": "/workspaces/togotv_cwl_for_remote_container/zatsu_cwl/wcout.txt"
        }
    ],
    "out": {
        "location": "file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl/wcout.txt",
        "basename": "wcout.txt",
        "class": "File",
        "checksum": "sha1$57d64e066b00b4788870f2f701df156f2709687d",
        "size": 68,
        "path": "/workspaces/togotv_cwl_for_remote_container/zatsu_cwl/wcout.txt"
    }
}INFO Final process status is success
```
:::

実行が成功したようです!
`wc_zatsu.cwl`の実行結果として､`wcout.txt`が出力されました｡

https://github.com/yonesora56/togotv_cwl_for_remote_container/blob/master/zatsu_cwl/wcout.txt

このままでも実行できることが確認できましたが､後のワークフローでのプロセスのために､以下のように修正しました｡

:::details 少し修正しました
### `arguments`フィールドの修正

現時点のバージョンでは､`arguments`フィールドの`grep`コマンドで出力されたファイルの部分が`l`になっています｡
これは少しわかりにくい(再現性が低くなってしまう)ので､バージョン2のファイルでは`- $(inputs.grep_out)`に変えて､`inputs`フィールドの名前も`grep_out`に変更しています｡

https://github.com/yonesora56/togotv_cwl_for_remote_container/blob/master/zatsu_cwl/wc_zatsu.cwl#L6-L8

https://github.com/yonesora56/togotv_cwl_for_remote_container/blob/master/zatsu_cwl/wc_zatsu_v2.cwl#L6-L8

次のワークフローのファイルではバージョン2を使用します｡
:::

これで2つのCWLファイルが揃いました｡一から書かずにCWLファイルが生成できるなんて､本当にすごいです!
次に､この`grep`と`wc`の処理を __一つのコマンドで実行できる__ ようにするファイル､すなわち __ワークフローを実行するファイル__ を作成します｡

&nbsp;

# ワークフローを記述する

これまで作ってきたファイルは一つの処理を記述したものです｡
その場合は､`class`フィールドに`CommandLineTool`と記述しています｡

https://github.com/yonezawa-sora/togotv_cwl_for_remote_container/blob/master/zatsu_cwl/grep_zatsu_v3.cwl#L3

一方､これから作るファイルは､この`class`フィールドに`Workflow`と記述します｡
以下から`grep-and-count.cwl`というファイルを作成していきます｡
このワークフローの記述は現在zatu-cwl-generatorではサポートされていないので､手動で記述していきます｡
このワークフローで使用するファイルは少し修正を加えた`grep_zatsu_v3.cwl`と`wc_zatsu_v2.cwl`です｡

:::message
書き方の説明は以下のトグルに書いていますので､お時間があれば参考にしてください｡
:::

::::details grep-and-count.cwlの記述

### `Class`フィールドの記述
まずはじめに､ワークフローの記述を行うことを明示するために､`class`フィールドに`Workflow`と記述します｡

https://github.com/yonesora56/togotv_cwl_for_remote_container/blob/master/zatsu_cwl/grep-and-count.cwl#L1-L3

続いて`inputs`, `outputs`, そしてワークフローの手順を記述する`steps` について注目します｡ 

### `inputs`フィールドの記述
`inputs`フィールド：ここでは､ __"ワークフロー"__ として必要な入力のパラメータについて記述します｡

https://github.com/yonesora56/togotv_cwl_for_remote_container/blob/master/zatsu_cwl/grep-and-count.cwl#L4-L8

今回は､最初のステップである｢grep｣コマンドの抜き出すパターンと､対象のファイルの2つのパラメータというふうに､それぞれのパラメータのタイプを記述しています｡

### `outputs`フィールドの記述

`outputs`フィールド：ここでは､｢wc｣コマンド出力のパラメータを記述します｡

https://github.com/yonesora56/togotv_cwl_for_remote_container/blob/master/zatsu_cwl/grep-and-count.cwl#L9-L15

`outputSource` は､ワークフロー全体の出力を明示的に指定するために使用されるフィールドであり､特定のステップの出力をワークフロー全体の出力としてマッピングします｡

### `steps`フィールドの記述

`steps`フィールド：ここでは､具体的なワークフローの手順を記述します｡ それぞれのステップでのインプット､アウトプットを"in"､"out"として記述します｡ 
`run`フィールドには先程作成したCommandLinetoolの処理を記述したファイルを記述しておきます｡ 
なお､`out`フィールドでは､リスト形式(\[ \] で囲む)で書くことで､複数の出力を書くことができます｡ 

https://github.com/yonesora56/togotv_cwl_for_remote_container/blob/master/zatsu_cwl/grep-and-count.cwl#L16-L27
::::

以下のコードが実際に記述した`grep-and-count.cwl`のファイルです｡

https://github.com/yonesora56/togotv_cwl_for_remote_container/blob/master/zatsu_cwl/grep-and-count.cwl

&nbsp;

## ワークフローを評価する

このファイルもこれまでと同じように`cwltool --validate`を使ってチェックしてみましょう｡

```bash:
cwltool --validate grep-and-count.cwl
INFO /usr/local/bin/cwltool 3.1.20240508115724
INFO Resolved 'grep-and-count.cwl' to 'file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl/grep-and-count.cwl'
grep-and-count.cwl is valid CWL.
```
大丈夫のようです! それでは次に進みます｡

## CWLviewerで可視化してみる

次に､このワークフローの全体像を確認するため､[CWLviewer](https://view.commonwl.org/)を使って可視化してみましょう｡

https://view.commonwl.org/workflows/github.com/yonesora56/togotv_cwl_for_remote_container/blob/master/zatsu_cwl/grep-and-count.cwl

![CWLviewer result](https://storage.googleapis.com/zenn-user-upload/5fe94034b62e-20240612.png)

`grep`コマンドの結果とそこから`wc`コマンドに渡されている様子が確認できます!
(__この自分が書いたCWLファイルが可視化されるのは本当にかっこいいのでぜひ試してみてください!__) 

&nbsp;

## ワークフローを実行してみる

それでは実際にこのワークフローを実行してみましょう｡ 
実行には今回､cwl-runnerを使用して実行します｡ また､ワークフローの実行結果がわかりやすいように､`workflow_result`ディレクトリを新しく作成し､その中に結果を保存していきます｡`--outdir`オプションでそのディレクトリを指定しています｡

```bash:
cwl-runner --debug --outdir ./workflow_result/ grep-and-count.cwl --grep_pattern one --target_file ./grepout.txt
```

:::details grep-and-count.cwlの実行
```bash:
cwl-runner --debug --outdir ./workflow_result/ grep-and-count.cwl --grep_pattern one --target_file ./grepout.txt
INFO /usr/local/bin/cwl-runner 3.1.20240508115724
INFO Resolved 'grep-and-count.cwl' to 'file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl/grep-and-count.cwl'
DEBUG Parsed job order from command line: {
    "__id": "grep-and-count.cwl",
    "grep_pattern": "one",
    "target_file": {
        "class": "File",
        "location": "file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl/grepout.txt"
    }
}
DEBUG [workflow ] initialized from file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl/grep-and-count.cwl
INFO [workflow ] start
DEBUG [workflow ] inputs {
    "grep_pattern": "one",
    "target_file": {
        "class": "File",
        "location": "file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl/grepout.txt",
        "size": 16,
        "basename": "grepout.txt",
        "nameroot": "grepout",
        "nameext": ".txt"
    }
}
INFO [workflow ] starting step 1_grep
DEBUG [step 1_grep] job input {
    "file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl/grep-and-count.cwl#1_grep/mock_txt": {
        "class": "File",
        "location": "file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl/grepout.txt",
        "size": 16,
        "basename": "grepout.txt",
        "nameroot": "grepout",
        "nameext": ".txt"
    },
    "file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl/grep-and-count.cwl#1_grep/one": "one"
}
DEBUG [step 1_grep] evaluated job input to {
    "file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl/grep-and-count.cwl#1_grep/mock_txt": {
        "class": "File",
        "location": "file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl/grepout.txt",
        "size": 16,
        "basename": "grepout.txt",
        "nameroot": "grepout",
        "nameext": ".txt"
    },
    "file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl/grep-and-count.cwl#1_grep/one": "one"
}
INFO [step 1_grep] start
DEBUG [job 1_grep] initializing from file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl/grep_zatsu_v3.cwl as part of step 1_grep
DEBUG [job 1_grep] {
    "mock_txt": {
        "class": "File",
        "location": "file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl/grepout.txt",
        "size": 16,
        "basename": "grepout.txt",
        "nameroot": "grepout",
        "nameext": ".txt"
    },
    "one": "one"
}
DEBUG [job 1_grep] path mappings is {
    "file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl/grepout.txt": [
        "/workspaces/togotv_cwl_for_remote_container/zatsu_cwl/grepout.txt",
        "/tmp/vslvo1xy/stg8dfd1268-9a09-422a-b392-90caffe7ea6b/grepout.txt",
        "File",
        true
    ]
}
DEBUG [job 1_grep] command line bindings is [
    {
        "position": [
            -1000000,
            0
        ],
        "datum": "grep"
    },
    {
        "position": [
            0,
            0
        ],
        "valueFrom": "$(inputs.one)"
    },
    {
        "position": [
            0,
            1
        ],
        "valueFrom": "$(inputs.mock_txt)"
    }
]
DEBUG [job 1_grep] initial work dir {}
INFO [job 1_grep] /tmp/nibotqbd$ grep \
    one \
    /tmp/vslvo1xy/stg8dfd1268-9a09-422a-b392-90caffe7ea6b/grepout.txt > /tmp/nibotqbd/grepout.txt
DEBUG Could not collect memory usage, job ended before monitoring began.
INFO [job 1_grep] completed success
DEBUG [job 1_grep] outputs {
    "all-for-debugging": [
        {
            "location": "file:///tmp/nibotqbd/grepout.txt",
            "basename": "grepout.txt",
            "nameroot": "grepout",
            "nameext": ".txt",
            "class": "File",
            "checksum": "sha1$a972f6d93fec7529fd4af8344ca298eea43dfbc5",
            "size": 16,
            "http://commonwl.org/cwltool#generation": 0
        }
    ],
    "out": {
        "location": "file:///tmp/nibotqbd/grepout.txt",
        "basename": "grepout.txt",
        "nameroot": "grepout",
        "nameext": ".txt",
        "class": "File",
        "checksum": "sha1$a972f6d93fec7529fd4af8344ca298eea43dfbc5",
        "size": 16,
        "http://commonwl.org/cwltool#generation": 0
    }
}
DEBUG [step 1_grep] produced output {
    "file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl/grep-and-count.cwl#1_grep/out": {
        "location": "file:///tmp/nibotqbd/grepout.txt",
        "basename": "grepout.txt",
        "nameroot": "grepout",
        "nameext": ".txt",
        "class": "File",
        "checksum": "sha1$a972f6d93fec7529fd4af8344ca298eea43dfbc5",
        "size": 16,
        "http://commonwl.org/cwltool#generation": 0
    }
}
INFO [step 1_grep] completed success
DEBUG [job 1_grep] Removing input staging directory /tmp/vslvo1xy
DEBUG [job 1_grep] Removing temporary directory /tmp/9zi_9lei
INFO [workflow ] starting step 2_wc
DEBUG [step 2_wc] job input {
    "file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl/grep-and-count.cwl#2_wc/grep_out": {
        "location": "file:///tmp/nibotqbd/grepout.txt",
        "basename": "grepout.txt",
        "nameroot": "grepout",
        "nameext": ".txt",
        "class": "File",
        "checksum": "sha1$a972f6d93fec7529fd4af8344ca298eea43dfbc5",
        "size": 16,
        "http://commonwl.org/cwltool#generation": 0
    }
}
DEBUG [step 2_wc] evaluated job input to {
    "file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl/grep-and-count.cwl#2_wc/grep_out": {
        "location": "file:///tmp/nibotqbd/grepout.txt",
        "basename": "grepout.txt",
        "nameroot": "grepout",
        "nameext": ".txt",
        "class": "File",
        "checksum": "sha1$a972f6d93fec7529fd4af8344ca298eea43dfbc5",
        "size": 16,
        "http://commonwl.org/cwltool#generation": 0
    }
}
INFO [step 2_wc] start
DEBUG [job 2_wc] initializing from file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl/wc_zatsu_v2.cwl as part of step 2_wc
DEBUG [job 2_wc] {
    "grep_out": {
        "location": "file:///tmp/nibotqbd/grepout.txt",
        "basename": "grepout.txt",
        "nameroot": "grepout",
        "nameext": ".txt",
        "class": "File",
        "checksum": "sha1$a972f6d93fec7529fd4af8344ca298eea43dfbc5",
        "size": 16,
        "http://commonwl.org/cwltool#generation": 0
    }
}
DEBUG [job 2_wc] path mappings is {
    "file:///tmp/nibotqbd/grepout.txt": [
        "/tmp/nibotqbd/grepout.txt",
        "/tmp/7ta0h195/stg8df6c6ed-1426-45e1-aeb6-c558696f0da9/grepout.txt",
        "File",
        true
    ]
}
DEBUG [job 2_wc] command line bindings is [
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
        "valueFrom": "$(inputs.grep_out)"
    }
]
DEBUG [job 2_wc] initial work dir {}
INFO [job 2_wc] /tmp/pbcmrdt1$ wc \
    -l \
    /tmp/7ta0h195/stg8df6c6ed-1426-45e1-aeb6-c558696f0da9/grepout.txt > /tmp/pbcmrdt1/wcout.txt
DEBUG Could not collect memory usage, job ended before monitoring began.
INFO [job 2_wc] completed success
DEBUG [job 2_wc] outputs {
    "all-for-debugging": [
        {
            "location": "file:///tmp/pbcmrdt1/wcout.txt",
            "basename": "wcout.txt",
            "nameroot": "wcout",
            "nameext": ".txt",
            "class": "File",
            "checksum": "sha1$2665d67534a7e977992ceb101d661bec4cbb1003",
            "size": 68,
            "http://commonwl.org/cwltool#generation": 0
        }
    ],
    "out": {
        "location": "file:///tmp/pbcmrdt1/wcout.txt",
        "basename": "wcout.txt",
        "nameroot": "wcout",
        "nameext": ".txt",
        "class": "File",
        "checksum": "sha1$2665d67534a7e977992ceb101d661bec4cbb1003",
        "size": 68,
        "http://commonwl.org/cwltool#generation": 0
    }
}
DEBUG [step 2_wc] produced output {
    "file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl/grep-and-count.cwl#2_wc/out": {
        "location": "file:///tmp/pbcmrdt1/wcout.txt",
        "basename": "wcout.txt",
        "nameroot": "wcout",
        "nameext": ".txt",
        "class": "File",
        "checksum": "sha1$2665d67534a7e977992ceb101d661bec4cbb1003",
        "size": 68,
        "http://commonwl.org/cwltool#generation": 0
    }
}
INFO [step 2_wc] completed success
INFO [workflow ] completed success
DEBUG [workflow ] outputs {
    "counts": {
        "location": "file:///tmp/pbcmrdt1/wcout.txt",
        "basename": "wcout.txt",
        "nameroot": "wcout",
        "nameext": ".txt",
        "class": "File",
        "checksum": "sha1$2665d67534a7e977992ceb101d661bec4cbb1003",
        "size": 68,
        "http://commonwl.org/cwltool#generation": 0
    },
    "grep_result": {
        "location": "file:///tmp/nibotqbd/grepout.txt",
        "basename": "grepout.txt",
        "nameroot": "grepout",
        "nameext": ".txt",
        "class": "File",
        "checksum": "sha1$a972f6d93fec7529fd4af8344ca298eea43dfbc5",
        "size": 16,
        "http://commonwl.org/cwltool#generation": 0
    }
}
DEBUG [job 2_wc] Removing input staging directory /tmp/7ta0h195
DEBUG [job 2_wc] Removing temporary directory /tmp/gtd5ycfx
DEBUG Moving /tmp/pbcmrdt1/wcout.txt to /workspaces/togotv_cwl_for_remote_container/zatsu_cwl/workflow_result/wcout.txt
DEBUG Moving /tmp/nibotqbd/grepout.txt to /workspaces/togotv_cwl_for_remote_container/zatsu_cwl/workflow_result/grepout.txt
DEBUG Removing intermediate output directory /tmp/pbcmrdt1
DEBUG Removing intermediate output directory /tmp/dv5paaif
DEBUG Removing intermediate output directory /tmp/nibotqbd
{
    "counts": {
        "location": "file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl/workflow_result/wcout.txt",
        "basename": "wcout.txt",
        "class": "File",
        "checksum": "sha1$2665d67534a7e977992ceb101d661bec4cbb1003",
        "size": 68,
        "path": "/workspaces/togotv_cwl_for_remote_container/zatsu_cwl/workflow_result/wcout.txt"
    },
    "grep_result": {
        "location": "file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl/workflow_result/grepout.txt",
        "basename": "grepout.txt",
        "class": "File",
        "checksum": "sha1$a972f6d93fec7529fd4af8344ca298eea43dfbc5",
        "size": 16,
        "path": "/workspaces/togotv_cwl_for_remote_container/zatsu_cwl/workflow_result/grepout.txt"
    }
}INFO Final process status is success
```
:::

無事新しく作成した`workflow_result`ディレクトリに`wcout.txt`と`grepout.txt`が出力されています!

https://github.com/yonesora56/togotv_cwl_for_remote_container/blob/master/zatsu_cwl/workflow_result/wcout.txt

このように､複数の処理を一つのコマンドで実行することが可能です｡どんなパラメータで実行したか､どの順番で実行したかなどを忘れても､このようにCWLを使うことで､複数の処理を一つのコマンドで実行することができます｡

&nbsp;

# 終わりに

この記事では､もう一つのCWLファイルと､2つのプロセスを実行できるワークフローのCWLファイルを書きました｡しかしながら､実際の解析の場面では更に多くのインプットするファイルや､解析する段階があると思います｡
次の記事では､この部分に焦点を当て､実際のバイオインフォマティクスの解析で使用するようなツールのCWLファイルを書いていきたいと思います｡

https://zenn.dev/sorayone/articles/cwl-document_3

&nbsp;

__今回の記事で使用したCWLのファイルをおいているリポジトリは以下からアクセスすることができます｡__
https://github.com/yonesora56/togotv_cwl_for_remote_container