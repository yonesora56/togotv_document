---
title: "CWLの作成環境をVScodeの｢Dev Containers｣の機能を使って構築する (3)"
emoji: "📑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["CWL", "bioinformatics"]
published: true
---
__今回の記事で使用したCWLのファイルをおいているリポジトリは以下からアクセスすることができます｡__
https://github.com/yonesora56/togotv_cwl_for_remote_container

前回の記事はこちらです
https://zenn.dev/sorayone/articles/cwl-document_1

https://zenn.dev/sorayone/articles/cwl-document_2

:::message
__本記事の対象となる方__
■ 環境構築ができたので､次のステップに進みたい方
■ 実際の解析の場面でCWLを使ってみたい方

(1)､(2)の記事を見ていただいたあと､更にCWLを勉強したいという方向けの記事になります｡
:::

# はじめに

これまで､環境構築および`grep`と`wc`の処理に関するワークフローを作成する過程を紹介しました｡
この記事では､具体的にバイオインフォマティクス分野で使用されているツールの例をCWLとして記述し､ __ワークフロー__ として実行する部分までをご紹介します｡
このプロセスにおいては､これまでと同様にzatsu-cwl-generatorを軸としてCWLファイルを作成し､その後に細かい修正を加えていきながら作成していきます｡

## 実行するプロセスの概要

今回は以下に示す5つのステップをワークフローとして記述します｡

**(1)blastpコマンドによる配列類似性検索**
**(2)awkによるヒットしたIDの抽出**
**(3)blastdbcmdによるヒットしたIDのfastaファイルの抽出**
**(4)ClustalOmegaによるマルチプルアラインメント**
**(5)fasttreeによるnewick形式ファイルの出力**

これにより､BLASTpでヒットした配列から､系統樹を作成する前段階の処理(newick形式のファイルを出力)をワークフローとして記述します｡

:::message
事前準備で必要なファイルのダウンロードなどは以下のプロセスで行っています｡
:::details ファイルのダウンロードとインデックスの作成
### 1\. クエリ配列の取得

blastpを実行する際にクエリとするタンパク質配列データとして､ウシのミオスタチンのタンパク質配列のfastaファイル (MSTN.fastaに名前を変更しています)を使用します｡ 
curlコマンドでUniProtから取得しました｡

```bash:
curl -O https://rest.uniprot.org/uniprotkb/O18836.fasta
```

### 2\. uniprot fastaファイルの取得

次に､BLASTのデータベースとして､UniProtのタンパク質配列のfastaファイル(uniprot\_sprot.fasta.gz)をftpサイトよりcurlコマンドで取得し､展開します｡ 

```bash:
curl -O https://ftp.uniprot.org/pub/databases/uniprot/current_release/knowledgebase/complete/uniprot_sprot.fasta.gz
pigz -d -p 10 uniprot_sprot.fasta.gz
``` 

### 3\. インデックスの作成

このファイルをインデックスにするため､ `makeblastdb` コマンドをdockerを使って実行しました(既にインストールされている場合はdockerを使用しなくても大丈夫です)｡ 
`makeblastdb` コマンド の実行は以下のように行いました｡ 

```bash:
docker run --rm -it -v `pwd`:`pwd` -w `pwd` biocontainers/blast:v2.2.31_cv2 makeblastdb -in uniprot_sprot.fasta -dbtype prot -hash_index -parse_seqids 
```
:::

&nbsp;

CWLとして記述するコマンドは以下の通りです｡ 
このように､5つのステップをワークフローとして記述します｡

```bash:
# Step1
blastp -query MSTN.fasta -db uniprot_sprot.fasta -evalue 1e-5 -num_threads 4 -outfmt 6 -out blastp_result.txt -max_target_seqs 20 

# Step2
awk '{ print $2 }' blastp_result.txt > blastp_result_id.txt

#Step3
blastdbcmd -db uniprot_sprot.fasta -entry_batch blastp_result_id.txt  -out blastdbcmd_result.fasta

#Step4
clustalo -i blastdbcmd_result.fasta --outfmt=fasta -o clustalo_result.fasta

#Step5
FastTree -boot 100 -out MSTN_tree.newick clustalo_result.fasta
```

:::message
この記事でも後述していますが､__docker imageがある場合にはそのimageを使用して実行する__ ことができるCWLファイルを作成していくので､すでにツールをインストールしている場合でも試すことが可能です｡
:::

&nbsp;

# zatsu-cwl-generatorを使ってblastpを実行するCWLファイルを書く

次に､上記の処理について､CWLファイルを記述していきましょう｡ zatsu-cwl-generatorを使ってコードを生成し､修正しながら作成していきます｡
今回は､`zatsu_cwl_bioinformatics`ディレクトリで作業を行っています｡修正のプロセスはトグル内に示してあるので､ぜひご覧ください｡
以下では､最初のプロセス､blastpのコマンドを例にして(修正のプロセスも含めながら)説明していきます｡

## (1) zatsu-cwl-generatorを使ってblastpのCWLファイルを生成する

これまでと同様に､zatsu-cwl-generatorを使ってCWLファイルを生成します｡

```bash:
zatsu-cwl-generator 'blastp -query MSTN.fasta -db uniprot_sprot.fasta -evalue 1e-5 -num_threads 4 -outfmt 6 -out blastp_result.txt -max_target_seqs 20' > 1_blastp.cwl
```

https://github.com/yonesora56/togotv_cwl_for_remote_container/blob/master/zatsu_cwl_bioinformatics/1_blastp.cwl

`cwltool --validate`でチェックします｡
  
```bash:
cwltool --validate 1_blastp.cwl
INFO /usr/local/bin/cwltool 3.1.20240508115724
INFO Resolved '1_blastp.cwl' to 'file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/1_blastp.cwl'
1_blastp.cwl:45:7: Warning: Field 'location' contains undefined reference to
                   'file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/blastp_result.txt'
WARNING 1_blastp.cwl:45:7: Warning: Field 'location' contains undefined reference to
                   'file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/blastp_result.txt'
1_blastp.cwl is valid CWL.
```
`1_blastp.cwl is valid CWL.`という出力がされているものの､`1_blastp.cwl is valid CWL.`として実行例として使った入力ファイルが存在しないことを表す警告が出ていますが、修正方法は後ほど説明します｡

:::message
ここで一旦実行してみようと思いますが､例えばツールをローカルでダウンロードしておらず､docker imageを使って実行したいという状況があるかもしれません｡ 
例えば､この記事を作成している環境は､Mac M1 でDev containersの機能を使用しているため､環境としては __Linux ARM64__ という状況です｡ 現在､LinuxのARM64に対応したツールは少なく､この場合ではdockerで動かしたほうが簡単そうです｡
しかしながら､ __dockerを使って実行する場合には､どのようにCWLファイルを記述すればよいのでしょうか?__
これについて､次のセクションで説明していきます｡
:::

&nbsp;

## (2) コンテナオプション(`-c`)を使って出力する

CWLでは､docker imageを使って実行する場合､これまでのCWLファイルでは書いていなかった`hints`フィールド(あるいは`requirements`フィールド)を使って記述します｡

https://www.commonwl.org/user_guide/topics/using-containers.html#using-containers

__zatsu-cwl-generatorでは､`-c`, `--container` オプションを使うことで､コンテナを使用するコマンドを出力することができます｡__

https://qiita.com/tm_tn/items/2c789c5b3c28e3eb3c9a

&nbsp;

以下のようにコマンドを実行します｡

```bash
zatsu-cwl-generator 'blastp -query MSTN.fasta -db uniprot_sprot.fasta -evalue 1e-5 -num_threads 4 -outfmt 6 -out blastp_result.txt -max_target_seqs 20' --container biocontainers/blast:v2.2.31_cv2 > 1_blastp_docker.cwl
```
これまでと同じようにコマンドを''でくくったあとに､`-c` もしくは`--container` オプションをつけて､image(tagの情報も含めて)を記載します｡ すると､以下のようにCWLファイルが出力されます｡

https://github.com/yonesora56/togotv_cwl_for_remote_container/blob/master/zatsu_cwl_bioinformatics/1_blastp_docker.cwl

これまでと同じように出力されますが､コンテナオプションをつけると､下に`hints`フィールドが出力されています｡

https://github.com/yonesora56/togotv_cwl_for_remote_container/blob/master/zatsu_cwl_bioinformatics/1_blastp_docker.cwl#L56-L58

このように､`hints`フィールドを使ってdocker imageを指定することができます｡

https://www.commonwl.org/v1.0/CommandLineTool.html#DockerRequirement

`cwltool --validate`を使ってチェックします｡

