---
title: "CWLの作成環境をVScodeの｢Dev Containers｣の機能を使って構築する (3)"
emoji: "📑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["CWL", "bioinformatics"]
published: false
---
__※今回の記事で使用したCWLファイルをおいているリポジトリは以下からアクセスすることができます__
https://github.com/yonezawa-sora/togotv_cwl_for_remote_container

:::message alert
この赤いフィールドは後ほど修正する点をメモしています
:::

:::message alert
__修正3：dockerの説明のあとにsingularityの説明を加える__
:::

# バイオインフォマティクスの解析手順をワークフロー化する

これまで､環境構築､`grep`と`wc`の処理に関するワークフローを作成する過程を紹介しました｡
このドキュメントでは､具体的にバイオインフォマティクス研究で使用されているツールを使った例をcwlとして記述し､ __ワークフロー__ として実行する部分までをご紹介します｡
このプロセスにおいては､zatsu-cwl-generatorを使ってCWLファイルを作成し､その後に細かい修正を加えていきながら作成していきます｡

&nbsp;

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

### 1\. クエリ配列

blastpを実行する際にクエリとするタンパク質配列データとして､ウシのミオスタチンのタンパク質配列のfastaファイル (MSTN.fastaに名前を変更しています)を使用します｡ 
curlコマンドでUniProtから取得しました｡

```bash:
curl -O https://rest.uniprot.org/uniprotkb/O18836.fasta
```

### 2\. インデックス(データベース)

次に､BLASTのデータベースとして､UniProtのタンパク質配列のfastaファイル(uniprot\_sprot.fasta.gz)をftpサイトよりcurlコマンドで取得し､展開します｡ 

