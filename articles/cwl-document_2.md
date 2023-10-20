---
Title: CWLの作成環境をVScodeの｢Dev Containers｣の機能を使って構築する (2)
Author: Sora Yonezawa
---

&nbsp;

:::success
今回の記事で使用したCWLのファイルをおいているリポジトリは以下からアクセスすることができます｡
__[togotv_cwl_for_remote_container](https://github.com/yonezawa-sora/togotv_cwl_for_remote_container)__
:::

:::danger
この赤いフィールドは後ほど修正する点をメモしています
:::

&nbsp;

# はじめに

:::info
__本記事の対象となる方__

■ 環境構築ができたのでもっとCWLをガシガシ書いていきたい人
■ CWLの文法を実際に書いてみて学びたい人

(1)の記事を見ていただいたあと､更にCWLを勉強したいという方向けの記事になります｡
:::

以前の記事では､環境構築やzatsu-cwl-generatorの使い方をご紹介しました｡
この記事では､以下の2点をご紹介します｡
__(1) zatsu-cwl-generatorで生成されたcwlファイルを修正ししながらワークフローを実際に記述する__
__(2) バイオインフォマティクスの解析に関わるコマンドでワークフローを記述する__

CWLの文法も少しずつ紹介しながら､様々なエラーを解消しつつ､__ワークフロー__ を作っていきます｡

&nbsp;

# 【Case1】grepコマンドとwcコマンドをCWL化する


初めに(1)の記事で作ったgrepコマンドのcwlファイルに加え､wcコマンドを使ったワークフローをCWLによって記述する例を紹介します｡

実行するのは､`grep one mock.txt > grep_out.txt` と `wc -l grep_out.txt > wc_out.txt` の2つです｡ 
この2つは､日本語ドキュメント(CWL Start Guide JP)でも説明されている例になります｡

```bash=
grep one mock.txt > grep_out.txt
wc -l grep_out.txt > wc_out.txt
```

#### 参考：[CWL Start Guide JP: CWL で書いてみる: コマンドラインツール](https://github.com/pitagora-network/pitagora-cwl/wiki/CWL-Start-Guide-JP#cwl-%E3%81%A7%E6%9B%B8%E3%81%84%E3%81%A6%E3%81%BF%E3%82%8B-%E3%82%B3%E3%83%9E%E3%83%B3%E3%83%89%E3%83%A9%E3%82%A4%E3%83%B3%E3%83%84%E3%83%BC%E3%83%AB)

&nbsp;

次に､実際にCWLファイルの記述を行っていきます｡今回主役となるコマンドは ｢grep｣ と｢wc｣の2つです｡ 今回記述する流れとしては以下の2ステップです｡
__Step1：コマンドの処理に関するcwlファイルを書く(今回は2つ)__
__Step2：ワークフロー全体を記述するcwlファイルを書く__

&nbsp;

CWLファイルは記述する内容を YAMLかJSON の形式で記述し、｢.cwl ｣という拡張子でファイルに保存します。実行時にこの CWL ファイルを実行エンジンに入力すると、ワークフローが実行される､という流れになっています｡まずはじめにスクリプトの最初の処理である `grep one mock.txt > grepout.txt` の処理をCWLファイルとして記述していきます｡

:::danger

__追記：https://view.commonwl.org/workflows?search=__
:::

&nbsp;

## zatsu-cwl-generatorを使ってcwlファイルを書く(Step1)

それでは実際に書いていきましょう｡ 今回は､コマンドラインツールであるzatsu-cwl-genratorを使ってファイルを __出力__ してみます｡

参考：[雑に始めるCWL！をもっと雑に始めたい](https://qiita.com/tm_tn/items/2c789c5b3c28e3eb3c9a)

&nbsp;


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
上記のようにこのままでのファイルでも実際に実行することができます｡
しかし､この出力ファイルには以下の部分に赤線が示されます｡

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

このようなケースでは､自分がどのように実行したいのか(とりあえず動かしてみたい､zatsu-cwl-genratorをテンプレートにしっかり書きたい､など)を定めておくことで､修正が不要かどうかなどが変わってきますので注意が必要です｡

zatsu-cwl-generatorやエラーメッセージを自分の意図に合わせてうまく活用していきましょう!

:::

&nbsp;

### wcコマンドのCWLファイルを書く

同様に､wcコマンドを使用したCWLファイルをzatsu-cwl-generatorで出力してみます｡

```bash=
zatsu-cwl-generator 'wc -l grep_out.txt > wc_out.txt' > wc_zatsu.cwl
```

出力された結果が以下になります｡

```yaml=
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

```bash=
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

```bash=
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

```bash=
cwltool --validate wc_zatsu.cwl
INFO /usr/local/bin/cwltool 3.1.20231016170136
INFO Resolved 'wc_zatsu.cwl' to 'file:///workspaces/togotv_shooting/zatsu_generator/wc_zatsu.cwl'
wc_zatsu.cwl is valid CWL.
```

&nbsp;

大丈夫なようなので､次に実際に実行してみます｡

```bash=
cwltool wc_zatsu.cwl
INFO /usr/local/bin/cwltool 3.1.20231016170136
INFO Resolved 'wc_zatsu.cwl' to 'file:///workspaces/togotv_shooting/zatsu_generator/wc_zatsu.cwl'
ERROR Input object failed validation:
Anonymous file object must have 'contents' and 'basename' fields.
```

エラーが発生しました｡どうやら`input`フィールドの部分でうまくいっていないようです｡
`contents` と `basename` フィールドを追加してみましょう｡

```yaml=
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

```yaml=
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

```text=
4 /tmp/o8n3tzc8/stg195a993d-1cac-4ba9-bfd2-4266ebad333f/grepout.txt
```

このように､エラーメッセージや結果を見るなどして適切なファイルに逐次修正することが大事です｡

&nbsp;

## ワークフローの記述を行う(Step2)

それでは､この2つのステップを実際にワークフローとして記述していきます｡
`grep-and-count.cwl`というファイルを作成していきます｡ 

以下では､これまでとは異なる部分を中心に解説します｡

`class`フィールド：ここにはこれまでと違い､`Workflow`と記述します｡

```yaml=
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

```yaml=
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

```yaml=
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

```bash=
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

&nbsp;
  
&nbsp;

# 【Case2】バイオインフォマティクスの解析をワークフロー化する

次に､ 具体的に生命科学で使用されているツールを使った例をワークフローとして記述してみる例を紹介します｡

今回は､

**(1) BLASTpコマンドによる配列類似性検索**
**(2)awkによるヒットしたIDの抽出**
**(3)blastdbcmdによるヒットしたIDのfastaファイルの抽出**
**(4) ClustalOmegaによるマルチプルアラインメント**
**(5)fasttreeによるブートストラップ値の算出**

の5つのステップをワークフローとして記述してみます｡

今回取り上げたツールをうち､awk以外はBiocontainersによってdocker imageが構築されています(awkはUbuntu:23.10をイメージとして使用しています)｡
docker hubのホームページ にアクセスし､コマンドをコピー&ペーストすることでこれらのimageをpullすることができます｡


:::danger
__修正1：まずdockerなしの状態で記述した後､dockerについて記述する__
__修正2：`arguments`ではなく､`hints`フィールドを書く(条件がゆるい)__
__修正3：dockerの説明のあとにsingularityの説明を加える__
:::

&nbsp;

## docker imageを取得する

まず､それぞれのdocker imageをdocker hubからダウンロードします｡ tag を指定する形で今回はダウンロード(docker pull ... をコピー& ペースト)しました｡

以下はターミナルで `docker image ls` を行った例です｡

```bash=
docker image ls
REPOSITORY               TAG                 IMAGE ID       CREATED       SIZE
ubuntu                   23.10               78938b354ee7   2 weeks ago   72.1MB
biocontainers/fasttree   v2.1.10-2-deb_cv1   bdfde5d6026c   4 years ago   118MB
biocontainers/clustalo   v1.2.4-2-deb_cv1    d14a5e27e301   4 years ago   118MB
biocontainers/blast      v2.2.31_cv2         5b25e08b9871   4 years ago   2.03GB
```

&nbsp;

## ワークフローの準備を行う

次に､インプットするファイルを準備していきます｡

### 1\. クエリ配列

blastpを実行するので､今回クエリとするのは､ ウシのミオスタチンのタンパク質配列のfastaファイル (MSTN.fastaに名前を変更しています)です｡ curlコマンドで取得しました｡

```bash=
curl -O https://rest.uniprot.org/uniprotkb/O18836.fasta
```


### 2\. インデックス(データベース)

次に､BLASTのデータベースとして使用するUniProtのタンパク質配列のfastaファイル(uniprot\_sprot.fasta.gz)をftpサイトよりcurlコマンドで取得しました｡ 

```bash=
curl -O https://ftp.uniprot.org/pub/databases/uniprot/current_release/knowledgebase/complete/ uniprot_sprot.fasta.gz  
``` 

このファイルをインデックスにするために､ `makeblastdb` コマンドを実行します｡ なお､この部分の処理は､ワークフローに組み込まない形で記述しました｡`makeblastdb` コマンド の実行は以下のように行いました｡

```bash=
docker run --rm -it -v `pwd`:`pwd` -w `pwd` biocontainers/blast:v2.2.31_cv2 makeblastdb -in uniprot_sprot.fasta -dbtype prot -hash_index -parse_seqids 
```

&nbsp;

## 実際にCWLファイルを書いてみる (CommandLineTool編)

次に､実際の処理について､CWLファイルを記述していきましょう｡ 今回実行する5つのステップのコマンドは以下のようになっています｡

```bash=
# Step1
docker run --rm -it -v `pwd`:`pwd` -w `pwd` biocontainers/blast:v2.2.31_cv2 blastp -query MSTN.fasta -db uniprot_sprot.fasta -evalue 1e-5 -num_threads 4 -outfmt 6 -out blastp_result_MSTN.txt -max_target_seqs 20 

# Step2
docker run --rm -it -v `pwd`:`pwd` -w `pwd` awk '{ print $2 }' blastp_result_MSTN2.txt > blastp_result_id.txt

#Step3
docker run --rm -it -v `pwd`:`pwd`  -w  `pwd` biocontainers/blast:v2.2.31_cv2 blastdbcmd -db uniprot_sprot.fasta -entry_batch blastp_result_id.txt  -out blastp_results_MSTN.fasta

#Step4
docker run --rm -it -v `pwd`:`pwd` -w `pwd` biocontainers/clustalo:v1.2.4-2-deb_cv1 clustalo -i `pwd`/blastp_results_MSTN.fasta --outfmt=fasta -o MSTN_aligned_sequence.fasta

#Step5
docker run --rm -it -v `pwd`:`pwd` -w `pwd` biocontainers/fasttree:v2.1.10-2-deb_cv1 FastTree -out MSTN_tree.newick `pwd`/MSTN_aligned_sequence.fasta
```

&nbsp;

### zatsu-cwl-generatorを使おう

これらを手作業で書くという方法に加えて､すでにこの環境で使えるようになっている ｢zatsu-cwl-generator｣ を使って､アウトラインをある程度出力してもらう､という方法があります｡今回はこの方法を試してみます｡

zatsu-cwl-generator に コマンドを""で囲んで入力します｡

:::danger
__変更：`-c`オプションを使ってコンテナの場合を紹介する__
__変更：出力された結果を__
:::

```bash=
zatsu-cwl-generator "docker run --rm -it -v `pwd`:`pwd` -w `pwd` biocontainers/blast:v2.2.31_cv2 blastp -query MSTN.fasta -db uniprot_sprot.fasta -evalue 1e-5 -num_threads 4 -outfmt 6 -out blastp_result_MSTN.txt -max_target_seqs 20" > blastp.cwl
```

  

出力されたファイルの中身を見てみましょう｡ ただし､これが完成版ではなく､実際に確認して修正することが必要です｡

```yaml=
#!/usr/bin/env cwl-runner
# Generated from: docker run --rm -it -v `pwd`:`pwd` -w `pwd` biocontainers/blast:v2.2.31_cv2 blastp -query MSTN.fasta -db uniprot_sprot.fasta -evalue 1e-5 -num_threads 4 -outfmt 6 -out blastp_result_MSTN.txt -max_target_seqs 20
class: CommandLineTool
cwlVersion: v1.0
baseCommand: docker
arguments:
  - $(inputs.run)
  - --rm
  - -it
  - -v
  - $(inputs.v)
  - -w
  - $(inputs.w)
  - $(inputs.blast:v2_2_31_cv2)
  - $(inputs.blastp)
  - -query
  - $(inputs.query)
  - -db
  - $(inputs.db)
  - -evalue
  - $(inputs.evalue)
  - -num_threads
  - $(inputs.num_threads)
  - -outfmt
  - $(inputs.outfmt)
  - -out
  - $(inputs.out)
  - -max_target_seqs
  - $(inputs.max_target_seqs)
inputs:
  - id: run
    type: Any
    default: run
  - id: v
    type: Any
    default: `pwd`:`pwd`
  - id: w
    type: Any
    default: `pwd`
  - id: blast:v2_2_31_cv2
    type: File
    default:
      class: File
      location: biocontainers/blast:v2.2.31_cv2
  - id: blastp
    type: Any
    default: blastp
  - id: query
    type: File
    default:
      class: File
      location: MSTN.fasta
  - id: db
    type: File
    default:
      class: File
      location: uniprot_sprot.fasta
  - id: evalue
    type: Any
    default: 1e-5
  - id: num_threads
    type: int
    default: 4
  - id: outfmt
    type: int
    default: 6
  - id: out
    type: File
    default:
      class: File
      location: blastp_result_MSTN.txt
  - id: max_target_seqs
    type: int
    default: 20
outputs:
  - id: all-for-debugging
    type:
      type: array
      items: [File, Directory]
    outputBinding:
      glob: "*"
```

パラメータなどの記述が標準出力に出力されます｡ しかしながら､修正すべき点がありますので修正していきましょう｡

&nbsp;

### dockerを使う時は...
まず､大きな点として､baseCommand の修正があります｡
ここで実行するコマンドですが､ここにはdocker ではなく､blastpと書き直します｡
これは､`docker run` ... からはじまる部分の処理は全てCWLで処理してくれます｡
なお､docker(のイメージを使うことを明示する)は "requirements" の部分で記述します｡
  

```yaml=
#!/usr/bin/env cwl-runner
# Generated from: docker run --rm -it -v `pwd`:`pwd` -w `pwd` biocontainers/blast:v2.2.31_cv2 blastp -query MSTN.fasta -db uniprot_sprot.fasta -evalue 1e-5 -num_threads 4 -outfmt 6 -out blastp_result_MSTN.txt -max_target_seqs 20

class: CommandLineTool
cwlVersion: v1.0
baseCommand: [blastp] 

requirements:
  DockerRequirement: 
    dockerPull: "biocontainers/blast:v2.2.31_cv2"  #ここにイメージを書く
```
&nbsp; 

### 参考：5.5 DockerRequirement(Common Workflow Language (CWL) Command Line Tool Description, v1.0.2より)

[![Dockerrequirement](https://hackmd.io/_uploads/r16IJoZxT.png)](https://www.commonwl.org/v1.0/CommandLineTool.html#DockerRequirement)

&nbsp;

:::danger
__追記：shebangを書くかどうか__(結論：どちらでも大丈夫)

__追記：zatsu-cwl-generatorをもっと使ってスクリプトを出力__


:::

&nbsp;

次にCase1でも書いていた`inputs` `outputs` セクションについて記述します｡ `arguments`の部分は今回は記述しない方法で書いていきます｡ 実際に書いてみたのが以下のスクリプトになります｡

```yaml=
#!/usr/bin/env cwl-runner
# Generated from: docker run --rm -it -v `pwd`:`pwd` -w `pwd` biocontainers/blast:v2.2.31_cv2 blastp -query MSTN.fasta -db uniprot_sprot.fasta -evalue 1e-5 -num_threads 4 -outfmt 6 -out blastp_result_MSTN.txt -max_target_seqs 20
class: CommandLineTool
cwlVersion: v1.0
baseCommand: [blastp]
doc: BLASTP step 

requirements:
  DockerRequirement: #Dockerimageを記述
    dockerPull: "biocontainers/blast:v2.2.31_cv2"

# input
inputs:
  protein_query:
    type: File
    inputBinding:
      prefix: "-query"
      position: 1
  protein_database:
    type: File
    default: 
      class: File
      path: ../data/uniprot_sprot.fasta
    inputBinding:
      prefix: "-db"
      position: 2
    secondaryFiles:
      - ^.fasta.phd
      - ^.fasta.phi
      - ^.fasta.phr
      - ^.fasta.pin
      - ^.fasta.pog
      - ^.fasta.psd
      - ^.fasta.psi
      - ^.fasta.psq

  e-value:
    type: float?
    default: 1e-5
    inputBinding:
      prefix: "-evalue"
      position: 3
  number_of_threads:
    type: int
    default: 4
    inputBinding:
      prefix: "-num_threads"
      position: 4
  outformat_type:
    type: int
    default: 6
    inputBinding:
      prefix: "-outfmt"
      position: 5
  output_file_name:
    type: string
    inputBinding:
      prefix: "-out"
      position: 6
  max_target_sequence:
    type: int
    default: 20
    inputBinding:
      prefix: "-max_target_seqs"
      position: 7

#outputs
outputs:
  blastp_output_file:
    type: File
    outputBinding:
      glob: "$(inputs.output_file_name)"

# stdout
stdout: $(inputs.output_file_name)
```
&nbsp;

:::success
### BLASTのワークフローの書き方

BLASTでは､inputsのインデックス(引数は `-db` )の指定の部分の記述方法についていくつか方法があるようです｡

例えば今回は `secondaryFiles` フィールドを活用し､以下のように記述しています｡ 

```yaml=
  protein_database:
    type: File
    default: 
      class: File
      path: ../data/uniprot_sprot.fasta #pathを記述
    inputBinding:
      prefix: "-db"
      position: 2
    secondaryFiles:
      - ^.fasta.phd
      - ^.fasta.phi
      - ^.fasta.phr
      - ^.fasta.pin
      - ^.fasta.pog
      - ^.fasta.psd
      - ^.fasta.psi
      - ^.fasta.psq
```
ファイルとして指定しておき､パスについても明記しています｡また､`secandaryFiles`で､インデックス参照時にprotein_databaseに指定しているファイルと同じディレクトリになくてはいけないファイルを指定しています｡

#### 参考1：[cwl-samples-2023/debuginfo.cwl](https://github.com/manabuishii/cwl-samples-2023/blob/main/debuginfo.cwl#L10-L11)
#### 参考2：[5.1 CommandInputParameter](https://www.commonwl.org/v1.0/CommandLineTool.html#CommandInputParameter)
:::
  
&nbsp;

### パラメータをYAMLファイルでまとめて与える

パラメータを一つのYAMLファイルにまとめて実行することができます｡
ファイルは以下のように書くことが可能です｡

```yaml=
protein_query: 
  class: File
  path: ../data/MSTN.fasta
protein_database: 
  class: File
  path: ../data/uniprot_sprot.fasta
e-value: 1e-5
number_of_threads: 4
outformat_type: 6
output_file_name: "blastp_result.txt"
max_target_sequence: 20

```
このファイルを使って実行してみましょう｡

&nbsp;

### 1_blastp.cwlの実行

```bash=
cwltool --outdir ./data ./Tools/1_blastp.cwl ./Tools/1_blastp.yml
```
今回は`--outdir`オプションを使って､出力先のディレクトリを指定して実行してみます｡

&nbsp;

```bash=
INFO /usr/local/bin/cwltool 3.1.20230906142556
INFO Resolved './Tools/1_blastp.cwl' to 'file:///workspaces/togotv_shooting/Tools/1_blastp.cwl'
INFO [job 1_blastp.cwl] /tmp/qe5n7kgg$ docker \
    run \
    -i \
    --mount=type=bind,source=/tmp/qe5n7kgg,target=/DcjUoi \
    --mount=type=bind,source=/tmp/p__s1km1,target=/tmp \
    --mount=type=bind,source=/workspaces/togotv_shooting/data/uniprot_sprot.fasta,target=/var/lib/cwl/stgd1ff326d-ac51-4352-b27b-9100
606072ca/uniprot_sprot.fasta,readonly \                                                                                                  --mount=type=bind,source=/workspaces/togotv_shooting/data/uniprot_sprot.fasta.phd,target=/var/lib/cwl/stgd1ff326d-ac51-4352-b27b-
9100606072ca/uniprot_sprot.fasta.phd,readonly \                                                                                          --mount=type=bind,source=/workspaces/togotv_shooting/data/uniprot_sprot.fasta.phi,target=/var/lib/cwl/stgd1ff326d-ac51-4352-b27b-
9100606072ca/uniprot_sprot.fasta.phi,readonly \                                                                                          --mount=type=bind,source=/workspaces/togotv_shooting/data/uniprot_sprot.fasta.phr,target=/var/lib/cwl/stgd1ff326d-ac51-4352-b27b-
9100606072ca/uniprot_sprot.fasta.phr,readonly \                                                                                          --mount=type=bind,source=/workspaces/togotv_shooting/data/uniprot_sprot.fasta.pin,target=/var/lib/cwl/stgd1ff326d-ac51-4352-b27b-
9100606072ca/uniprot_sprot.fasta.pin,readonly \                                                                                          --mount=type=bind,source=/workspaces/togotv_shooting/data/uniprot_sprot.fasta.pog,target=/var/lib/cwl/stgd1ff326d-ac51-4352-b27b-
9100606072ca/uniprot_sprot.fasta.pog,readonly \                                                                                          --mount=type=bind,source=/workspaces/togotv_shooting/data/uniprot_sprot.fasta.psd,target=/var/lib/cwl/stgd1ff326d-ac51-4352-b27b-
9100606072ca/uniprot_sprot.fasta.psd,readonly \                                                                                          --mount=type=bind,source=/workspaces/togotv_shooting/data/uniprot_sprot.fasta.psi,target=/var/lib/cwl/stgd1ff326d-ac51-4352-b27b-
9100606072ca/uniprot_sprot.fasta.psi,readonly \                                                                                          --mount=type=bind,source=/workspaces/togotv_shooting/data/uniprot_sprot.fasta.psq,target=/var/lib/cwl/stgd1ff326d-ac51-4352-b27b-
9100606072ca/uniprot_sprot.fasta.psq,readonly \                                                                                          --mount=type=bind,source=/workspaces/togotv_shooting/data/MSTN.fasta,target=/var/lib/cwl/stge2ab10d6-20a1-4698-88fe-dc0997863178/
MSTN.fasta,readonly \                                                                                                                    --workdir=/DcjUoi \
    --read-only=true \
    --user=1000:1000 \
    --rm \
    --cidfile=/tmp/d3yca0_g/20230928025733-664451.cid \
    --env=TMPDIR=/tmp \
    --env=HOME=/DcjUoi \
    biocontainers/blast:v2.2.31_cv2 \
    blastp \
    -query \
    /var/lib/cwl/stge2ab10d6-20a1-4698-88fe-dc0997863178/MSTN.fasta \
    -db \
    /var/lib/cwl/stgd1ff326d-ac51-4352-b27b-9100606072ca/uniprot_sprot.fasta \
    -evalue \
    0.00001 \
    -num_threads \
    4 \
    -outfmt \
    6 \
    -out \
    blastp_result.txt \
    -max_target_seqs \
    20
INFO [job 1_blastp.cwl] Max memory used: 11MiB
INFO [job 1_blastp.cwl] completed success
{
    "blastp_output_file": {
        "location": "file:///workspaces/togotv_shooting/data/blastp_result.txt",
        "basename": "blastp_result.txt",
        "class": "File",
        "checksum": "sha1$31464e2d0bdfd65f89e0542688844d3686bf278c",
        "size": 1536,
        "path": "/workspaces/togotv_shooting/data/blastp_result.txt"
    }
}INFO Final process status is success

```
実行が成功したようです｡blastpの結果も出力されました｡
同様に他のCWLファイルも､zatsu-cwl-generatorで一旦出力後､修正する形で書いてみましょう｡その結果を以下に示します｡ 

&nbsp;

## CWLファイル(CommandLineTool)記述例

### 2_awk.cwl

```yaml=
#!/usr/bin/env cwl-runner
# Generated from: docker run --rm -it -v `pwd`:`pwd` -w `pwd` ubuntu:23.10 awk '{ print $2 }' blastp_result_MSTN.txt > blastp_result_id.txt
class: CommandLineTool
cwlVersion: v1.0
baseCommand: [awk]

requirements:
  DockerRequirement:
    dockerPull: "ubuntu:23.10"

# 固定の引数
arguments:
  - valueFrom: '{ print $2 }' 
    position: 1

inputs:
  blastp_result:
    type: File
    inputBinding:
      position: 2
  output_id_file_name:
    type: string
    inputBinding:
      position: 3
outputs:
  awk_output:
    type: File
    outputBinding:
      glob: "$(inputs.output_id_file_name)"
stdout: $(inputs.output_id_file_name)
```

&nbsp;

### 3\_blastdbcmd.cwl

```yaml=
#!/usr/bin/env cwl-runner
# Generated from: docker run --rm -it -v /workspaces/togotv_shooting/data:/workspaces/togotv_shooting/data  -w  /workspaces/togotv_shooting/data biocontainers/blast:v2.2.31_cv2 blastdbcmd -db uniprot_sprot.fasta -entry_batch blastp_result_id.txt  -out blastp_results_MSTN.fasta
class: CommandLineTool
cwlVersion: v1.0
baseCommand: [blastdbcmd]

requirements:
  DockerRequirement:
    dockerPull: "biocontainers/blast:v2.2.31_cv2"

# Inputs
inputs:
  blastdbcmd_protein_database:
    type: File
    default: 
      class: File
      path: ../data/uniprot_sprot.fasta
    inputBinding:
      prefix: "-db"
      position: 1
    secondaryFiles:
      - ^.fasta.phd
      - ^.fasta.phi
      - ^.fasta.phr
      - ^.fasta.pin
      - ^.fasta.pog
      - ^.fasta.psd
      - ^.fasta.psi
      - ^.fasta.psq
  id_query:
    type: File
    inputBinding:
      prefix: "-entry_batch"
      position: 2
  blastdbcmd_output_name:
    type: string
    inputBinding:
      prefix: "-out"
      position: 3
#Outputs
outputs:
  blastdbcmd_output:
    type: File
    outputBinding:
      glob: "$(inputs.blastdbcmd_output_name)"
```

&nbsp;

### 4_clustalo.cwl

```yaml=
#!/usr/bin/env cwl-runner
# Generated from: docker run --rm -it -v /workspaces/togotv_shooting/data:/workspaces/togotv_shooting/data -w /workspaces/togotv_shooting/data biocontainers/clustalo:v1.2.4-2-deb_cv1 clustalo -i /workspaces/togotv_shooting/data/blastp_results_MSTN.fasta --outfmt=fasta -o MSTN_aligned_sequence.fasta
class: CommandLineTool
cwlVersion: v1.0
baseCommand: [clustalo]

requirements:
  DockerRequirement:
    dockerPull: "biocontainers/clustalo:v1.2.4-2-deb_cv1"

# Inputs
inputs:
  clustalo_inputs:
    type: File
    inputBinding:
      prefix: "-i"
      position: 1
  clustalo_format:
    type: string
    default: "fasta"
    inputBinding:
      prefix: "--outfmt"
      position: 2
  clustalo_output_name:
    type: string
    inputBinding:
      prefix: "-o"
      position: 3

outputs:
  clustalo_output:
    type: File
    outputBinding:
      glob: "$(inputs.clustalo_output_name)"
```

&nbsp; 

### 5_fasttree.cwl

```yaml=
#!/usr/bin/env cwl-runner
# Generated from: docker run --rm -it -v /workspaces/togotv_shooting/data:/workspaces/togotv_shooting/data -w /workspaces/togotv_shooting/data biocontainers/fasttree:v2.1.10-2-deb_cv1 FastTree -out MSTN_tree.newick /workspaces/togotv_shooting/data/MSTN_aligned_sequence.fasta
class: CommandLineTool
cwlVersion: v1.0
baseCommand: [fasttree]

requirements:
  DockerRequirement:
    dockerPull: "biocontainers/fasttree:v2.1.10-2-deb_cv1"

#Inputs
inputs:
  fasttree_output_file_name:
    type: string
    inputBinding:
      prefix: "-out"
      position: 1
  fasttree_input_file:
    type: File
    inputBinding:
      position: 2
# Outputs
outputs:
  fasttree_output_file:
    type: File
    outputBinding:
      glob: "$(inputs.fasttree_output_file_name)" 
```

&nbsp;

### ワークフローを記述する

:::danger
__修正：best practicesに載っているような書き方で最後書く__
:::

これまで､CommanlinetoolのCWLファイルを書いてみました｡次にこれらの5つのステップを実行するワークフローを記述していきます｡この例では､ワークフロー全体に関するパラメータを｢1\_protein\_query｣のように数字をつけています｡ 前の処理のアウトプットを受け取る部分は｢blastp\_result: step1\_blastp/blastp\_output\_file｣のように記述し､それ以外の全ての処理に関するパラメータをinputsで記述しています｡
```yaml=
#!/usr/bin/env cwl-runner
cwlVersion: v1.0
class: Workflow
doc: "5つのステップを実行するワークフロー: blastp, awk, blastdbcmd, clustalo, and fasttree"
#最初のインプットのみ記述しようと思ったが､全部書いてわかりやすく書く方針に
# 前のステップの出力の書き方に注意
inputs: 
  #blastp
  1_protein_query: 
    type: File
  2_protein_database:
    type: File
    secondaryFiles:
      - ^.fasta.phd
      - ^.fasta.phi
      - ^.fasta.phr
      - ^.fasta.pin
      - ^.fasta.pog
      - ^.fasta.psd
      - ^.fasta.psi
      - ^.fasta.psq
  3_e-value: 
    type: float?
  4_number_of_threads: 
    type: int
  5_outformat_type: 
    type: int
  6_output_file_name: 
    type: string
  7_max_target_sequence: 
    type: int
  #awk
  8_output_id_file_name:
    type: string
  #blastdbcmd
  9_blastdbcmd_protein_database:
    type: File
    secondaryFiles:
      - ^.fasta.phd
      - ^.fasta.phi
      - ^.fasta.phr
      - ^.fasta.pin
      - ^.fasta.pog
      - ^.fasta.psd
      - ^.fasta.psi
      - ^.fasta.psq
  10_blastdbcmd_output_name:
    type: string
  #clustalo
  11_clustalo_format:
    type: string
  12_clustalo_output_name:
    type: string
  #fasttree
  13_fasttree_output_file_name:
    type: string
  

outputs:
  final_output_bootstrap: 
    type: File
    outputSource: step5_fasttree/fasttree_output_file
steps:
  step1_blastp:
    run: ../Tools/1_blastp.cwl 
    in: 
      protein_query: 1_protein_query
      protein_database: 2_protein_database
      e-value: 3_e-value
      number_of_threads: 4_number_of_threads
      outformat_type: 5_outformat_type
      output_file_name: 6_output_file_name
      max_target_sequence: 7_max_target_sequence
    out: [blastp_output_file] 

  step2_awk:
    run: ../Tools/2_awk.cwl
    in:
      blastp_result: step1_blastp/blastp_output_file 
      output_id_file_name: 8_output_id_file_name
    out: [awk_output]

  step3_blastdbcmd:
    run: ../Tools/3_blastdbcmd.cwl
    in: 
      blastdbcmd_protein_database: 9_blastdbcmd_protein_database
      id_query: step2_awk/awk_output 
      blastdbcmd_output_name: 10_blastdbcmd_output_name
    out: [blastdbcmd_output]

  step4_clustalo:
    run: ../Tools/4_clustalo.cwl
    in:
      clustalo_inputs: step3_blastdbcmd/blastdbcmd_output
      clustalo_format: 11_clustalo_format
      clustalo_output_name: 12_clustalo_output_name
    out: [clustalo_output]

  step5_fasttree:
    run: ../Tools/5_fasttree.cwl
    in:
      fasttree_output_file_name: 13_fasttree_output_file_name
      fasttree_input_file: step4_clustalo/clustalo_output
    out: [fasttree_output_file]
```

最後に全てのパラメータ(1\_protein\_queryのように書いている部分)を記述したファイル を作成します｡すでに述べたように､パラメータはYAMLファイルとしてまとめて記述することが可能です｡

以下のような記述になりました｡

&nbsp;

### パラメータファイル(YAMLファイル)

```yaml=
# 1_blastp.cwl のパラメータ
1_protein_query:
  class: File
  path: ../data/MSTN.fasta
2_protein_database:
  class: File
  path: ../data/uniprot_sprot.fasta
3_e-value: 1e-5
4_number_of_threads: 4
5_outformat_type: 6
6_output_file_name: "blastp_result.txt"
7_max_target_sequence: 20

# 2_awk.cwl のパラメータ
8_output_id_file_name: "blastp_result_id.txt"

# 3_blastdbcmd.cwl のパラメータ
9_blastdbcmd_protein_database:
  class: File
  path: "../data/uniprot_sprot.fasta"
10_blastdbcmd_output_name: "blastdbcmd_output.fasta"

# 4_clustalo.cwl のパラメータ
11_clustalo_format: "fasta"
12_clustalo_output_name: "clustalo_output.fasta"

# 5_fasttree.cwl のパラメータ
13_fasttree_output_file_name: "fasttree_output.newick"
```

&nbsp;

最後に`cwltool –-validate`を実行します｡
```yaml=
cwltool --validate ./workflow/6_phylogenetic-workflow.cwl ./workflow/input.yaml
INFO /usr/local/bin/cwltool 3.1.20230906142556
INFO Resolved './workflow/6_phylogenetic-workflow.cwl' to 'file:///workspaces/togotv_shooting/workflow/6_phylogenetic-workflow.cwl'
./workflow/6_phylogenetic-workflow.cwl is valid CWL.
```

書き方としては問題が無いようです｡
続いて､CWLファイルに対して`--help`オプションを行ってみます｡

```bash=
cwltool ./workflow/6_phylogenetic-workflow.cwl --help
```

:::danger
__追記：ここでも先頭何行か見せて､docフィールドを書くことのご利益が得られることを書く__
:::


それでは､実際に実行してみます｡

```yaml=
cwl-runner ./workflow/6_phylogenetic-workflow.cwl ./workflow/input.yml
```

実行結果
```bash=
INFO /usr/local/bin/cwl-runner 3.1.20230906142556
INFO Resolved './workflow/6_phylogenetic-workflow.cwl' to 'file:///workspaces/togotv_shooting/workflow/6_phylogenetic-workflow.cwl
'                                                                                                                                 INFO [workflow ] start
INFO [workflow ] starting step step1_blastp
INFO [step step1_blastp] start
INFO [job step1_blastp] /tmp/2s81aqwz$ docker \
    run \
    -i \
    --mount=type=bind,source=/tmp/2s81aqwz,target=/XKQdXE \
    --mount=type=bind,source=/tmp/omtpty6t,target=/tmp \
    --mount=type=bind,source=/workspaces/togotv_shooting/data/uniprot_sprot.fasta,target=/var/lib/cwl/stg9579ad21-d27b-4919-a2d2-f
7dc16036a89/uniprot_sprot.fasta,readonly \                                                                                            --mount=type=bind,source=/workspaces/togotv_shooting/data/uniprot_sprot.fasta.phd,target=/var/lib/cwl/stg9579ad21-d27b-4919-a2
d2-f7dc16036a89/uniprot_sprot.fasta.phd,readonly \                                                                                    --mount=type=bind,source=/workspaces/togotv_shooting/data/uniprot_sprot.fasta.phi,target=/var/lib/cwl/stg9579ad21-d27b-4919-a2
d2-f7dc16036a89/uniprot_sprot.fasta.phi,readonly \                                                                                    --mount=type=bind,source=/workspaces/togotv_shooting/data/uniprot_sprot.fasta.phr,target=/var/lib/cwl/stg9579ad21-d27b-4919-a2
d2-f7dc16036a89/uniprot_sprot.fasta.phr,readonly \                                                                                    --mount=type=bind,source=/workspaces/togotv_shooting/data/uniprot_sprot.fasta.pin,target=/var/lib/cwl/stg9579ad21-d27b-4919-a2
d2-f7dc16036a89/uniprot_sprot.fasta.pin,readonly \                                                                                    --mount=type=bind,source=/workspaces/togotv_shooting/data/uniprot_sprot.fasta.pog,target=/var/lib/cwl/stg9579ad21-d27b-4919-a2
d2-f7dc16036a89/uniprot_sprot.fasta.pog,readonly \                                                                                    --mount=type=bind,source=/workspaces/togotv_shooting/data/uniprot_sprot.fasta.psd,target=/var/lib/cwl/stg9579ad21-d27b-4919-a2
d2-f7dc16036a89/uniprot_sprot.fasta.psd,readonly \                                                                                    --mount=type=bind,source=/workspaces/togotv_shooting/data/uniprot_sprot.fasta.psi,target=/var/lib/cwl/stg9579ad21-d27b-4919-a2
d2-f7dc16036a89/uniprot_sprot.fasta.psi,readonly \                                                                                    --mount=type=bind,source=/workspaces/togotv_shooting/data/uniprot_sprot.fasta.psq,target=/var/lib/cwl/stg9579ad21-d27b-4919-a2
d2-f7dc16036a89/uniprot_sprot.fasta.psq,readonly \                                                                                    --mount=type=bind,source=/workspaces/togotv_shooting/data/MSTN.fasta,target=/var/lib/cwl/stg074f1962-1693-4b69-8f5d-66b5ca45e6
76/MSTN.fasta,readonly \                                                                                                              --workdir=/XKQdXE \
    --read-only=true \
    --user=1000:1000 \
    --rm \
    --cidfile=/tmp/2fxzkh85/20230928045240-374338.cid \
    --env=TMPDIR=/tmp \
    --env=HOME=/XKQdXE \
    biocontainers/blast:v2.2.31_cv2 \
    blastp \
    -query \
    /var/lib/cwl/stg074f1962-1693-4b69-8f5d-66b5ca45e676/MSTN.fasta \
    -db \
    /var/lib/cwl/stg9579ad21-d27b-4919-a2d2-f7dc16036a89/uniprot_sprot.fasta \
    -evalue \
    0.00001 \
    -num_threads \
    4 \
    -outfmt \
    6 \
    -out \
    blastp_result.txt \
    -max_target_seqs \
    20
INFO [job step1_blastp] Max memory used: 6MiB
INFO [job step1_blastp] completed success
INFO [step step1_blastp] completed success
INFO [workflow ] starting step step2_awk
INFO [step step2_awk] start
INFO [job step2_awk] /tmp/dll4rpk0$ docker \
    run \
    -i \
    --mount=type=bind,source=/tmp/dll4rpk0,target=/XKQdXE \
    --mount=type=bind,source=/tmp/end82mmy,target=/tmp \
    --mount=type=bind,source=/tmp/2s81aqwz/blastp_result.txt,target=/var/lib/cwl/stg79db6e9b-c182-4436-8210-c6e4c82c3883/blastp_re
sult.txt,readonly \                                                                                                                   --workdir=/XKQdXE \
    --read-only=true \
    --log-driver=none \
    --user=1000:1000 \
    --rm \
    --cidfile=/tmp/hp2d02kz/20230928045242-157190.cid \
    --env=TMPDIR=/tmp \
    --env=HOME=/XKQdXE \
    ubuntu:23.10 \
    awk \
    '{ print $2 }' \
    /var/lib/cwl/stg79db6e9b-c182-4436-8210-c6e4c82c3883/blastp_result.txt \
    blastp_result_id.txt > /tmp/dll4rpk0/blastp_result_id.txt
INFO [job step2_awk] completed success
INFO [step step2_awk] completed success
INFO [workflow ] starting step step3_blastdbcmd
INFO [step step3_blastdbcmd] start
INFO [job step3_blastdbcmd] /tmp/qjene9aj$ docker \
    run \
    -i \
    --mount=type=bind,source=/tmp/qjene9aj,target=/XKQdXE \
    --mount=type=bind,source=/tmp/_adhvpty,target=/tmp \
    --mount=type=bind,source=/workspaces/togotv_shooting/data/uniprot_sprot.fasta,target=/var/lib/cwl/stgcc8a126f-0ce6-4f81-a4ba-3
1f50dcb2572/uniprot_sprot.fasta,readonly \                                                                                            --mount=type=bind,source=/workspaces/togotv_shooting/data/uniprot_sprot.fasta.phd,target=/var/lib/cwl/stgcc8a126f-0ce6-4f81-a4
ba-31f50dcb2572/uniprot_sprot.fasta.phd,readonly \                                                                                    --mount=type=bind,source=/workspaces/togotv_shooting/data/uniprot_sprot.fasta.phi,target=/var/lib/cwl/stgcc8a126f-0ce6-4f81-a4
ba-31f50dcb2572/uniprot_sprot.fasta.phi,readonly \                                                                                    --mount=type=bind,source=/workspaces/togotv_shooting/data/uniprot_sprot.fasta.phr,target=/var/lib/cwl/stgcc8a126f-0ce6-4f81-a4
ba-31f50dcb2572/uniprot_sprot.fasta.phr,readonly \                                                                                    --mount=type=bind,source=/workspaces/togotv_shooting/data/uniprot_sprot.fasta.pin,target=/var/lib/cwl/stgcc8a126f-0ce6-4f81-a4
ba-31f50dcb2572/uniprot_sprot.fasta.pin,readonly \                                                                                    --mount=type=bind,source=/workspaces/togotv_shooting/data/uniprot_sprot.fasta.pog,target=/var/lib/cwl/stgcc8a126f-0ce6-4f81-a4
ba-31f50dcb2572/uniprot_sprot.fasta.pog,readonly \                                                                                    --mount=type=bind,source=/workspaces/togotv_shooting/data/uniprot_sprot.fasta.psd,target=/var/lib/cwl/stgcc8a126f-0ce6-4f81-a4
ba-31f50dcb2572/uniprot_sprot.fasta.psd,readonly \                                                                                    --mount=type=bind,source=/workspaces/togotv_shooting/data/uniprot_sprot.fasta.psi,target=/var/lib/cwl/stgcc8a126f-0ce6-4f81-a4
ba-31f50dcb2572/uniprot_sprot.fasta.psi,readonly \                                                                                    --mount=type=bind,source=/workspaces/togotv_shooting/data/uniprot_sprot.fasta.psq,target=/var/lib/cwl/stgcc8a126f-0ce6-4f81-a4
ba-31f50dcb2572/uniprot_sprot.fasta.psq,readonly \                                                                                    --mount=type=bind,source=/tmp/dll4rpk0/blastp_result_id.txt,target=/var/lib/cwl/stg0b724761-074f-4d10-aaf8-8109d1c3da00/blastp
_result_id.txt,readonly \                                                                                                             --workdir=/XKQdXE \
    --read-only=true \
    --user=1000:1000 \
    --rm \
    --cidfile=/tmp/8kcrikj8/20230928045243-166987.cid \
    --env=TMPDIR=/tmp \
    --env=HOME=/XKQdXE \
    biocontainers/blast:v2.2.31_cv2 \
    blastdbcmd \
    -db \
    /var/lib/cwl/stgcc8a126f-0ce6-4f81-a4ba-31f50dcb2572/uniprot_sprot.fasta \
    -entry_batch \
    /var/lib/cwl/stg0b724761-074f-4d10-aaf8-8109d1c3da00/blastp_result_id.txt \
    -out \
    blastdbcmd_output.fasta
INFO [job step3_blastdbcmd] completed success
INFO [step step3_blastdbcmd] completed success
INFO [workflow ] starting step step4_clustalo
INFO [step step4_clustalo] start
INFO [job step4_clustalo] /tmp/0ch20d7y$ docker \
    run \
    -i \
    --mount=type=bind,source=/tmp/0ch20d7y,target=/XKQdXE \
    --mount=type=bind,source=/tmp/kj4ogriu,target=/tmp \
    --mount=type=bind,source=/tmp/qjene9aj/blastdbcmd_output.fasta,target=/var/lib/cwl/stgedb08f31-1999-4c21-9ea9-87f61c72e296/bla
stdbcmd_output.fasta,readonly \                                                                                                       --workdir=/XKQdXE \
    --read-only=true \
    --user=1000:1000 \
    --rm \
    --cidfile=/tmp/t3efhk2e/20230928045244-184118.cid \
    --env=TMPDIR=/tmp \
    --env=HOME=/XKQdXE \
    biocontainers/clustalo:v1.2.4-2-deb_cv1 \
    clustalo \
    -i \
    /var/lib/cwl/stgedb08f31-1999-4c21-9ea9-87f61c72e296/blastdbcmd_output.fasta \
    --outfmt \
    fasta \
    -o \
    clustalo_output.fasta
INFO [job step4_clustalo] completed success
INFO [step step4_clustalo] completed success
INFO [workflow ] starting step step5_fasttree
INFO [step step5_fasttree] start
INFO [job step5_fasttree] /tmp/8kehyeh7$ docker \
    run \
    -i \
    --mount=type=bind,source=/tmp/8kehyeh7,target=/XKQdXE \
    --mount=type=bind,source=/tmp/7u14_wey,target=/tmp \
    --mount=type=bind,source=/tmp/0ch20d7y/clustalo_output.fasta,target=/var/lib/cwl/stg358ccb55-655e-428a-b172-d4444da82abf/clust
alo_output.fasta,readonly \                                                                                                           --workdir=/XKQdXE \
    --read-only=true \
    --user=1000:1000 \
    --rm \
    --cidfile=/tmp/godhcg8g/20230928045245-203729.cid \
    --env=TMPDIR=/tmp \
    --env=HOME=/XKQdXE \
    biocontainers/fasttree:v2.1.10-2-deb_cv1 \
    fasttree \
    -out \
    fasttree_output.newick \
    /var/lib/cwl/stg358ccb55-655e-428a-b172-d4444da82abf/clustalo_output.fasta
FastTree Version 2.1.10 Double precision (No SSE3)
Alignment: /var/lib/cwl/stg358ccb55-655e-428a-b172-d4444da82abf/clustalo_output.fasta
Amino acid distances: BLOSUM45 Joins: balanced Support: SH-like 1000
Search: Normal +NNI +SPR (2 rounds range 10) +ML-NNI opt-each=1
TopHits: 1.00*sqrtN close=default refresh=0.80
ML Model: Jones-Taylor-Thorton, CAT approximation with 20 rate categories
Initial topology in 0.00 seconds
Refining topology: 15 rounds ME-NNIs, 2 rounds ME-SPRs, 7 rounds ML-NNIs
Total branch-length 0.205 after 0.01 sec
ML-NNI round 1: LogLk = -1664.746 NNIs 2 max delta 0.00 Time 0.12
      0.12 seconds: Site likelihoods with rate category 1 of 20
Switched to using 20 rate categories (CAT approximation)
Rate categories were divided by 0.677 so that average rate = 1.0
CAT-based log-likelihoods may not be comparable across runs
Use -gamma for approximate but comparable Gamma(20) log-likelihoods
ML-NNI round 2: LogLk = -1632.213 NNIs 0 max delta 0.00 Time 0.24
Turning off heuristics for final round of ML NNIs (converged)
      0.24 seconds: ML NNI round 3 of 7, 1 of 11 splits
ML-NNI round 3: LogLk = -1632.213 NNIs 0 max delta 0.00 Time 0.33 (final)
Optimize all lengths: LogLk = -1632.213 Time 0.35
Total time: 0.44 seconds Unique: 13/20 Bad splits: 0/10
INFO [job step5_fasttree] Max memory used: 0MiB
INFO [job step5_fasttree] completed success
INFO [step step5_fasttree] completed success
INFO [workflow ] completed success
{
    "final_output_bootstrap": {
        "location": "file:///workspaces/togotv_shooting/fasttree_output.newick",
        "basename": "fasttree_output.newick",
        "class": "File",
        "checksum": "sha1$2174533dd0b0da12a274ed2e0849f7c878c9376b",
        "size": 807,
        "path": "/workspaces/togotv_shooting/fasttree_output.newick"
    }
}INFO Final process status is success
```

解析が成功しているようです｡ 実際に､newick形式のファイルも出力されています｡ このように､何回もコマンドを実行せず､1つのコマンドで実行することができます｡
__CWLで記述したファイルは様々な環境で実行することができますので､ぜひご自身でクローンして確かめてください!__

&nbsp;

:::danger
# 追記：リンク集を作成する

1. 作ったワークフローのリポジトリ
:::