```bash:
cwltool --validate 1_blastp_docker.cwl
INFO /usr/local/bin/cwltool 3.1.20240508115724
INFO Resolved '1_blastp_docker.cwl' to 'file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/1_blastp_docker.cwl'
1_blastp_docker.cwl:45:7: Warning: Field 'location' contains undefined reference to
                          'file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/blastp_result.txt'
WARNING 1_blastp_docker.cwl:45:7: Warning: Field 'location' contains undefined reference to
                          'file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/blastp_result.txt'
1_blastp_docker.cwl is valid CWL.
```
こちらについても､`1_blastp_docker.cwl is valid CWL.`という出力がされているものの､`1_blastp_docker.cwl is valid CWL.`として実行例として使った入力ファイルが存在しないことを表す警告が出ていますが､修正方法は後ほど説明します｡
一旦このファイルを使って実行してみましょう｡

```bash:
cwltool --debug 1_blastp_docker.cwl
```

これまでと異なり､エラーが発生しました｡

:::details エラー内容
```bash:
INFO /usr/local/bin/cwltool 3.1.20240508115724
INFO Resolved '1_blastp_docker.cwl' to 'file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/1_blastp_docker.cwl'
1_blastp_docker.cwl:45:7: Warning: Field 'location' contains undefined reference to
                          'file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/blastp_result.txt'
WARNING 1_blastp_docker.cwl:45:7: Warning: Field 'location' contains undefined reference to
                          'file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/blastp_result.txt'
DEBUG Can't make command line argument from Any
DEBUG Parsed job order from command line: {
    "__id": "1_blastp_docker.cwl",
    "query": {
        "class": "File",
        "location": "file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/MSTN.fasta"
    },
    "db": {
        "class": "File",
        "location": "file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/uniprot_sprot.fasta"
    },
    "num_threads": 8,
    "outfmt": 7,
    "out": {
        "class": "File",
        "location": "file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/blastp_result.txt"
    },
    "max_target_seqs": 20
}
DEBUG [job 1_blastp_docker.cwl] initializing from file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/1_blastp_docker.cwl
DEBUG [job 1_blastp_docker.cwl] {
    "query": {
        "class": "File",
        "location": "file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/MSTN.fasta",
        "size": 476,
        "basename": "MSTN.fasta",
        "nameroot": "MSTN",
        "nameext": ".fasta"
    },
    "db": {
        "class": "File",
        "location": "file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/uniprot_sprot.fasta",
        "size": 284812502,
        "basename": "uniprot_sprot.fasta",
        "nameroot": "uniprot_sprot",
        "nameext": ".fasta"
    },
    "num_threads": 8,
    "outfmt": 7,
    "out": {
        "class": "File",
        "location": "file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/blastp_result.txt",
        "basename": "blastp_result.txt",
        "nameroot": "blastp_result",
        "nameext": ".txt"
    },
    "max_target_seqs": 20,
    "evalue": 1e-05
}
ERROR Input object failed validation:
[Errno 2] No such file or directory: '/workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/blastp_result.txt'
Traceback (most recent call last):
  File "/usr/local/lib/python3.10/site-packages/cwltool/pathmapper.py", line 178, in visit
    st = os.lstat(deref)
FileNotFoundError: [Errno 2] No such file or directory: '/workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/blastp_result.txt'

During handling of the above exception, another exception occurred:

Traceback (most recent call last):
  File "/usr/local/lib/python3.10/site-packages/cwltool/main.py", line 1314, in main
    (out, status) = real_executor(
  File "/usr/local/lib/python3.10/site-packages/cwltool/executors.py", line 63, in __call__
    return self.execute(process, job_order_object, runtime_context, logger)
  File "/usr/local/lib/python3.10/site-packages/cwltool/executors.py", line 146, in execute
    self.run_jobs(process, job_order_object, logger, runtime_context)
  File "/usr/local/lib/python3.10/site-packages/cwltool/executors.py", line 221, in run_jobs
    for job in jobiter:
  File "/usr/local/lib/python3.10/site-packages/cwltool/command_line_tool.py", line 992, in job
    builder.pathmapper = self.make_path_mapper(reffiles, builder.stagedir, runtimeContext, True)
  File "/usr/local/lib/python3.10/site-packages/cwltool/command_line_tool.py", line 485, in make_path_mapper
    return PathMapper(reffiles, runtimeContext.basedir, stagedir, separateDirs)
  File "/usr/local/lib/python3.10/site-packages/cwltool/pathmapper.py", line 104, in __init__
    self.setup(dedup(referenced_files), basedir)
  File "/usr/local/lib/python3.10/site-packages/cwltool/pathmapper.py", line 207, in setup
    self.visit(
  File "/usr/local/lib/python3.10/site-packages/cwltool/pathmapper.py", line 167, in visit
    with SourceLine(
  File "schema_salad/sourceline.py", line 249, in __exit__
schema_salad.exceptions.ValidationException: [Errno 2] No such file or directory: '/workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinforma
tics/blastp_result.txt'                                                                                                                            
```
:::

出力ファイルが存在しないというエラー(`FileNotFoundError: [Errno 2] No such file or directory: '/workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/blastp_result.txt'`)が出ています｡
これは`cwltool --validate`を実行したときの警告と関係があるようです｡
それではこれについて修正してみましょう｡

&nbsp;

## (3) 修正プロセス 1

`inputs`フィールドに書いてある部分が修正が必要のようです｡
https://github.com/yonesora56/togotv_cwl_for_remote_container/blob/master/zatsu_cwl_bioinformatics/1_blastp_docker.cwl#L41-L45

ここの`blastp_result.txt`はファイルそのものではなく､__ファイル名__ ､つまり文字列になるはずなので､以下のように修正してみました｡

https://github.com/yonesora56/togotv_cwl_for_remote_container/blob/master/zatsu_cwl_bioinformatics/1_blastp_docker_v2.cwl#L41-L43