```bash:
curl -O https://ftp.uniprot.org/pub/databases/uniprot/current_release/knowledgebase/complete/uniprot_sprot.fasta.gz
unpigz -p 10 uniprot_sprot.fasta.gz
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

&nbsp;

## zatsu-cwl-generatorを使ってblastpを実行するcwlファイルを書く

次に､上記の処理について､cwlファイルを記述していきましょう｡ zatsu-cwl-generatorを使ってコードを生成し､修正しながら作成していきます｡
今回は､`zatsu_cwl_bioinformatics`ディレクトリで作業を行っています｡修正のプロセスはトグル内に示してあるので､ぜひご覧ください｡
以下では､最初のプロセス､blastpのコマンドを例にして(修正のプロセスも含めながら)説明していきます｡

### (1) zatsu-cwl-generatorを使ってblastpのCWLファイルを生成する

これまでと同様に､zatsu-cwl-generatorを使ってCWLファイルを生成します｡

```bash:
zatsu-cwl-generator 'blastp -query MSTN.fasta -db uniprot_sprot.fasta -evalue 1e-5 -num_threads 4 -outfmt 6 -out blastp_result.txt -max_target_seqs 20' > 1_blastp.cwl
```

https://github.com/yonesora56/togotv_cwl_for_remote_container/blob/master/zatsu_cwl_bioinformatics/1_blastp.cwl

`--validate`でチェックします｡
  
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
生成されたファイルは問題がなさそうです｡

:::message
ここで一旦実行してみようと思いますが､例えばツールをローカルでダウンロードしておらず､docker imageを使って実行したいという状況があるかもしれません｡
例えば､この記事を作成している環境は､Mac M1 でDev containersの機能を使用しているため､環境としては __Linux ARM64__ という状況です｡
現在､LinuxのARM64に対応したツールは少なく､この場合ではdockerで動かしたほうが簡単そうです｡
しかしながら､ __dockerを使って実行する場合には､どのようにcwlファイルを記述すればよいのでしょうか?__
これについて､次のセクションで説明していきます｡
:::

&nbsp;

### (2) コンテナオプション(`-c`)を使って出力する

CWLでは､docker imageを使って実行する場合､これまでのcwlファイルでは書いていなかった`hints`フィールド(あるいは`requirements`フィールド)を使って記述します｡

https://www.commonwl.org/user_guide/topics/using-containers.html#using-containers

__zatsu-cwl-generatorでは､`-c`, `--container` オプションを使うことで､コンテナを使用するコマンドを出力することができます｡__

https://qiita.com/tm_tn/items/2c789c5b3c28e3eb3c9a

&nbsp;

以下のようにコマンドを実行します｡

```bash
zatsu-cwl-generator 'blastp -query MSTN.fasta -db uniprot_sprot.fasta -evalue 1e-5 -num_threads 4 -outfmt 6 -out blastp_result.txt -max_target_seqs 20' --container biocontainers/blast:v2.2.31_cv2 > 1_blastp_docker.cwl
```
これまでと同じようにコマンドを''でくくったあとに､`-c` もしくは`--container` オプションをつけて､image(tagの情報も含めて)を記載します｡ 
すると､以下のようにCWLファイルが出力されます｡

https://github.com/yonesora56/togotv_cwl_for_remote_container/blob/master/zatsu_cwl_bioinformatics/1_blastp_docker.cwl

これまでと同じように出力されますが､コンテナオプションをつけると､下に`hints`フィールドが出力されています｡

https://github.com/yonesora56/togotv_cwl_for_remote_container/blob/master/zatsu_cwl_bioinformatics/1_blastp_docker.cwl#L56-L58

このように､`hints`フィールドを使ってdocker imageを指定することができます｡

https://www.commonwl.org/v1.0/CommandLineTool.html#DockerRequirement

`--validate`を使ってチェックします｡

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
今のところ問題はなさそうです｡
それではこのファイルを使って実行してみましょう｡

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
それではこれについて修正してみましょう｡

&nbsp;

### (3) 修正プロセス 1

`inputs`フィールドに書いてある部分が修正が必要のようです｡
https://github.com/yonesora56/togotv_cwl_for_remote_container/blob/master/zatsu_cwl_bioinformatics/1_blastp_docker.cwl#L41-L45

ここは､ファイルそのものではなく､__ファイル名__ になるはずなので､以下のように修正してみました｡

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
今回もファイルの出力はうまくいきましたが､空のファイルが出力されてしまいました｡
エラーを見てみると､`BLAST Database error: No alias or index file found for protein database [/var/lib/cwl/stgb5786c95-6694-4f59-9707-d79eced66e64/uniprot_sprot.fasta]`などが見られ､データベースが見つからないというエラーが出ています｡
これはBLASTなどに見られるエラーで､インデックス作成時に複数のインデックスファイルも含めて記述する必要があります｡
それでは､これを修正してみましょう｡

&nbsp;

### (4) 修正プロセス 2 複数のインデックスファイルの記載

`inputs`フィールドの部分を修正してみます｡

https://github.com/yonesora56/togotv_cwl_for_remote_container/blob/master/zatsu_cwl_bioinformatics/1_blastp_docker_v2.cwl#L27-L31

BLASTでは､inputsのインデックス(引数は `-db` )の指定の部分の記述方法についていくつか方法があるようです｡

https://qiita.com/Yohei__K/items/b947655eb59a8853c172

今回は `secondaryFiles` フィールドを活用し､以下のように記述してみます｡

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

https://github.com/yonesora56/togotv_cwl_for_remote_container/blob/master/zatsu_cwl_bioinformatics/out/blastp_result.txt

しっかり出力結果も得られています! ウシのミオスタチンと配列類似性があるタンパク質配列が得られています｡
これでCWLの修正は完了ですが､以下のように､更に改善することもできます｡

上記のcwlファイルの例では､E-valueの指定を`Any`にしていましたが､具体的な型を指定することで､より正確な入力を受け付けることができます｡

https://github.com/yonesora56/togotv_cwl_for_remote_container/blob/master/zatsu_cwl_bioinformatics/1_blastp_docker_v3.cwl#L41-L43

E-valueは`1e-05`のように入力するため､`float`として記載しました｡

https://github.com/yonesora56/togotv_cwl_for_remote_container/blob/master/zatsu_cwl_bioinformatics/1_blastp_docker_v4.cwl#L41-L43

これにより､`Expecting one of: ['Directory', 'File', 'boolean', 'double', 'float', 'int', 'long', 'null', 'stderr', 'stdout', 'string']` とハイライトされていた部分が解消されました｡
(この部分を修正したファイルは`v4`として保存し､最後のセクションでワークフローを書く際にはこのファイルを使います｡)

****

## 他のファイルもcwlファイルとして書いてみる

blastpのプロセスのように､他のプロセスもcwlファイルとして書いてみます｡
__zatsu-cwl-generatorを使って生成し__､修正したバージョンは`v2`などのようにファイル名を変更しています｡
また､`awk`以外のプロセスはdocker imageとしてBioContainersにあるものを使用し､`hints`フィールドを追加しています｡

:::message
今回取り上げているツールのdocker imageを[docker hub](https://hub.docker.com/)から取得しました｡
[blast](https://hub.docker.com/r/biocontainers/blast)､[clustalo](https://hub.docker.com/r/biocontainers/clustalo)､[fasttree](https://hub.docker.com/r/biocontainers/fasttree)はすべて[BioContainers](https://biocontainers.pro/)によって構築されています[^1]｡
```bash:
docker image ls
REPOSITORY               TAG                 IMAGE ID       CREATED       SIZE
biocontainers/fasttree   v2.1.10-2-deb_cv1   bdfde5d6026c   4 years ago   118MB
biocontainers/clustalo   v1.2.4-2-deb_cv1    d14a5e27e301   4 years ago   118MB
biocontainers/blast      v2.2.31_cv2         5b25e08b9871   5 years ago   2.03GB
```
:::

:::details 2_awk_v2.cwl

#### zatsu-cwl-generatorを使って生成

```bash
zatsu-cwl-generator "awk '{ print $2 }' blastp_result.txt > blastp_result_id.txt" > 2_awk.cwl
```

awkのプロセスでは､`arguments`フィールドを追加し､以下のように修正しました｡

https://github.com/yonesora56/togotv_cwl_for_remote_container/blob/master/zatsu_cwl_bioinformatics/2_awk_v2.cwl

```bash:実行
cwltool --debug 2_awk_v2.cwl
```
実行したファイルは以下の通りです｡

https://github.com/yonesora56/togotv_cwl_for_remote_container/blob/master/zatsu_cwl_bioinformatics/out/blastp_result_id.txt
:::

:::details 3_blastdbcmd_v2.cwl

#### zatsu-cwl-generatorを使って生成

```bash
zatsu-cwl-generator "blastdbcmd -db uniprot_sprot.fasta -entry_batch blastp_result_id.txt  -out blastdbcmd_result.fasta" --container biocontainers/blast:v2.2.31_cv2 > 3_blastdbcmd_docker.cwl
```

最初のblastp検索のように､`secondaryFiles`フィールドを使って複数のインデックスファイルを指定している他､ファイル名を`string`に変更しています｡

https://github.com/yonesora56/togotv_cwl_for_remote_container/blob/master/zatsu_cwl_bioinformatics/3_blastdbcmd_docker_v2.cwl

```bash:実行
cwltool --debug 3_blastdbcmd_docker_v2.cwl
```

実行した結果は以下の通りです｡ 無事､ファイルが出力されています｡

https://github.com/yonesora56/togotv_cwl_for_remote_container/blob/master/zatsu_cwl_bioinformatics/out/blastdbcmd_result.fasta
:::

:::details 4_clustalo.cwl

#### zatsu-cwl-generatorを使って生成

```bash
zatsu-cwl-generator "clustalo -i blastdbcmd_result.fasta --outfmt=fasta -o clustalo_result.fasta" --container biocontainers/clustalo:v1.2.4-2-deb_cv1 > 4_clustalo_docker.cwl
```

clustaloのプロセスでは､__修正が必要なく､生成することができました__

https://github.com/yonesora56/togotv_cwl_for_remote_container/blob/master/zatsu_cwl_bioinformatics/4_clustalo_docker.cwl

```bash:実行
cwltool --debug 4_clustalo_docker.cwl
```

実行した結果は以下の通りです｡ 無事､ファイルが出力されています｡

https://github.com/yonesora56/togotv_cwl_for_remote_container/blob/master/zatsu_cwl_bioinformatics/out/clustalo_result.fasta
:::

:::details 5_fasttree.cwl

#### zatsu-cwl-generatorを使って生成

```bash
zatsu-cwl-generator "fasttree -nt clustalo_result.fasta > fasttree_result.nwk" --container biocontainers/fasttree:v2.1.10-2-deb_cv1 > 5_fasttree_docker.cwl
```
fasttreeのプロセスでは､__修正が必要なく､生成することができました__

https://github.com/yonesora56/togotv_cwl_for_remote_container/blob/master/zatsu_cwl_bioinformatics/5_fasttree_docker.cwl

```bash:実行
cwltool --debug 5_fasttree_docker.cwl
```

実行した結果は以下の通りです｡ 無事､ファイルが出力されています｡

https://github.com/yonesora56/togotv_cwl_for_remote_container/blob/master/zatsu_cwl_bioinformatics/out/fasttree_result.nwk
:::

これで全てのプロセスをCWLファイルとして書くことができました｡
__このように､zatsu-cwl-generatorを使うことで､簡単にCWLファイルを生成することができます｡__

:::message
### Apptainer (Singularity)も使ってみよう
dockerを

:::
https://sc.ddbj.nig.ac.jp/software/Apptainer/

****

## ワークフローを記述する

これまで､Commandlinetoolのcwlファイル(一つ一つの処理を記述)を書きました｡
次にこれらの5つのステップを実行するワークフロー､`blast2tree.cwl`を記述していきます｡
この例では､ワークフロー全体に関するパラメータを｢1\_protein\_query｣のように数字(`1_`､`2_`)をつけています｡ 
前の処理のアウトプットを受け取る部分は｢blastp\_result: step1\_blastp/blastp\_output\_file｣のように記述し､それ以外の全ての処理に関するパラメータをinputsで記述しています｡

&nbsp;

作成したcwlファイルは以下のようになっています｡
`inputs`フィールド､`steps`フィールド､`outputs`フィールドをそれぞれ書いてみました｡ また､後々に実行する際のために､`doc`フィールドも追加しています｡

https://github.com/yonesora56/togotv_cwl_for_remote_container/blob/master/zatsu_cwl_bioinformatics/blast2tree.cwl

一旦 `--validate`で書き方を確認してみましょう｡

```bash
cwltool --validate blast2tree.cwl
INFO /usr/local/bin/cwltool 3.1.20240508115724
INFO Resolved 'blast2tree.cwl' to 'file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/blast2tree.cwl'
2_awk_v2.cwl:14:7: Warning: Field 'location' contains undefined reference to
                   'file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/blastp_result.txt'
