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

&nbsp;

# grepコマンドとwcコマンドをCWL化する


初めに(1)の記事で作ったgrepコマンドのcwlファイルに加え､wcコマンドを使ったワークフローをCWLによって記述する例を紹介します｡

実行するのは､`grep one mock.txt > grep_out.txt` と `wc -l grep_out.txt > wc_out.txt` の2つです｡ 
この2つは､日本語ドキュメント(CWL Start Guide JP)でも説明されている例になります｡

```bash:example
grep one mock.txt > grep_out.txt
wc -l grep_out.txt > wc_out.txt
```

#### 参考：[CWL Start Guide JP: CWL で書いてみる: コマンドラインツール](https://github.com/pitagora-network/pitagora-cwl/wiki/CWL-Start-Guide-JP#cwl-%E3%81%A7%E6%9B%B8%E3%81%84%E3%81%A6%E3%81%BF%E3%82%8B-%E3%82%B3%E3%83%9E%E3%83%B3%E3%83%89%E3%83%A9%E3%82%A4%E3%83%B3%E3%83%84%E3%83%BC%E3%83%AB)

&nbsp;

前回は環境構築と､zatsu-cwl-genratorを使ってCWLファイルの生成と実行を行いました｡
この記事では､更に手を動かす作業を行っていきます｡主役となるコマンドは ｢grep｣ と｢wc｣の2つです｡ 
今回記述する流れとしては以下の2ステップです｡
__Step1：コマンドの処理に関するcwlファイルを書く(今回は2つ)__
__Step2：ワークフロー全体を記述するcwlファイルを書く__

&nbsp;

## zatsu-cwl-generatorを使ってcwlファイルを書く(Step1)

それでは実際に書いていきましょう｡ 
まず､前回書いていたgrepの処理に関するCWLファイルを以下に示します｡

https://github.com/yonezawa-sora/togotv_cwl_for_remote_container/blob/master/zatsu_generator/grep_zatsu.cwl

このファイルは､zatsu-cwl-generatorを使って生成したものを､少し修正しています｡
再度実行すると以下のようになります｡

```bash:
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

参考：[user_guide 2.16 best-practices](https://www.commonwl.org/user_guide/topics/best-practices.html)

&nbsp;

同様に､wcコマンドを使用したCWLファイルをzatsu-cwl-generatorで出力してみます｡

```bash=
zatsu-cwl-generator 'wc -l grep_out.txt > wc_out.txt' > wc_zatsu.cwl
```

出力された結果が以下になります｡

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
stdout: wc_out.txt
```

&nbsp;

先程と同様に､まず`--validate` オプションを使って評価してみます｡

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

警告は出ましたが､基本的な部分は大丈夫なようです｡
少し修正して､以下のように書いてみました｡

```bash:
#!/usr/bin/env cwl-runner
# Generated from: wc -l grep_out.txt > wc_out.txt
class: CommandLineTool
cwlVersion: v1.0
baseCommand: wc
arguments:
  - -l
  - $(inputs.grep_file)
inputs:
  - id: grep_file #わかりやすいように名前を変更
    type: File
    default:
      class: File #locationフィールドを一旦削除
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

もう一度`--validate`オプションで確認してみます｡

```bash:
cwltool --validate wc_zatsu.cwl
INFO /usr/local/bin/cwltool 3.1.20231016170136
INFO Resolved 'wc_zatsu.cwl' to 'file:///workspaces/togotv_shooting/zatsu_generator/wc_zatsu.cwl'
wc_zatsu.cwl is valid CWL.
```

&nbsp;

大丈夫なようなので､次に実際に実行してみます｡

```bash:
cwltool wc_zatsu.cwl
INFO /usr/local/bin/cwltool 3.1.20231016170136
INFO Resolved 'wc_zatsu.cwl' to 'file:///workspaces/togotv_shooting/zatsu_generator/wc_zatsu.cwl'
ERROR Input object failed validation:
Anonymous file object must have 'contents' and 'basename' fields.
```

エラーが発生しました｡どうやら`input`フィールドの部分でうまくいっていないようです｡
`contents` と `basename` フィールドを追加してみましょう｡

```yaml:
#!/usr/bin/env cwl-runner
# Generated from: wc -l grep_out.txt > wc_out.txt
class: CommandLineTool
cwlVersion: v1.0
baseCommand: wc
arguments:
  - -l
  - $(inputs.grep_file)
inputs:
  - id: grep_file
    type: File
    default:
      class: File
      basename: "grepout.txt"
      contents: "This is a text file."
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

再度実行して結果を見てみた所､カウントが本来は4なのですが､0と表示されていました｡
そこで､pathを明示すれば良いのかなと考え､以下のように`path`フィールドを追加して再修正しました｡

```yaml:
#!/usr/bin/env cwl-runner
# Generated from: wc -l grep_out.txt > wc_out.txt
class: CommandLineTool
cwlVersion: v1.0
baseCommand: wc
arguments:
  - -l
  - $(inputs.grep_file)
inputs:
  - id: grep_file
    type: File
    default:
      class: File
      basename: "grepout.txt"
      contents: "This is a text file."
      path: ./grepout.txt # pathフィールドを追加
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

結果を見てみると､4としっかりカウントされていました｡

```text:
4 /tmp/o8n3tzc8/stg195a993d-1cac-4ba9-bfd2-4266ebad333f/grepout.txt
```

このように､エラーメッセージや結果を見るなどして適切なファイルに逐次修正することが大事です｡

&nbsp;

## ワークフローの記述を行う(Step2)

それでは､この2つのステップを実際にワークフローとして記述していきます｡
`grep-and-count.cwl`というファイルを作成していきます｡ 

以下では､これまでとは異なる部分を中心に解説します｡

`class`フィールド：ここにはこれまでと違い､`Workflow`と記述します｡

```yaml:
#!/usr/bin/env cwl-runner
cwlVersion: v1.0
class: Workflow
```
&nbsp;

続いて`inputs`, `outputs`, そしてワークフローの手順を記述する`steps` について注目します｡ 

`inputs`フィールド：ここでは､ __｢ワークフロー｣__ として必要な入力のパラメータについて記述します｡

```yaml=
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

### 実際に実行してみる

それでは実際にこのワークフローを実行してみましょう｡ 
ワークフローの実行には今回､cwl-runnerを使用して実行します｡

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

このように処理され､最終的にwc\_out.txtが出力されます｡