それでは再度実行してみましょう｡
:::details 修正後のCWLファイル
```bash:1_blastp_docker_v2.cwl
cwltool --debug 1_blastp_docker_v2.cwl
INFO /usr/local/bin/cwltool 3.1.20240508115724
INFO Resolved '1_blastp_docker_v2.cwl' to 'file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/1_blastp_docker_v2.cwl'
DEBUG Can't make command line argument from Any
DEBUG Parsed job order from command line: {
    "__id": "1_blastp_docker_v2.cwl",
    "query": {
        "class": "File",
        "location": "file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/MSTN.fasta"
    },
    "db": {
        "class": "File",
        "location": "file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/uniprot_sprot.fasta"
    },
    "num_threads": 8,
    "outfmt": 7,
    "out": "blastp_result.txt",
    "max_target_seqs": 20
}
DEBUG [job 1_blastp_docker_v2.cwl] initializing from file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/1_blastp_docker_v2
.cwl                                                                                                                                               DEBUG [job 1_blastp_docker_v2.cwl] {
    "query": {
        "class": "File",
        "location": "file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/MSTN.fasta",
        "size": 476,
        "basename": "MSTN.fasta",
        "nameroot": "MSTN",
        "nameext": ".fasta"
    },
    "db": {
        "class": "File",
        "location": "file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/uniprot_sprot.fasta",
        "size": 284812502,
        "basename": "uniprot_sprot.fasta",
        "nameroot": "uniprot_sprot",
        "nameext": ".fasta"
    },
    "num_threads": 8,
    "outfmt": 7,
    "out": "blastp_result.txt",
    "max_target_seqs": 20,
    "evalue": 1e-05
}
DEBUG [job 1_blastp_docker_v2.cwl] path mappings is {
    "file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/MSTN.fasta": [
        "/workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/MSTN.fasta",
        "/var/lib/cwl/stga7998393-9a81-4cc7-9591-8fe9236fcf5d/MSTN.fasta",
        "File",
        true
    ],
    "file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/uniprot_sprot.fasta": [
        "/workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/uniprot_sprot.fasta",
        "/var/lib/cwl/stgb5786c95-6694-4f59-9707-d79eced66e64/uniprot_sprot.fasta",
        "File",
        true
    ]
}
DEBUG [job 1_blastp_docker_v2.cwl] command line bindings is [
    {
        "position": [
            -1000000,
            0
        ],
        "datum": "blastp"
    },
    {
        "position": [
            0,
            0
        ],
        "datum": "-query"
    },
    {
        "position": [
            0,
            1
        ],
        "valueFrom": "$(inputs.query)"
    },
    {
        "position": [
            0,
            2
        ],
        "datum": "-db"
    },
    {
        "position": [
            0,
            3
        ],
        "valueFrom": "$(inputs.db)"
    },
    {
        "position": [
            0,
            4
        ],
        "datum": "-evalue"
    },
    {
        "position": [
            0,
            5
        ],
        "valueFrom": "$(inputs.evalue)"
    },
    {
        "position": [
            0,
            6
        ],
        "datum": "-num_threads"
    },
    {
        "position": [
            0,
            7
        ],
        "valueFrom": "$(inputs.num_threads)"
    },
    {
        "position": [
            0,
            8
        ],
        "datum": "-outfmt"
    },
    {
        "position": [
            0,
            9
        ],
        "valueFrom": "$(inputs.outfmt)"
    },
    {
        "position": [
            0,
            10
        ],
        "datum": "-out"
    },
    {
        "position": [
            0,
            11
        ],
        "valueFrom": "$(inputs.out)"
    },
    {
        "position": [
            0,
            12
        ],
        "datum": "-max_target_seqs"
    },
    {
        "position": [
            0,
            13
        ],
        "valueFrom": "$(inputs.max_target_seqs)"
    }
]
DEBUG [job 1_blastp_docker_v2.cwl] initial work dir {}
INFO [job 1_blastp_docker_v2.cwl] /tmp/17begc9b$ docker \
    run \
    -i \
    --mount=type=bind,source=/tmp/17begc9b,target=/jFvgAu \
    --mount=type=bind,source=/tmp/pvwugmew,target=/tmp \
    --mount=type=bind,source=/workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/MSTN.fasta,target=/var/lib/cwl/stga7998393-9a81-4
cc7-9591-8fe9236fcf5d/MSTN.fasta,readonly \                                                                                                            --mount=type=bind,source=/workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/uniprot_sprot.fasta,target=/var/lib/cwl/stgb5786c
95-6694-4f59-9707-d79eced66e64/uniprot_sprot.fasta,readonly \                                                                                          --workdir=/jFvgAu \
    --read-only=true \
    --user=1000:1000 \
    --rm \
    --cidfile=/tmp/nquyx9tk/20240527074301-528880.cid \
    --env=TMPDIR=/tmp \
    --env=HOME=/jFvgAu \
    biocontainers/blast:v2.2.31_cv2 \
    blastp \
    -query \
    /var/lib/cwl/stga7998393-9a81-4cc7-9591-8fe9236fcf5d/MSTN.fasta \
    -db \
    /var/lib/cwl/stgb5786c95-6694-4f59-9707-d79eced66e64/uniprot_sprot.fasta \
    -evalue \
    0.00001 \
    -num_threads \
    8 \
    -outfmt \
    7 \
    -out \
    blastp_result.txt \
    -max_target_seqs \
    20
WARNING: The requested image's platform (linux/amd64) does not match the detected host platform (linux/arm64/v8) and no specific platform was reque
sted                                                                                                                                               BLAST Database error: No alias or index file found for protein database [/var/lib/cwl/stgb5786c95-6694-4f59-9707-d79eced66e64/uniprot_sprot.fasta] 
in search path [/jFvgAu::]                                                                                                                         WARNING [job 1_blastp_docker_v2.cwl] exited with status: 2
WARNING [job 1_blastp_docker_v2.cwl] completed permanentFail
DEBUG [job 1_blastp_docker_v2.cwl] outputs {
    "all-for-debugging": [
        {
            "location": "file:///tmp/17begc9b/blastp_result.txt",
            "basename": "blastp_result.txt",
            "nameroot": "blastp_result",
            "nameext": ".txt",
            "class": "File",
            "checksum": "sha1$da39a3ee5e6b4b0d3255bfef95601890afd80709",
            "size": 0,
            "http://commonwl.org/cwltool#generation": 0
        }
    ]
}
DEBUG [job 1_blastp_docker_v2.cwl] Removing input staging directory /tmp/e_qr0ldv
DEBUG [job 1_blastp_docker_v2.cwl] Removing temporary directory /tmp/pvwugmew
DEBUG Moving /tmp/17begc9b/blastp_result.txt to /workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/blastp_result.txt
DEBUG Removing intermediate output directory /tmp/17begc9b
{
    "all-for-debugging": [
        {
            "location": "file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/blastp_result.txt",
            "basename": "blastp_result.txt",
            "class": "File",
            "checksum": "sha1$da39a3ee5e6b4b0d3255bfef95601890afd80709",
            "size": 0,
            "path": "/workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/blastp_result.txt"
        }
    ]
}WARNING Final process status is permanentFail
```
:::
今回はファイルの出力はされたものの､空のファイルが出力されてしまいました｡
エラーを見てみると､`BLAST Database error: No alias or index file found for protein database [/var/lib/cwl/stgb5786c95-6694-4f59-9707-d79eced66e64/uniprot_sprot.fasta]`などが見られ､データベースが見つからないというエラーが出ています｡
これはBLASTなどに見られるエラーで､ __インデックス作成時に複数のインデックスファイルも含めて記述する必要があります｡__
それでは､これを修正してみましょう｡

&nbsp;

## (4) 修正プロセス 2 複数のインデックスファイルの記載

`inputs`フィールドの部分を修正してみます｡

https://github.com/yonesora56/togotv_cwl_for_remote_container/blob/master/zatsu_cwl_bioinformatics/1_blastp_docker_v2.cwl#L27-L31

BLASTでは､inputsのインデックス(引数は `-db` )の指定の部分の記述方法について下記のような方法があるようです｡

https://qiita.com/Yohei__K/items/b947655eb59a8853c172

この記事では別のアプローチとして､ `secondaryFiles` フィールドを活用し､以下のように記述してみます｡

https://github.com/yonesora56/togotv_cwl_for_remote_container/blob/master/zatsu_cwl_bioinformatics/1_blastp_docker_v3.cwl#L27-L40

このように記述することで､複数のインデックスファイルを指定することができます｡

https://www.commonwl.org/v1.0/CommandLineTool.html#CommandInputParameter

それでは再度実行してみましょう｡