WARNING 2_awk_v2.cwl:14:7: Warning: Field 'location' contains undefined reference to
                   'file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/blastp_result.txt'
3_blastdbcmd_docker_v2.cwl:32:7: Warning: Field 'location' contains undefined reference to
                                 'file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/blastp_result_id.txt'
WARNING 3_blastdbcmd_docker_v2.cwl:32:7: Warning: Field 'location' contains undefined reference to
                                 'file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/blastp_result_id.txt'
4_clustalo_docker.cwl:17:7: Warning: Field 'location' contains undefined reference to
                            'file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/blastdbcmd_result.fasta'
WARNING 4_clustalo_docker.cwl:17:7: Warning: Field 'location' contains undefined reference to
                            'file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/blastdbcmd_result.fasta'
5_fasttree_docker.cwl:14:7: Warning: Field 'location' contains undefined reference to
                            'file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/clustalo_result.fasta'
WARNING 5_fasttree_docker.cwl:14:7: Warning: Field 'location' contains undefined reference to
                            'file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/clustalo_result.fasta'
ERROR Tool definition failed validation:

blast2tree.cwl:79:11: Source 'all-for-debugging' of type {"type": "array", "items": ["File",
                      "Directory"]} is incompatible
blast2tree.cwl:84:7:     with sink 'blastp_result' of type "File"
blast2tree.cwl:99:11: Source 'all-for-debugging' of type {"type": "array", "items": ["File",
                      "Directory"]} is incompatible
