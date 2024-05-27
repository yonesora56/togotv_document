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

## zatsu-cwl-generatorを使ってcwlファイルを書く

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

:::details 修正後のCWLファイル
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
これでCWLの修正は完了ですが､更に改善することもできます｡

:::message 

上記のcwlファイルの例では､E-valueの指定を`Any`にしていましたが､具体的な型を指定することで､より正確な入力を受け付けることができます｡

https://github.com/yonesora56/togotv_cwl_for_remote_container/blob/master/zatsu_cwl_bioinformatics/1_blastp_docker_v3.cwl#L41-L43

`1e-05`のように指数表記で入力するため､`float?`のように記載してみました｡

https://github.com/yonesora56/togotv_cwl_for_remote_container/blob/master/zatsu_cwl_bioinformatics/1_blastp_docker_v4.cwl#L41-L43

これにより､`Expecting one of: ['Directory', 'File', 'boolean', 'double', 'float', 'int', 'long', 'null', 'stderr', 'stdout', 'string']` とハイライトされていた部分が解消されました｡
:::

****

## 他のファイルもcwlファイルとして書いてみる

blastpのプロセスのように､他のプロセスもCWLファイルとして書いてみます｡
zatsu-cwl-generatorを使って生成し､修正したバージョンは`v2`などのようにファイル名を変更しています｡
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

```bash
zatsu-cwl-generator "awk '{ print $2 }' blastp_result.txt > blastp_result_id.txt" > 2_awk.cwl
```

色々試行錯誤した結果､このファイルで実行できました｡
https://github.com/yonesora56/togotv_cwl_for_remote_container/blob/master/zatsu_cwl_bioinformatics/2_awk_v2.cwl

実行したファイルは以下の通りです｡

https://github.com/yonesora56/togotv_cwl_for_remote_container/blob/master/zatsu_cwl_bioinformatics/blastp_result_id.txt
:::

:::details 3_blastdbcmd_v2.cwl

:::


&nbsp;

&nbsp;

&nbsp;

&nbsp;

&nbsp;

&nbsp;

&nbsp;
  
&nbsp;

### パラメータをYAMLファイルでまとめて与える

パラメータを一つのYAMLファイルにまとめて実行することができます｡
ファイルは以下のように書くことが可能です｡

```yaml:
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

&nbsp;

&nbsp;

&nbsp;

&nbsp;

## ワークフローを記述する

:::danger
__修正：best practicesに載っているような書き方で最後書く__
:::

これまで､CommanlinetoolのCWLファイルを書いてみました｡次にこれらの5つのステップを実行するワークフローを記述していきます｡この例では､ワークフロー全体に関するパラメータを｢1\_protein\_query｣のように数字をつけています｡ 前の処理のアウトプットを受け取る部分は｢blastp\_result: step1\_blastp/blastp\_output\_file｣のように記述し､それ以外の全ての処理に関するパラメータをinputsで記述しています｡
```yaml:
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

```yaml:
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
```yaml:
cwltool --validate ./workflow/6_phylogenetic-workflow.cwl ./workflow/input.yaml
INFO /usr/local/bin/cwltool 3.1.20230906142556
INFO Resolved './workflow/6_phylogenetic-workflow.cwl' to 'file:///workspaces/togotv_shooting/workflow/6_phylogenetic-workflow.cwl'
./workflow/6_phylogenetic-workflow.cwl is valid CWL.
```

書き方としては問題が無いようです｡
続いて､CWLファイルに対して`--help`オプションを行ってみます｡

```bash:
cwltool ./workflow/6_phylogenetic-workflow.cwl --help
```

:::danger
__追記：ここでも先頭何行か見せて､docフィールドを書くことのご利益が得られることを書く__
:::


それでは､実際に実行してみます｡

```yaml:
cwl-runner ./workflow/6_phylogenetic-workflow.cwl ./workflow/input.yml
```

実行結果
```bash:

```

解析が成功しているようです｡ 実際に､newick形式のファイルも出力されています｡ このように､何回もコマンドを実行せず､1つのコマンドで実行することができます｡
__CWLで記述したファイルは様々な環境で実行することができますので､ぜひご自身でクローンして確かめてください!__

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