:::details 修正後のcwlファイルでの実行結果
```bash:1_blastp_docker_v3.cwl
cwltool --debug 1_blastp_docker_v3.cwl
INFO /usr/local/bin/cwltool 3.1.20240508115724
INFO Resolved '1_blastp_docker_v3.cwl' to 'file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/1_blastp_docker_v3.cwl'
DEBUG Can't make command line argument from Any
DEBUG Parsed job order from command line: {
    "__id": "1_blastp_docker_v3.cwl",
    "query": {
        "class": "File",
        "location": "file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/MSTN.fasta"
    },
    "db": {
        "class": "File",
        "location": "file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/uniprot_sprot.fasta"
    },
    "num_threads": 8,
    "outfmt": 7,
    "out": "blastp_result.txt",
    "max_target_seqs": 20
}
DEBUG [job 1_blastp_docker_v3.cwl] initializing from file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/1_blastp_docker_v3
.cwl                                                                                                                                               DEBUG [job 1_blastp_docker_v3.cwl] {
    "query": {
        "class": "File",
        "location": "file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/MSTN.fasta",
        "size": 476,
        "basename": "MSTN.fasta",
        "nameroot": "MSTN",
        "nameext": ".fasta"
    },
    "db": {
        "class": "File",
        "location": "file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/uniprot_sprot.fasta",
        "size": 284812502,
        "basename": "uniprot_sprot.fasta",
        "nameroot": "uniprot_sprot",
        "nameext": ".fasta",
        "secondaryFiles": [
            {
                "location": "file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/uniprot_sprot.fasta.phd",
                "basename": "uniprot_sprot.fasta.phd",
                "class": "File",
                "nameroot": "uniprot_sprot.fasta",
                "nameext": ".phd"
            },
            {
                "location": "file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/uniprot_sprot.fasta.phi",
                "basename": "uniprot_sprot.fasta.phi",
                "class": "File",
                "nameroot": "uniprot_sprot.fasta",
                "nameext": ".phi"
            },
            {
                "location": "file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/uniprot_sprot.fasta.phr",
                "basename": "uniprot_sprot.fasta.phr",
                "class": "File",
                "nameroot": "uniprot_sprot.fasta",
                "nameext": ".phr"
            },
            {
                "location": "file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/uniprot_sprot.fasta.pin",
                "basename": "uniprot_sprot.fasta.pin",
                "class": "File",
                "nameroot": "uniprot_sprot.fasta",
                "nameext": ".pin"
            },
            {
                "location": "file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/uniprot_sprot.fasta.pog",
                "basename": "uniprot_sprot.fasta.pog",
                "class": "File",
                "nameroot": "uniprot_sprot.fasta",
                "nameext": ".pog"
            },
            {
                "location": "file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/uniprot_sprot.fasta.psd",
                "basename": "uniprot_sprot.fasta.psd",
                "class": "File",
                "nameroot": "uniprot_sprot.fasta",
                "nameext": ".psd"
            },
            {
                "location": "file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/uniprot_sprot.fasta.psi",
                "basename": "uniprot_sprot.fasta.psi",
                "class": "File",
                "nameroot": "uniprot_sprot.fasta",
                "nameext": ".psi"
            },
            {
                "location": "file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/uniprot_sprot.fasta.psq",
                "basename": "uniprot_sprot.fasta.psq",
                "class": "File",
                "nameroot": "uniprot_sprot.fasta",
                "nameext": ".psq"
            }
        ]
    },
    "num_threads": 8,
    "outfmt": 7,
    "out": "blastp_result.txt",
    "max_target_seqs": 20,
    "evalue": 1e-05
}
DEBUG [job 1_blastp_docker_v3.cwl] path mappings is {
    "file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/MSTN.fasta": [
        "/workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/MSTN.fasta",
        "/var/lib/cwl/stgcb307528-6f4f-4251-9517-0121cb068d7e/MSTN.fasta",
        "File",
        true
    ],
    "file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/uniprot_sprot.fasta": [
        "/workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/uniprot_sprot.fasta",
        "/var/lib/cwl/stg90d0f4dc-8137-4082-8c5d-4eaf670f8786/uniprot_sprot.fasta",
        "File",
        true
    ],
    "file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/uniprot_sprot.fasta.phd": [
        "/workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/uniprot_sprot.fasta.phd",
        "/var/lib/cwl/stg90d0f4dc-8137-4082-8c5d-4eaf670f8786/uniprot_sprot.fasta.phd",
        "File",
        true
    ],
    "file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/uniprot_sprot.fasta.phi": [
        "/workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/uniprot_sprot.fasta.phi",
        "/var/lib/cwl/stg90d0f4dc-8137-4082-8c5d-4eaf670f8786/uniprot_sprot.fasta.phi",
        "File",
        true
    ],
    "file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/uniprot_sprot.fasta.phr": [
        "/workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/uniprot_sprot.fasta.phr",
        "/var/lib/cwl/stg90d0f4dc-8137-4082-8c5d-4eaf670f8786/uniprot_sprot.fasta.phr",
        "File",
        true
    ],
    "file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/uniprot_sprot.fasta.pin": [
        "/workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/uniprot_sprot.fasta.pin",
        "/var/lib/cwl/stg90d0f4dc-8137-4082-8c5d-4eaf670f8786/uniprot_sprot.fasta.pin",
        "File",
        true
    ],
    "file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/uniprot_sprot.fasta.pog": [
        "/workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/uniprot_sprot.fasta.pog",
        "/var/lib/cwl/stg90d0f4dc-8137-4082-8c5d-4eaf670f8786/uniprot_sprot.fasta.pog",
        "File",
        true
    ],
    "file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/uniprot_sprot.fasta.psd": [
        "/workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/uniprot_sprot.fasta.psd",
        "/var/lib/cwl/stg90d0f4dc-8137-4082-8c5d-4eaf670f8786/uniprot_sprot.fasta.psd",
        "File",
        true
    ],
    "file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/uniprot_sprot.fasta.psi": [
        "/workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/uniprot_sprot.fasta.psi",
        "/var/lib/cwl/stg90d0f4dc-8137-4082-8c5d-4eaf670f8786/uniprot_sprot.fasta.psi",
        "File",
        true
    ],
    "file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/uniprot_sprot.fasta.psq": [
        "/workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/uniprot_sprot.fasta.psq",
        "/var/lib/cwl/stg90d0f4dc-8137-4082-8c5d-4eaf670f8786/uniprot_sprot.fasta.psq",
        "File",
        true
    ]
}
DEBUG [job 1_blastp_docker_v3.cwl] command line bindings is [
    {
        "position": [
            -1000000,
            0
        ],
        "datum": "blastp"
    },
    {
        "position": [
            0,
            0
        ],
        "datum": "-query"
    },
    {
        "position": [
            0,
            1
        ],
        "valueFrom": "$(inputs.query)"
    },
    {
        "position": [
            0,
            2
        ],
        "datum": "-db"
    },
    {
        "position": [
            0,
            3
        ],
        "valueFrom": "$(inputs.db)"
    },
    {
        "position": [
            0,
            4
        ],
        "datum": "-evalue"
    },
    {
        "position": [
            0,
            5
        ],
        "valueFrom": "$(inputs.evalue)"
    },
    {
        "position": [
            0,
            6
        ],
        "datum": "-num_threads"
    },
    {
        "position": [
            0,
            7
        ],
        "valueFrom": "$(inputs.num_threads)"
    },
    {
        "position": [
            0,
            8
        ],
        "datum": "-outfmt"
    },
    {
        "position": [
            0,
            9
        ],
        "valueFrom": "$(inputs.outfmt)"
    },
    {
        "position": [
            0,
            10
        ],
        "datum": "-out"
    },
    {
        "position": [
            0,
            11
        ],
        "valueFrom": "$(inputs.out)"
    },
    {
        "position": [
            0,
            12
        ],
        "datum": "-max_target_seqs"
    },
    {
        "position": [
            0,
            13
        ],
        "valueFrom": "$(inputs.max_target_seqs)"
    }
]
DEBUG [job 1_blastp_docker_v3.cwl] initial work dir {}
INFO [job 1_blastp_docker_v3.cwl] /tmp/8hge4mo3$ docker \
    run \
    -i \
    --mount=type=bind,source=/tmp/8hge4mo3,target=/RrTBZF \
    --mount=type=bind,source=/tmp/7sot2rlv,target=/tmp \
    --mount=type=bind,source=/workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/MSTN.fasta,target=/var/lib/cwl/stgcb307528-6f4f-4
251-9517-0121cb068d7e/MSTN.fasta,readonly \                                                                                                            --mount=type=bind,source=/workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/uniprot_sprot.fasta,target=/var/lib/cwl/stg90d0f4
dc-8137-4082-8c5d-4eaf670f8786/uniprot_sprot.fasta,readonly \                                                                                          --mount=type=bind,source=/workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/uniprot_sprot.fasta.phd,target=/var/lib/cwl/stg90
d0f4dc-8137-4082-8c5d-4eaf670f8786/uniprot_sprot.fasta.phd,readonly \                                                                                  --mount=type=bind,source=/workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/uniprot_sprot.fasta.phi,target=/var/lib/cwl/stg90
d0f4dc-8137-4082-8c5d-4eaf670f8786/uniprot_sprot.fasta.phi,readonly \                                                                                  --mount=type=bind,source=/workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/uniprot_sprot.fasta.phr,target=/var/lib/cwl/stg90
d0f4dc-8137-4082-8c5d-4eaf670f8786/uniprot_sprot.fasta.phr,readonly \                                                                                  --mount=type=bind,source=/workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/uniprot_sprot.fasta.pin,target=/var/lib/cwl/stg90
d0f4dc-8137-4082-8c5d-4eaf670f8786/uniprot_sprot.fasta.pin,readonly \                                                                                  --mount=type=bind,source=/workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/uniprot_sprot.fasta.pog,target=/var/lib/cwl/stg90
d0f4dc-8137-4082-8c5d-4eaf670f8786/uniprot_sprot.fasta.pog,readonly \                                                                                  --mount=type=bind,source=/workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/uniprot_sprot.fasta.psd,target=/var/lib/cwl/stg90
d0f4dc-8137-4082-8c5d-4eaf670f8786/uniprot_sprot.fasta.psd,readonly \                                                                                  --mount=type=bind,source=/workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/uniprot_sprot.fasta.psi,target=/var/lib/cwl/stg90
d0f4dc-8137-4082-8c5d-4eaf670f8786/uniprot_sprot.fasta.psi,readonly \                                                                                  --mount=type=bind,source=/workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/uniprot_sprot.fasta.psq,target=/var/lib/cwl/stg90
d0f4dc-8137-4082-8c5d-4eaf670f8786/uniprot_sprot.fasta.psq,readonly \                                                                                  --workdir=/RrTBZF \
    --read-only=true \
    --user=1000:1000 \
    --rm \
    --cidfile=/tmp/jj7sjx1j/20240527075910-603277.cid \
    --env=TMPDIR=/tmp \
    --env=HOME=/RrTBZF \
    biocontainers/blast:v2.2.31_cv2 \
    blastp \
    -query \
    /var/lib/cwl/stgcb307528-6f4f-4251-9517-0121cb068d7e/MSTN.fasta \
    -db \
    /var/lib/cwl/stg90d0f4dc-8137-4082-8c5d-4eaf670f8786/uniprot_sprot.fasta \
    -evalue \
    0.00001 \
    -num_threads \
    8 \
    -outfmt \
    7 \
    -out \
    blastp_result.txt \
    -max_target_seqs \
    20
WARNING: The requested image's platform (linux/amd64) does not match the detected host platform (linux/arm64/v8) and no specific platform was reque
sted                                                                                                                                               INFO [job 1_blastp_docker_v3.cwl] completed success
DEBUG [job 1_blastp_docker_v3.cwl] outputs {
    "all-for-debugging": [
        {
            "location": "file:///tmp/8hge4mo3/blastp_result.txt",
            "basename": "blastp_result.txt",
            "nameroot": "blastp_result",
            "nameext": ".txt",
            "class": "File",
            "checksum": "sha1$48540e3f3f63e4dd5a26ca6899cea47e8447849e",
            "size": 1923,
            "http://commonwl.org/cwltool#generation": 0
        }
    ]
}
DEBUG [job 1_blastp_docker_v3.cwl] Removing input staging directory /tmp/4qgo5l0s
DEBUG [job 1_blastp_docker_v3.cwl] Removing temporary directory /tmp/7sot2rlv
DEBUG Moving /tmp/8hge4mo3/blastp_result.txt to /workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/blastp_result.txt
DEBUG Removing intermediate output directory /tmp/8hge4mo3
{
    "all-for-debugging": [
        {
            "location": "file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/blastp_result.txt",
            "basename": "blastp_result.txt",
            "class": "File",
            "checksum": "sha1$48540e3f3f63e4dd5a26ca6899cea47e8447849e",
            "size": 1923,
            "path": "/workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/blastp_result.txt"
        }
    ]
}INFO Final process status is success
```
:::