blast2tree.cwl:104:7:   with sink 'nt' of type "File"
blast2tree.cwl:92:11: Source 'all-for-debugging' of type {"type": "array", "items": ["File",
                      "Directory"]} is incompatible
blast2tree.cwl:97:7:     with sink 'i' of type "File"
```

エラーが出ています｡ `blast2tree.cwl:79:11: Source 'all-for-debugging' of type {"type": "array", "items": ["File","Directory"]} is incompatible` というように､型の不一致があるようです｡
このエラーを解消するために､`CommandLineTool`定義のそれぞれ5つのファイルを少し修正してみます｡
例えば､`1_blastp_docker_v4.cwl`の`outputs`フィールドを以下のように修正してみます｡

https://github.com/yonesora56/togotv_cwl_for_remote_container/blob/master/zatsu_cwl_bioinformatics/1_blastp_docker_v5.cwl#L56-L58

他のファイルも同様に少し修正します｡(バージョンをあげました)

:::details 修正したファイル

https://github.com/yonesora56/togotv_cwl_for_remote_container/blob/master/zatsu_cwl_bioinformatics/3_blastdbcmd_docker_v3.cwl

https://github.com/yonesora56/togotv_cwl_for_remote_container/blob/master/zatsu_cwl_bioinformatics/4_clustalo_docker_v2.cwl
:::

これで再度`--validate`を実行してみます｡

```bash
cwltool --validate blast2tree_v2.cwl
INFO /usr/local/bin/cwltool 3.1.20240508115724
INFO Resolved 'blast2tree_v2.cwl' to 'file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/blast2tree_v2.cwl'
blast2tree_v2.cwl is valid CWL.
```

これでエラーが解消されました! それでは実際に実行してみましょう｡
ワークフローの結果を分けるために､`--outdir`オプションを使って実行します｡

```bash
cwl-runner --outdir ./workflow_result/ blast2tree_v2.cwl
```
:::details ワークフロー実行プロセス

```bash
INFO /usr/local/bin/cwl-runner 3.1.20240508115724
INFO Resolved 'blast2tree_v2.cwl' to 'file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/blast2tree_v2.cwl'
INFO [workflow ] start
INFO [workflow ] starting step step1_blastp
INFO [step step1_blastp] start
INFO [job step1_blastp] /tmp/whqd9v49$ docker \
    run \
    -i \
    --mount=type=bind,source=/tmp/whqd9v49,target=/SXZFEh \
    --mount=type=bind,source=/tmp/_1phxxww,target=/tmp \
    --mount=type=bind,source=/workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/MSTN.fasta,target=/var/lib/cwl/stg3a2a6105-7709-4108-b72c-7736725ede96/MSTN.fasta,readonly \
    --mount=type=bind,source=/workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/uniprot_sprot.fasta,target=/var/lib/cwl/stg9d3f538c-a076-4b8c-89bb-3596859ae6bb/uniprot_sprot.fasta,readonly \
    --mount=type=bind,source=/workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/uniprot_sprot.fasta.phd,target=/var/lib/cwl/stg9d3f538c-a076-4b8c-89bb-3596859ae6bb/uniprot_sprot.fasta.phd,readonly \
    --mount=type=bind,source=/workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/uniprot_sprot.fasta.phi,target=/var/lib/cwl/stg9d3f538c-a076-4b8c-89bb-3596859ae6bb/uniprot_sprot.fasta.phi,readonly \
    --mount=type=bind,source=/workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/uniprot_sprot.fasta.phr,target=/var/lib/cwl/stg9d3f538c-a076-4b8c-89bb-3596859ae6bb/uniprot_sprot.fasta.phr,readonly \
    --mount=type=bind,source=/workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/uniprot_sprot.fasta.pin,target=/var/lib/cwl/stg9d3f538c-a076-4b8c-89bb-3596859ae6bb/uniprot_sprot.fasta.pin,readonly \
    --mount=type=bind,source=/workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/uniprot_sprot.fasta.pog,target=/var/lib/cwl/stg9d3f538c-a076-4b8c-89bb-3596859ae6bb/uniprot_sprot.fasta.pog,readonly \
    --mount=type=bind,source=/workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/uniprot_sprot.fasta.psd,target=/var/lib/cwl/stg9d3f538c-a076-4b8c-89bb-3596859ae6bb/uniprot_sprot.fasta.psd,readonly \
    --mount=type=bind,source=/workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/uniprot_sprot.fasta.psi,target=/var/lib/cwl/stg9d3f538c-a076-4b8c-89bb-3596859ae6bb/uniprot_sprot.fasta.psi,readonly \
    --mount=type=bind,source=/workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/uniprot_sprot.fasta.psq,target=/var/lib/cwl/stg9d3f538c-a076-4b8c-89bb-3596859ae6bb/uniprot_sprot.fasta.psq,readonly \
    --workdir=/SXZFEh \
    --read-only=true \
    --user=1000:1000 \
    --rm \
    --cidfile=/tmp/nzr4c1jz/20240528024151-894077.cid \
    --env=TMPDIR=/tmp \
    --env=HOME=/SXZFEh \
    biocontainers/blast:v2.2.31_cv2 \
    blastp \
    -query \
    /var/lib/cwl/stg3a2a6105-7709-4108-b72c-7736725ede96/MSTN.fasta \
    -db \
    /var/lib/cwl/stg9d3f538c-a076-4b8c-89bb-3596859ae6bb/uniprot_sprot.fasta \
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
INFO [job step1_blastp] completed success
INFO [step step1_blastp] completed success
INFO [workflow ] starting step step2_awk
INFO [step step2_awk] start
INFO [job step2_awk] /tmp/6z_470bu$ awk \
    '{ print $2 }' \
    /tmp/9dh7jo1d/stgf46698f9-3621-4a3c-8129-91340308984f/blastp_output.txt > /tmp/6z_470bu/blastp_result_id.txt
INFO [job step2_awk] completed success
INFO [step step2_awk] completed success
INFO [workflow ] starting step step3_blastdbcmd
INFO [step step3_blastdbcmd] start
INFO [job step3_blastdbcmd] /tmp/cvkfd42s$ docker \
    run \
    -i \
    --mount=type=bind,source=/tmp/cvkfd42s,target=/SXZFEh \
    --mount=type=bind,source=/tmp/t2ucva7i,target=/tmp \
    --mount=type=bind,source=/workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/uniprot_sprot.fasta,target=/var/lib/cwl/stg7e12378f-4095-4e60-b1a8-fb647ee48f3c/uniprot_sprot.fasta,readonly \
    --mount=type=bind,source=/workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/uniprot_sprot.fasta.phd,target=/var/lib/cwl/stg7e12378f-4095-4e60-b1a8-fb647ee48f3c/uniprot_sprot.fasta.phd,readonly \
    --mount=type=bind,source=/workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/uniprot_sprot.fasta.phi,target=/var/lib/cwl/stg7e12378f-4095-4e60-b1a8-fb647ee48f3c/uniprot_sprot.fasta.phi,readonly \
    --mount=type=bind,source=/workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/uniprot_sprot.fasta.phr,target=/var/lib/cwl/stg7e12378f-4095-4e60-b1a8-fb647ee48f3c/uniprot_sprot.fasta.phr,readonly \
    --mount=type=bind,source=/workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/uniprot_sprot.fasta.pin,target=/var/lib/cwl/stg7e12378f-4095-4e60-b1a8-fb647ee48f3c/uniprot_sprot.fasta.pin,readonly \
    --mount=type=bind,source=/workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/uniprot_sprot.fasta.pog,target=/var/lib/cwl/stg7e12378f-4095-4e60-b1a8-fb647ee48f3c/uniprot_sprot.fasta.pog,readonly \
    --mount=type=bind,source=/workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/uniprot_sprot.fasta.psd,target=/var/lib/cwl/stg7e12378f-4095-4e60-b1a8-fb647ee48f3c/uniprot_sprot.fasta.psd,readonly \
    --mount=type=bind,source=/workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/uniprot_sprot.fasta.psi,target=/var/lib/cwl/stg7e12378f-4095-4e60-b1a8-fb647ee48f3c/uniprot_sprot.fasta.psi,readonly \
    --mount=type=bind,source=/workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/uniprot_sprot.fasta.psq,target=/var/lib/cwl/stg7e12378f-4095-4e60-b1a8-fb647ee48f3c/uniprot_sprot.fasta.psq,readonly \
    --mount=type=bind,source=/tmp/6z_470bu/blastp_result_id.txt,target=/var/lib/cwl/stg9eb1c473-3851-4d12-ae06-17d0ab07f41d/blastp_result_id.txt,readonly \
    --workdir=/SXZFEh \
    --read-only=true \
    --user=1000:1000 \
    --rm \
    --cidfile=/tmp/y6vi93xy/20240528024152-921898.cid \
    --env=TMPDIR=/tmp \
    --env=HOME=/SXZFEh \
    biocontainers/blast:v2.2.31_cv2 \
    blastdbcmd \
    -db \
    /var/lib/cwl/stg7e12378f-4095-4e60-b1a8-fb647ee48f3c/uniprot_sprot.fasta \
    -entry_batch \
    /var/lib/cwl/stg9eb1c473-3851-4d12-ae06-17d0ab07f41d/blastp_result_id.txt \
    -out \
    blastdbcmd_result.fasta