今回は成功しています! 結果を見てみましょう｡

https://github.com/yonesora56/togotv_cwl_for_remote_container/blob/master/zatsu_cwl_bioinformatics/blastp_result.txt

しっかり出力結果も得られています! ウシのミオスタチンと配列類似性があるタンパク質配列が得られています｡
これでCWLの修正は完了ですが､以下のように､更に改善することもできます｡

## (5) 型の指定

上記のcwlファイルの例では､E-valueの指定を`Any`にしていましたが､具体的な型を指定することで､より正確な入力を受け付けることができます｡

https://github.com/yonesora56/togotv_cwl_for_remote_container/blob/master/zatsu_cwl_bioinformatics/1_blastp_docker_v3.cwl#L41-L43

E-valueは`1e-05`のように入力するため､`float`として記載しました｡

https://github.com/yonesora56/togotv_cwl_for_remote_container/blob/master/zatsu_cwl_bioinformatics/1_blastp_docker_v4.cwl#L41-L43

これにより､`Expecting one of: ['Directory', 'File', 'boolean', 'double', 'float', 'int', 'long', 'null', 'stderr', 'stdout', 'string']` とハイライトされていた部分が解消されました｡
(この部分を修正したファイルは`v4`として保存しています)

****

# 他のファイルもCWLファイルとして記述する

blastpのプロセスのように､他のプロセスもCWLファイルとして書いてみます｡
__zatsu-cwl-generatorを使って生成し__､修正したバージョンは`v2`などのようにファイル名を変更しています｡
また､`awk`以外のプロセスはdocker imageとしてBioContainersにあるものを使用し､`hints`フィールドを追加しています｡