WARNING: The requested image's platform (linux/amd64) does not match the detected host platform (linux/arm64/v8) and no specific platform was requested
INFO [job step3_blastdbcmd] completed success
INFO [step step3_blastdbcmd] completed success
INFO [workflow ] starting step step4_clustalo
INFO [step step4_clustalo] start
INFO [job step4_clustalo] /tmp/vo2athjh$ docker \
    run \
    -i \
    --mount=type=bind,source=/tmp/vo2athjh,target=/SXZFEh \
    --mount=type=bind,source=/tmp/qdbc882d,target=/tmp \
    --mount=type=bind,source=/tmp/cvkfd42s/blastdbcmd_result.fasta,target=/var/lib/cwl/stg6c1198d1-d950-4179-aaf2-f060b99cb1d6/blastdbcmd_result.fasta,readonly \
    --workdir=/SXZFEh \
    --read-only=true \
    --user=1000:1000 \
    --rm \
    --cidfile=/tmp/k17yd02h/20240528024153-956824.cid \
    --env=TMPDIR=/tmp \
    --env=HOME=/SXZFEh \
    biocontainers/clustalo:v1.2.4-2-deb_cv1 \
    clustalo \
    -i \
    /var/lib/cwl/stg6c1198d1-d950-4179-aaf2-f060b99cb1d6/blastdbcmd_result.fasta \
    --outfmt=fasta \
    -o \
    clustalo_output.fasta
WARNING: The requested image's platform (linux/amd64) does not match the detected host platform (linux/arm64/v8) and no specific platform was requested
INFO [job step4_clustalo] completed success
INFO [step step4_clustalo] completed success
INFO [workflow ] starting step step5_fasttree
INFO [step step5_fasttree] start
INFO [job step5_fasttree] /tmp/o9azchvf$ docker \
    run \
    -i \
    --mount=type=bind,source=/tmp/o9azchvf,target=/SXZFEh \
    --mount=type=bind,source=/tmp/c40hkkod,target=/tmp \
    --mount=type=bind,source=/tmp/vo2athjh/clustalo_output.fasta,target=/var/lib/cwl/stgb5fba895-0d32-4648-bea2-fb41b2489c80/clustalo_output.fasta,readonly \
    --workdir=/SXZFEh \
    --read-only=true \
    --log-driver=none \
    --user=1000:1000 \
    --rm \
    --cidfile=/tmp/yj95fyeq/20240528024155-004697.cid \
    --env=TMPDIR=/tmp \
    --env=HOME=/SXZFEh \
    biocontainers/fasttree:v2.1.10-2-deb_cv1 \
    fasttree \
    -nt \
    /var/lib/cwl/stgb5fba895-0d32-4648-bea2-fb41b2489c80/clustalo_output.fasta > /tmp/o9azchvf/fasttree_result.nwk