:::message
今回取り上げているツールのdocker imageを[docker hub](https://hub.docker.com/)から取得しました｡
[blast](https://hub.docker.com/r/biocontainers/blast)､[clustalo](https://hub.docker.com/r/biocontainers/clustalo)､[fasttree](https://hub.docker.com/r/biocontainers/fasttree)はすべて[BioContainers](https://biocontainers.pro/)によって構築されています｡
```bash:
docker image ls
REPOSITORY               TAG                 IMAGE ID       CREATED       SIZE
biocontainers/fasttree   v2.1.10-2-deb_cv1   bdfde5d6026c   4 years ago   118MB
biocontainers/clustalo   v1.2.4-2-deb_cv1    d14a5e27e301   4 years ago   118MB
biocontainers/blast      v2.2.31_cv2         5b25e08b9871   5 years ago   2.03GB
```
:::

:::details 2_awk_v2.cwl
### (1) zatsu-cwl-generatorを使って生成

```bash
zatsu-cwl-generator "awk '{ print $2 }' blastp_result.txt > blastp_result_id.txt" > 2_awk.cwl
```
awkのプロセスでは､`arguments`フィールドを追加し､以下のように修正しました｡

https://github.com/yonesora56/togotv_cwl_for_remote_container/blob/master/zatsu_cwl_bioinformatics/2_awk_v2.cwl

### (2) CWLファイルの評価

```bash
cwltool --validate 2_awk_v2.cwl
INFO /usr/local/bin/cwltool 3.1.20240508115724
INFO Resolved '2_awk_v2.cwl' to 'file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/2_awk_v2.cwl'
2_awk_v2.cwl is valid CWL.
```

### (3) 実行

```bash:実行
cwltool --debug 2_awk_v2.cwl
```
実行したファイルは以下の通りです｡

https://github.com/yonesora56/togotv_cwl_for_remote_container/blob/master/zatsu_cwl_bioinformatics/blastp_result_id.txt
:::

:::details 3_blastdbcmd_v2.cwl
### (1) zatsu-cwl-generatorを使って生成

```bash
zatsu-cwl-generator "blastdbcmd -db uniprot_sprot.fasta -entry_batch blastp_result_id.txt  -out blastdbcmd_result.fasta" --container biocontainers/blast:v2.2.31_cv2 > 3_blastdbcmd_docker.cwl
```

`1_blastp.cwl`のように､`secondaryFiles`フィールドを使って複数のインデックスファイルを指定している他､ファイル名を`string`に変更し､修正を加えました｡

https://github.com/yonesora56/togotv_cwl_for_remote_container/blob/master/zatsu_cwl_bioinformatics/3_blastdbcmd_docker_v2.cwl

### (2) CWLファイルの評価

```bash:
cwltool --validate 3_blastdbcmd_docker_v2.cwl
INFO /usr/local/bin/cwltool 3.1.20240508115724
INFO Resolved '3_blastdbcmd_docker_v2.cwl' to 'file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/3_blastdbcmd_docker_v2.cwl'
3_blastdbcmd_docker_v2.cwl is valid CWL.
```

### (3) 実行

```bash:実行
cwltool --debug 3_blastdbcmd_docker_v2.cwl
```

実行した結果は以下の通りです｡ 無事､ファイルが出力されています｡

https://github.com/yonesora56/togotv_cwl_for_remote_container/blob/master/zatsu_cwl_bioinformatics/blastdbcmd_result.fasta
:::

:::details 4_clustalo.cwl
### (1) zatsu-cwl-generatorを使って生成

```bash
zatsu-cwl-generator "clustalo -i blastdbcmd_result.fasta --outfmt=fasta -o clustalo_result.fasta" --container biocontainers/clustalo:v1.2.4-2-deb_cv1 > 4_clustalo_docker.cwl
```

### (2) CWLファイルの評価

```bash:
cwltool --validate 4_clustalo_docker.cwl
INFO /usr/local/bin/cwltool 3.1.20240508115724
INFO Resolved '4_clustalo_docker.cwl' to 'file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/4_clustalo_docker.cwl'
4_clustalo_docker.cwl is valid CWL.
```

clustaloのプロセスでは､__修正が必要なく､生成することができているようです｡__ (と思いましたが､ワークフローの記述の際に修正が必要になりました)

https://github.com/yonesora56/togotv_cwl_for_remote_container/blob/master/zatsu_cwl_bioinformatics/4_clustalo_docker.cwl

### (3) 実行

```bash:実行
cwltool --debug 4_clustalo_docker.cwl
```

実行した結果は以下の通りです｡ 無事､ファイルが出力されています｡

https://github.com/yonesora56/togotv_cwl_for_remote_container/blob/master/zatsu_cwl_bioinformatics/clustalo_result.fasta
:::

:::details 5_fasttree.cwl
### (1) zatsu-cwl-generatorを使って生成

```bash
zatsu-cwl-generator "fasttree -nt clustalo_result.fasta > fasttree_result.nwk" --container biocontainers/fasttree:v2.1.10-2-deb_cv1 > 5_fasttree_docker.cwl
```

### (2) CWLファイルの評価

```bash:
cwltool --validate 5_fasttree_docker.cwl
INFO /usr/local/bin/cwltool 3.1.20240508115724
INFO Resolved '5_fasttree_docker.cwl' to 'file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/5_fasttree_docker.cwl'
5_fasttree_docker.cwl is valid CWL.
```

fasttreeのプロセスでは､__修正が必要なく､生成することができているようです｡__

https://github.com/yonesora56/togotv_cwl_for_remote_container/blob/master/zatsu_cwl_bioinformatics/5_fasttree_docker.cwl

### (3) 実行

```bash:実行
cwltool --debug 5_fasttree_docker.cwl
```

実行した結果は以下の通りです｡ 無事､ファイルが出力されています｡

https://github.com/yonesora56/togotv_cwl_for_remote_container/blob/master/zatsu_cwl_bioinformatics/fasttree_result.nwk
:::

全てのプロセスをCWLファイルとして書くことができました｡少し修正する部分が必要でしたが､おそらくこれでワークフローも書けるはずです｡
__このように､zatsu-cwl-generatorを使うことで､簡単にCWLファイルを生成することができます｡__

:::message
### Apptainer (Singularity)も使ってみよう

dockerのコンテナは､`root`権限を持つユーザーが実行します｡
しかしながら､国立遺伝学研究所が所有しているスーパーコンピューターのような環境では､`root`権限を持つユーザーが実行することができません｡
そこで､`Singularity`を使ってコンテナを実行することができます｡
例えば､docker hubで検索したコンテナを変換することができます｡
:::

https://sc.ddbj.nig.ac.jp/#%E9%81%BA%E4%BC%9D%E7%A0%94%E3%82%B9%E3%83%BC%E3%83%91%E3%83%BC%E3%82%B3%E3%83%B3%E3%83%94%E3%83%A5%E3%83%BC%E3%82%BF%E3%82%B7%E3%82%B9%E3%83%86%E3%83%A0

https://sc.ddbj.nig.ac.jp/software/Apptainer/

****

# ワークフローを記述する

これまで､`Commandlinetool`のCWLファイル(一つ一つの処理を記述)を書きました｡
次にこれらの5つのステップを実行するワークフロー､`blast2tree.cwl`を記述していきます｡
この例では､ワークフロー全体に関するパラメータを｢1\_protein\_query｣のように数字(`1_`､`2_`)をつけています｡ 
前の処理のアウトプットを受け取る部分は｢blastp\_result: step1\_blastp/blastp\_output\_file｣のように記述し､それ以外の全ての処理に関するパラメータをinputsで記述しています｡

&nbsp;

作成したCWLファイルは以下のようになっています｡`inputs`フィールド､`steps`フィールド､`outputs`フィールドをそれぞれ書いてみました｡ 
また､後に実行する際のために､`doc`フィールドも追加しています｡

https://github.com/yonesora56/togotv_cwl_for_remote_container/blob/master/zatsu_cwl_bioinformatics/blast2tree.cwl

## 正しい記述かどうかを確認する

一旦 `cwltool --validate`で書き方を確認してみましょう｡

```bash
cwltool --validate blast2tree.cwl
INFO /usr/local/bin/cwltool 3.1.20240508115724
INFO Resolved 'blast2tree.cwl' to 'file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/blast2tree.cwl'
ERROR Tool definition failed validation:

blast2tree.cwl:99:11: Source 'all-for-debugging' of type {"type": "array", "items": ["File",
                      "Directory"]} is incompatible
blast2tree.cwl:104:7:   with sink 'nt' of type "File"
blast2tree.cwl:79:11: Source 'all-for-debugging' of type {"type": "array", "items": ["File",
                      "Directory"]} is incompatible
blast2tree.cwl:84:7:     with sink 'blastp_result' of type "File"
blast2tree.cwl:92:11: Source 'all-for-debugging' of type {"type": "array", "items": ["File",
                      "Directory"]} is incompatible
blast2tree.cwl:97:7:     with sink 'i' of type "File"
blast2tree.cwl:92:11: Source 'all-for-debugging' of type {"type": "array", "items": ["File",
                      "Directory"]} is incompatible
blast2tree.cwl:117:5:   with sink 'blastdbcmd_output' of type "File"
blast2tree.cwl:79:11: Source 'all-for-debugging' of type {"type": "array", "items": ["File",
                      "Directory"]} is incompatible
blast2tree.cwl:109:5:   with sink 'blastp_output' of type "File"
blast2tree.cwl:99:11: Source 'all-for-debugging' of type {"type": "array", "items": ["File",
                      "Directory"]} is incompatible
blast2tree.cwl:121:5:   with sink 'clustalo_output' of type "File"
```

なにやらエラーが出ています｡ `blast2tree.cwl:79:11: Source 'all-for-debugging' of type {"type": "array", "items": ["File","Directory"]} is incompatible` というように､型の不一致があるようです｡
このエラーを解消するために､`CommandLineTool`定義のそれぞれ5つのファイルを少し修正してみます｡
例えば､`1_blastp_docker_v4.cwl`の`outputs`フィールドを以下のように修正してみます｡

https://github.com/yonesora56/togotv_cwl_for_remote_container/blob/master/zatsu_cwl_bioinformatics/1_blastp_docker_v5.cwl#L56-L58

他のファイルも同様に少し修正します｡(バージョンをあげました)

:::details 修正したファイル
https://github.com/yonesora56/togotv_cwl_for_remote_container/blob/master/zatsu_cwl_bioinformatics/3_blastdbcmd_docker_v3.cwl

https://github.com/yonesora56/togotv_cwl_for_remote_container/blob/master/zatsu_cwl_bioinformatics/4_clustalo_docker_v2.cwl
:::

再度作成したCWLファイルは以下のようになっています｡

https://github.com/yonesora56/togotv_cwl_for_remote_container/blob/master/zatsu_cwl_bioinformatics/blast2tree_v2.cwl

これで再度`cwltool --validate`を実行してみます｡

```bash
cwltool --validate blast2tree_v2.cwl
INFO /usr/local/bin/cwltool 3.1.20240508115724
INFO Resolved 'blast2tree_v2.cwl' to 'file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/blast2tree_v2.cwl'
blast2tree_v2.cwl is valid CWL.
```
これでエラーが解消されました! 

## CWLviewerでワークフローを確認する

このワークフローの全体像をCWLviewerで確認しましょう｡
[`blast2tree_v2.cwl`](https://github.com/yonesora56/togotv_cwl_for_remote_container/blob/master/zatsu_cwl_bioinformatics/blast2tree_v2.cwl)をCWLviewerにアップロードしてみます｡

https://view.commonwl.org/workflows/github.com/yonesora56/togotv_cwl_for_remote_container/blob/master/zatsu_cwl_bioinformatics/blast2tree_v2.cwl

![CWLviewer3](https://storage.googleapis.com/zenn-user-upload/f4574f7c2bce-20240612.png)

全体としてはこのようになっています｡5つものステップがワークフローとして記載されていることが改めてわかります｡
(毎回自分はこれに感動しています)

## 実際に実行する

それでは実際に実行してみましょう｡
ワークフローの結果を分けるために､`--outdir`オプションを使って実行します｡

```bash
cwl-runner --outdir ./workflow_result/ blast2tree_v2.cwl
```
:::details ワークフロー実行プロセス

```bash
cwl-runner --outdir ./workflow_result/ blast2tree_v2.cwl
INFO /usr/local/bin/cwl-runner 3.1.20240508115724
INFO Resolved 'blast2tree_v2.cwl' to 'file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/blast2tree_v2.cwl'
INFO [workflow ] start
INFO [workflow ] starting step step1_blastp
INFO [step step1_blastp] start
INFO [job step1_blastp] /tmp/uj4w3esx$ docker \
    run \
    -i \
    --mount=type=bind,source=/tmp/uj4w3esx,target=/WlXEKf \
    --mount=type=bind,source=/tmp/pozju8zg,target=/tmp \
    --mount=type=bind,source=/workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/MSTN.fasta,target=/var/lib/cwl/stgced530ce-9349-463a-99b6-aa9c4fc9f28c/
MSTN.fasta,readonly \                                                                                                                                                        --mount=type=bind,source=/workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/uniprot_sprot.fasta,target=/var/lib/cwl/stg0f1360dc-539d-4d47-99f0-2d3c
6f883a16/uniprot_sprot.fasta,readonly \                                                                                                                                      --mount=type=bind,source=/workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/uniprot_sprot.fasta.phd,target=/var/lib/cwl/stg0f1360dc-539d-4d47-99f0-
2d3c6f883a16/uniprot_sprot.fasta.phd,readonly \                                                                                                                              --mount=type=bind,source=/workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/uniprot_sprot.fasta.phi,target=/var/lib/cwl/stg0f1360dc-539d-4d47-99f0-
2d3c6f883a16/uniprot_sprot.fasta.phi,readonly \                                                                                                                              --mount=type=bind,source=/workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/uniprot_sprot.fasta.phr,target=/var/lib/cwl/stg0f1360dc-539d-4d47-99f0-
2d3c6f883a16/uniprot_sprot.fasta.phr,readonly \                                                                                                                              --mount=type=bind,source=/workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/uniprot_sprot.fasta.pin,target=/var/lib/cwl/stg0f1360dc-539d-4d47-99f0-
2d3c6f883a16/uniprot_sprot.fasta.pin,readonly \                                                                                                                              --mount=type=bind,source=/workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/uniprot_sprot.fasta.pog,target=/var/lib/cwl/stg0f1360dc-539d-4d47-99f0-
2d3c6f883a16/uniprot_sprot.fasta.pog,readonly \                                                                                                                              --mount=type=bind,source=/workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/uniprot_sprot.fasta.psd,target=/var/lib/cwl/stg0f1360dc-539d-4d47-99f0-
2d3c6f883a16/uniprot_sprot.fasta.psd,readonly \                                                                                                                              --mount=type=bind,source=/workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/uniprot_sprot.fasta.psi,target=/var/lib/cwl/stg0f1360dc-539d-4d47-99f0-
2d3c6f883a16/uniprot_sprot.fasta.psi,readonly \                                                                                                                              --mount=type=bind,source=/workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/uniprot_sprot.fasta.psq,target=/var/lib/cwl/stg0f1360dc-539d-4d47-99f0-
2d3c6f883a16/uniprot_sprot.fasta.psq,readonly \                                                                                                                              --workdir=/WlXEKf \
    --read-only=true \
    --user=1000:1000 \
    --rm \
    --cidfile=/tmp/n2b6wz52/20240612103142-461008.cid \
    --env=TMPDIR=/tmp \
    --env=HOME=/WlXEKf \
    biocontainers/blast:v2.2.31_cv2 \
    blastp \
    -query \
    /var/lib/cwl/stgced530ce-9349-463a-99b6-aa9c4fc9f28c/MSTN.fasta \
    -db \
    /var/lib/cwl/stg0f1360dc-539d-4d47-99f0-2d3c6f883a16/uniprot_sprot.fasta \
    -evalue \
    0.01 \
    -num_threads \
    8 \
    -outfmt \
    6 \
    -out \
    blastp_output.txt \
    -max_target_seqs \
    20
WARNING: The requested image's platform (linux/amd64) does not match the detected host platform (linux/arm64/v8) and no specific platform was requested
INFO [job step1_blastp] Max memory used: 0MiB
INFO [job step1_blastp] completed success
INFO [step step1_blastp] completed success
INFO [workflow ] starting step step2_awk
INFO [step step2_awk] start
INFO [job step2_awk] /tmp/6lqwjots$ awk \
    '{ print $2 }' \
    /tmp/4qdjlcga/stge55ffb48-5bb9-454a-a7ff-c429efc74aaf/blastp_output.txt > /tmp/6lqwjots/blastp_result_id.txt
INFO [job step2_awk] completed success
INFO [step step2_awk] completed success
INFO [workflow ] starting step step3_blastdbcmd
INFO [step step3_blastdbcmd] start
INFO [job step3_blastdbcmd] /tmp/rdv3hv2x$ docker \
    run \
    -i \
    --mount=type=bind,source=/tmp/rdv3hv2x,target=/WlXEKf \
    --mount=type=bind,source=/tmp/r1olsbb3,target=/tmp \
    --mount=type=bind,source=/workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/uniprot_sprot.fasta,target=/var/lib/cwl/stgc0263af5-6318-4781-8db1-a5fa
4f0738bb/uniprot_sprot.fasta,readonly \                                                                                                                                      --mount=type=bind,source=/workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/uniprot_sprot.fasta.phd,target=/var/lib/cwl/stgc0263af5-6318-4781-8db1-
a5fa4f0738bb/uniprot_sprot.fasta.phd,readonly \                                                                                                                              --mount=type=bind,source=/workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/uniprot_sprot.fasta.phi,target=/var/lib/cwl/stgc0263af5-6318-4781-8db1-
a5fa4f0738bb/uniprot_sprot.fasta.phi,readonly \                                                                                                                              --mount=type=bind,source=/workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/uniprot_sprot.fasta.phr,target=/var/lib/cwl/stgc0263af5-6318-4781-8db1-
a5fa4f0738bb/uniprot_sprot.fasta.phr,readonly \                                                                                                                              --mount=type=bind,source=/workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/uniprot_sprot.fasta.pin,target=/var/lib/cwl/stgc0263af5-6318-4781-8db1-
a5fa4f0738bb/uniprot_sprot.fasta.pin,readonly \                                                                                                                              --mount=type=bind,source=/workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/uniprot_sprot.fasta.pog,target=/var/lib/cwl/stgc0263af5-6318-4781-8db1-
a5fa4f0738bb/uniprot_sprot.fasta.pog,readonly \                                                                                                                              --mount=type=bind,source=/workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/uniprot_sprot.fasta.psd,target=/var/lib/cwl/stgc0263af5-6318-4781-8db1-
a5fa4f0738bb/uniprot_sprot.fasta.psd,readonly \                                                                                                                              --mount=type=bind,source=/workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/uniprot_sprot.fasta.psi,target=/var/lib/cwl/stgc0263af5-6318-4781-8db1-
a5fa4f0738bb/uniprot_sprot.fasta.psi,readonly \                                                                                                                              --mount=type=bind,source=/workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/uniprot_sprot.fasta.psq,target=/var/lib/cwl/stgc0263af5-6318-4781-8db1-
a5fa4f0738bb/uniprot_sprot.fasta.psq,readonly \                                                                                                                              --mount=type=bind,source=/tmp/6lqwjots/blastp_result_id.txt,target=/var/lib/cwl/stg3930e103-780f-4d8c-bdca-c36a8e0359e3/blastp_result_id.txt,readonly \
    --workdir=/WlXEKf \
    --read-only=true \
    --user=1000:1000 \
    --rm \
    --cidfile=/tmp/4hh0ljvr/20240612103143-671398.cid \
    --env=TMPDIR=/tmp \
    --env=HOME=/WlXEKf \
    biocontainers/blast:v2.2.31_cv2 \
    blastdbcmd \
    -db \
    /var/lib/cwl/stgc0263af5-6318-4781-8db1-a5fa4f0738bb/uniprot_sprot.fasta \
    -entry_batch \
    /var/lib/cwl/stg3930e103-780f-4d8c-bdca-c36a8e0359e3/blastp_result_id.txt \
    -out \
    blastdbcmd_result.fasta
WARNING: The requested image's platform (linux/amd64) does not match the detected host platform (linux/arm64/v8) and no specific platform was requested
INFO [job step3_blastdbcmd] completed success
INFO [step step3_blastdbcmd] completed success
INFO [workflow ] starting step step4_clustalo
INFO [step step4_clustalo] start
INFO [job step4_clustalo] /tmp/xz4llstr$ docker \
    run \
    -i \
    --mount=type=bind,source=/tmp/xz4llstr,target=/WlXEKf \
    --mount=type=bind,source=/tmp/c8axor6c,target=/tmp \
    --mount=type=bind,source=/tmp/rdv3hv2x/blastdbcmd_result.fasta,target=/var/lib/cwl/stg8426b713-ef9d-46bc-8e36-daaa52a1f2f8/blastdbcmd_result.fasta,readonly \
    --workdir=/WlXEKf \
    --read-only=true \
    --user=1000:1000 \
    --rm \
    --cidfile=/tmp/6ykuaryi/20240612103144-728058.cid \
    --env=TMPDIR=/tmp \
    --env=HOME=/WlXEKf \
    biocontainers/clustalo:v1.2.4-2-deb_cv1 \
    clustalo \
    -i \
    /var/lib/cwl/stg8426b713-ef9d-46bc-8e36-daaa52a1f2f8/blastdbcmd_result.fasta \
    --outfmt=fasta \
    -o \
    clustalo_output.fasta
WARNING: The requested image's platform (linux/amd64) does not match the detected host platform (linux/arm64/v8) and no specific platform was requested
INFO [job step4_clustalo] completed success
INFO [step step4_clustalo] completed success
INFO [workflow ] starting step step5_fasttree
INFO [step step5_fasttree] start
INFO [job step5_fasttree] /tmp/9_v2dbe8$ docker \
    run \
    -i \
    --mount=type=bind,source=/tmp/9_v2dbe8,target=/WlXEKf \
    --mount=type=bind,source=/tmp/k186douq,target=/tmp \
    --mount=type=bind,source=/tmp/xz4llstr/clustalo_output.fasta,target=/var/lib/cwl/stgb1ab0516-de4a-4a6c-b9ad-47f48f617aa4/clustalo_output.fasta,readonly \
    --workdir=/WlXEKf \
    --read-only=true \
    --log-driver=none \
    --user=1000:1000 \
    --rm \
    --cidfile=/tmp/9an6c2tv/20240612103145-770529.cid \
    --env=TMPDIR=/tmp \
    --env=HOME=/WlXEKf \
    biocontainers/fasttree:v2.1.10-2-deb_cv1 \
    fasttree \
    -nt \
    /var/lib/cwl/stgb1ab0516-de4a-4a6c-b9ad-47f48f617aa4/clustalo_output.fasta > /tmp/9_v2dbe8/fasttree_result.nwk
WARNING: The requested image's platform (linux/amd64) does not match the detected host platform (linux/arm64/v8) and no specific platform was requested
FastTree Version 2.1.10 Double precision (No SSE3)
Alignment: /var/lib/cwl/stgb1ab0516-de4a-4a6c-b9ad-47f48f617aa4/clustalo_output.fasta
Nucleotide distances: Jukes-Cantor Joins: balanced Support: SH-like 1000
Search: Normal +NNI +SPR (2 rounds range 10) +ML-NNI opt-each=1
TopHits: 1.00*sqrtN close=default refresh=0.80
ML Model: Jukes-Cantor, CAT approximation with 20 rate categories
Ignored unknown character D (seen 311 times)
Ignored unknown character E (seen 310 times)
Ignored unknown character F (seen 176 times)
Ignored unknown character H (seen 77 times)
Ignored unknown character I (seen 332 times)
Ignored unknown character K (seen 364 times)
Ignored unknown character L (seen 451 times)
Ignored unknown character M (seen 110 times)
Ignored unknown character P (seen 312 times)
Ignored unknown character Q (seen 232 times)
Ignored unknown character R (seen 237 times)
Ignored unknown character S (seen 278 times)
Ignored unknown character V (seen 279 times)
Ignored unknown character W (seen 78 times)
Ignored unknown character X (seen 205 times)
Ignored unknown character Y (seen 169 times)
WARNING! ONLY 19.6% NUCLEOTIDE CHARACTERS -- IS THIS REALLY A NUCLEOTIDE ALIGNMENT?
Initial topology in 0.01 seconds
Refining topology: 15 rounds ME-NNIs, 2 rounds ME-SPRs, 7 rounds ML-NNIs
Total branch-length 0.107 after 0.01 sec
ML-NNI round 1: LogLk = -167.434 NNIs 5 max delta 0.00 Time 0.02
Switched to using 20 rate categories (CAT approximation)
Rate categories were divided by 0.628 so that average rate = 1.0
CAT-based log-likelihoods may not be comparable across runs
Use -gamma for approximate but comparable Gamma(20) log-likelihoods
ML-NNI round 2: LogLk = -165.205 NNIs 3 max delta 0.00 Time 0.02
Turning off heuristics for final round of ML NNIs (converged)
ML-NNI round 3: LogLk = -165.204 NNIs 2 max delta 0.00 Time 0.03 (final)
Optimize all lengths: LogLk = -165.204 Time 0.03
Total time: 0.06 seconds Unique: 13/20 Bad splits: 0/10
INFO [job step5_fasttree] completed success
INFO [step step5_fasttree] completed success
INFO [workflow ] completed success
{
    "awk_output": {
        "location": "file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/workflow_result/blastp_result_id.txt",
        "basename": "blastp_result_id.txt",
        "class": "File",
        "checksum": "sha1$354af893fc9afa558d0a01d6d68bf9d3a935cb53",
        "size": 418,
        "path": "/workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/workflow_result/blastp_result_id.txt"
    },
    "blastdbcmd_output": {
        "location": "file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/workflow_result/blastdbcmd_result.fasta",
        "basename": "blastdbcmd_result.fasta",
        "class": "File",
        "checksum": "sha1$e9167fd8855ce938590f64e179b3aedb0edd9e15",
        "size": 9581,
        "path": "/workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/workflow_result/blastdbcmd_result.fasta"
    },
    "blastp_output": {
        "location": "file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/workflow_result/blastp_output.txt",
        "basename": "blastp_output.txt",
        "class": "File",
        "checksum": "sha1$31464e2d0bdfd65f89e0542688844d3686bf278c",
        "size": 1536,
        "path": "/workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/workflow_result/blastp_output.txt"
    },
    "clustalo_output": {
        "location": "file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/workflow_result/clustalo_output.fasta",
        "basename": "clustalo_output.fasta",
        "class": "File",
        "checksum": "sha1$57ec8d531af2e6a5a0add1f1fc8ff413a12d62f6",
        "size": 9621,
        "path": "/workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/workflow_result/clustalo_output.fasta"
    },
    "fasttree_output": {
        "location": "file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/workflow_result/fasttree_result.nwk",
        "basename": "fasttree_result.nwk",
        "class": "File",
        "checksum": "sha1$90a12b35e23309a1e73fac58d05794beeab49a38",
        "size": 807,
        "path": "/workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/workflow_result/fasttree_result.nwk"
    }
}INFO Final process status is success
```
:::

無事､全ての処理が終了しているようです! それでは､実際に出力されたファイルを見てみましょう｡

https://github.com/yonesora56/togotv_cwl_for_remote_container/blob/master/zatsu_cwl_bioinformatics/workflow_result/fasttree_result.nwk

無事､出力されています｡このように､ワークフローをCWLで記述することで､一つのコマンドで複数の処理を一度に実行することができます｡

&nbsp;

:::message
### `doc`フィールドを積極的に書こう!

複雑なプロセスを書いたあと､見返した際に｢あれ､このプロセスはなんだっけ...｣となってしまうことは容易に想像できます｡
そこで､CWLファイルを書く際には､`doc`フィールドを積極的に書くことをおすすめします｡

例えば､先程作成した`Workflow`定義のCWLファイルに対して`--help`オプションをつけて実行してみます(位置に注意!)｡

```bash:
cwltool blast2tree_v2.cwl --help
INFO /usr/local/bin/cwltool 3.1.20240508115724
INFO Resolved 'blast2tree_v2.cwl' to 'file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/blast2tree_v2.cwl'
usage: blast2tree_v2.cwl [-h] [--1_protein_query 1_PROTEIN_QUERY] [--2_protein_database 2_PROTEIN_DATABASE] [--3_evalue 3_EVALUE]
                         [--4_number_of_threads 4_NUMBER_OF_THREADS] [--5_outformat_type 5_OUTFORMAT_TYPE] [--6_output_file_name 6_OUTPUT_FILE_NAME]
                         [--7_max_target_sequence 7_MAX_TARGET_SEQUENCE] [--8_blastdbcmd_protein_database 8_BLASTDBCMD_PROTEIN_DATABASE]
                         [--10_clustalo_output_name 10_CLUSTALO_OUTPUT_NAME]
                         [job_order]

blastp, awk, blastdbcmd, clustalo, and fasttreeの5つのステップを実行

positional arguments:
  job_order             Job input json file

options:
  -h, --help            show this help message and exit
  --1_protein_query 1_PROTEIN_QUERY
                        blastp process input protein query file(fasta format)
  --2_protein_database 2_PROTEIN_DATABASE
                        blastp process input all index file
  --3_evalue 3_EVALUE   blastp process evalue cutoff
  --4_number_of_threads 4_NUMBER_OF_THREADS
                        number of threads
  --5_outformat_type 5_OUTFORMAT_TYPE
                        blastp process output format type
  --6_output_file_name 6_OUTPUT_FILE_NAME
                        blastp process output file name
  --7_max_target_sequence 7_MAX_TARGET_SEQUENCE
                        blastp process max target sequence
  --8_blastdbcmd_protein_database 8_BLASTDBCMD_PROTEIN_DATABASE
                        blastdbcmd process protein database
  --10_clustalo_output_name 10_CLUSTALO_OUTPUT_NAME
                        clustalo process output name
```
このように､`doc`フィールドを書いておくことで､プロセスの説明を見ることができます｡
:::

# 終わりに

今回は､バイオインフォマティクス分野で使用されるツールをCWLファイルとして記述し､ワークフローまで実行するプロセスを記述しました｡
zatsu-cwl-generatorを使うことで､簡単にCWLファイルを生成することで､CWLを記述するハードルを下げることができます｡
これまでの記事で説明しているプロセスとしては､まとめると以下のようになっています｡

1. zatsu-cwl-generatorを使って､コマンドラインツールの処理をCWLファイルとして生成する
2. cwltoolの `--validate`オプションを使って､CWLファイルをチェックする
3. 問題がある場合､修正を行う(ドキュメントを読む､など)
4. cwltoolを使って､CWLファイルを実行する(`--debug`オプションをつけておく)
5. エラーが出た場合､修正を行う

このように､ __zatsu-cwl-generator__ をCWLファイルを書くプロセスの起点として利用することが可能です｡
ぜひ､皆さんも活用して素晴らしいCWLライフを満喫してください!

&nbsp;

## 参考

これまでの3つの記事では詳しい部分についてはあまり解説していません｡
CWLの詳細な記述方法については､以下のようなドキュメントが充実しているので､ぜひそちらを参考にしてください｡

https://www.commonwl.org/user_guide/#common-workflow-language-user-guide

https://www.commonwl.org/user_guide/topics/best-practices.html#best-practices

&nbsp;

:::details 参考：Dockerでコマンド実行する例
```bash:
# Step1
docker run --rm -it -v `pwd`:`pwd` -w `pwd` biocontainers/blast:v2.2.31_cv2 blastp -query MSTN.fasta -db uniprot_sprot.fasta -evalue 1e-5 -num_threads 4 -outfmt 6 -out blastp_result_MSTN.txt -max_target_seqs 20 

# Step2
docker run --rm -it -v `pwd`:`pwd` -w `pwd` ubuntu:23.10 awk '{ print $2 }' `pwd`/data/blastp_result.txt > `pwd`/data/blastp_result_id.txt

#Step3
docker run --rm -it -v `pwd`:`pwd`  -w  `pwd` biocontainers/blast:v2.2.31_cv2 blastdbcmd -db uniprot_sprot.fasta -entry_batch blastp_result_id.txt  -out blastdbcmd_result.fasta

#Step4
docker run --rm -it -v `pwd`:`pwd` -w `pwd` biocontainers/clustalo:v1.2.4-2-deb_cv1 clustalo -i `pwd`/blastp_results_MSTN.fasta --outfmt=fasta -o clustalo_result.fasta

#Step5
docker run --rm -it -v `pwd`:`pwd` -w `pwd` biocontainers/fasttree:v2.1.10-2-deb_cv1 FastTree -boot 100 -out MSTN_tree.newick `pwd`/clustalo_result.fasta
```
:::

&nbsp;

__今回の記事で使用したCWLのファイルをおいているリポジトリは以下からアクセスすることができます｡__
https://github.com/yonesora56/togotv_cwl_for_remote_container