WARNING: The requested image's platform (linux/amd64) does not match the detected host platform (linux/arm64/v8) and no specific platform was requested
FastTree Version 2.1.10 Double precision (No SSE3)
Alignment: /var/lib/cwl/stgb5fba895-0d32-4648-bea2-fb41b2489c80/clustalo_output.fasta
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
    "final_output_bootstrap": {
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

無事､出力されています｡このように､ワークフローを記述することで､複数の処理を一度に実行することができます｡

&nbsp;

:::message `doc`フィールドを積極的に書こう

複雑なプロセスを書いたあと､見返した際に｢あれ､このプロセスはなんだっけ...｣となってしまうことは容易に想像できます｡
そこで､cwlファイルを書く際には､`doc`フィールドを積極的に書くことをおすすめします｡

例えば､先程作成した`Workflow`定義のcwlファイルに対して`--help`オプションをつけて実行してみます｡

```bash:
cwltool blast2tree_v2.cwl --help
INFO /usr/local/bin/cwltool 3.1.20240508115724
INFO Resolved 'blast2tree_v2.cwl' to 'file:///workspaces/togotv_cwl_for_remote_container/zatsu_cwl_bioinformatics/blast2tree_v2.cwl'
usage: blast2tree_v2.cwl [-h] [--1_protein_query 1_PROTEIN_QUERY] [--2_protein_database 2_PROTEIN_DATABASE] [--3_evalue 3_EVALUE] [--4_number_of_threads 4_NUMBER_OF_THREADS] [--5_outformat_type 5_OUTFORMAT_TYPE]
                         [--6_output_file_name 6_OUTPUT_FILE_NAME] [--7_max_target_sequence 7_MAX_TARGET_SEQUENCE] [--8_blastdbcmd_protein_database 8_BLASTDBCMD_PROTEIN_DATABASE] [--10_clustalo_output_name 10_CLUSTALO_OUTPUT_NAME]
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
zatsu-cwl-generatorを使うことで､簡単にcwlファイルを生成することで､CWLを記述するハードルを下げることができます｡
この記事で説明しているプロセスとしては､まとめると以下のようになっています｡

1. zatsu-cwl-generatorを使って､コマンドラインツールの処理をcwlファイルとして生成する
2. cwltoolの `--validate`オプションを使って､cwlファイルをチェックする
3. 問題がある場合､修正を行う
4. cwltoolを使って､CWLファイルを実行する
5. エラーが出た場合､修正を行う

このように､ __zatsu-cwl-generator__ をcwlファイルを書くプロセスの起点として利用することが可能です｡
ぜひ､皆さんも活用してください!

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

[^1]: [バイオインフォマティクスのツールを再現性よく実行するためのコンテナ仮想化ツール群 BioContainers](https://kazumaxneo.hatenablog.com/entry/2018/10/02/112232)