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
__修正1：まずdockerなしの状態で記述した後､dockerについて記述する__
__修正2：`arguments`ではなく､`hints`フィールドを書く(条件がゆるい)__
__修正3：dockerの説明のあとにsingularityの説明を加える__
__変更：`-c`オプションを使ってコンテナの場合を紹介する__
__変更：出力された結果を__
:::

# バイオインフォマティクスの解析手順をワークフロー化する

これまで､環境構築､`grep`と`wc`の処理に関するワークフローを作る作業を紹介しました｡
ここでは､具体的にバイオインフォマティクス研究で使用されているツールを使った例を __ワークフロー__ として記述してみる例を紹介します｡
今回は以下に示す5つのステップをワークフローとして記述してみます｡

**(1)blastpコマンドによる配列類似性検索**
**(2)awkによるヒットしたIDの抽出**
**(3)blastdbcmdによるヒットしたIDのfastaファイルの抽出**
**(4)ClustalOmegaによるマルチプルアラインメント**
**(5)fasttreeによるnewick形式ファイルの出力**

これにより､BLASTpでヒットした配列から､系統樹を作成する前段階の処理(newick形式のファイルを出力)をワークフローとして記述します｡
これらのコマンドは､Biocondaやhomebrewなどからインストールして実行することが可能ですが､今回は､dockerを使ったコマンドのCWLファイル作成を行います｡

:::message
この記事でも後述していますが､__docker imageがある場合にはそのimageを使用して実行する__ CWLファイルを書くので､すでにツールをインストールしている場合でも試すことが可能です｡
:::

&nbsp;

&nbsp;

## docker imageを取得する

:::message
すでに構築した環境にBLASTなどのツールをインストールしている方はこの手順は飛ばしてください｡
:::

今回取り上げるツールのdocker imageを[docker hub](https://hub.docker.com/)から取得します｡
[blast](https://hub.docker.com/r/biocontainers/blast)､[clustalo](https://hub.docker.com/r/biocontainers/clustalo)､[fasttree](https://hub.docker.com/r/biocontainers/fasttree)はすべて[BioContainers](https://biocontainers.pro/)によって構築されています[^1]｡
また､awkについては[ubuntu](https://hub.docker.com/_/ubuntu)のimageを取得しました｡

:::message
今回は､tag を指定する形でダウンロード(docker pull ... をコピー&ペースト)しました｡
:::

以下はターミナルで `docker image ls` を行った例です｡
```bash:
docker image ls
REPOSITORY               TAG                 IMAGE ID       CREATED       SIZE
ubuntu                   23.10               78938b354ee7   2 weeks ago   72.1MB
biocontainers/fasttree   v2.1.10-2-deb_cv1   bdfde5d6026c   4 years ago   118MB
biocontainers/clustalo   v1.2.4-2-deb_cv1    d14a5e27e301   4 years ago   118MB
biocontainers/blast      v2.2.31_cv2         5b25e08b9871   4 years ago   2.03GB
```

&nbsp;

&nbsp;

## ワークフローの準備を行う

次に､インプットするファイルを準備していきます｡

### 1\. クエリ配列

blastpを実行するので､今回クエリとするのは､ウシのミオスタチンのタンパク質配列のfastaファイル (MSTN.fastaに名前を変更しています)です｡ 
curlコマンドでUniProtから取得しました｡
```bash:
curl -O https://rest.uniprot.org/uniprotkb/O18836.fasta
```

### 2\. インデックス(データベース)

次に､BLASTのデータベースとして使用するUniProtのタンパク質配列のfastaファイル(uniprot\_sprot.fasta.gz)をftpサイトよりcurlコマンドで取得しました｡ 

```bash:
curl -O https://ftp.uniprot.org/pub/databases/uniprot/current_release/knowledgebase/complete/ uniprot_sprot.fasta.gz  
``` 

このファイルをインデックスにするため､ `makeblastdb` コマンドを実行しました｡ 
`makeblastdb` コマンド の実行は以下のように行いました｡ なお､解析するデータは全てdataディレクトリにおいていますが､UniProtのタンパク質配列のfastaファイルはサイズが大きいのでgitiignoreに記述しています｡
```bash:
docker run --rm -it -v `pwd`:`pwd` -w `pwd` biocontainers/blast:v2.2.31_cv2 makeblastdb -in uniprot_sprot.fasta -dbtype prot -hash_index -parse_seqids 
```

https://github.com/yonezawa-sora/togotv_cwl_for_remote_container/tree/master/data

&nbsp;

## zatsu-cwl-generatorを使ってcwlファイルを書く

次に､実際の処理について､CWLファイルを記述していきましょう｡ 今回もzatsu-cwl-generatorを使って書いていきます｡
今回実行する5つのステップのコマンドは以下のようになっています｡ 

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

:::details Dockerコマンドで実行する場合(確認済み)
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

## コンテナオプション(`-c`)を使って出力する

それでは実際に作成していきます｡ 
今回はblastpのコマンドを例にするとともに､Dockerを使用するコマンドで出力する例を紹介したいと思います｡

zatsu-cwl-generatorでは､`-c`オプションを使うことで､コンテナを使用するコマンドを出力することができます｡
以下のように実行します｡

```bash
zatsu-cwl-generator "blastp -query MSTN.fasta -db uniprot_sprot.fasta -evalue 1e-5 -num_threads 4 -outfmt 6 -out blastp_result.txt -max_target_seqs 20" -c biocontainers/blast:v2.2.31_cv2
```

これまでと同じようにコマンドを""でくくったあとに､`-c` もしくは`--container` オプションをつけて､imageを記載します｡ すると､以下のようにCWLファイルが出力されます｡

:::details コンテナオプションを使って出力した場合のCWLファイル
```yaml
#!/usr/bin/env cwl-runner
# Generated from: blastp -query MSTN.fasta -db uniprot_sprot.fasta -evalue 1e-5 -num_threads 4 -outfmt 6 -out blastp_result.txt -max_target_seqs 20
class: CommandLineTool
cwlVersion: v1.0
baseCommand: blastp
arguments:
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
      location: blastp_result.txt
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
hints:
  - class: DockerRequirement
    dockerPull: biocontainers/blast:v2.2.31_cv2
```

実は､下に`hints`フィールドというものが出力されています｡

```yaml
hints:
  - class: DockerRequirement
    dockerPull: biocontainers/blast:v2.2.31_cv2
```



&nbsp;

```yaml
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

```yaml
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

```yaml:
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

```bash:
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

```yaml:
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

```yaml:
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

```yaml:
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

```yaml:
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

# 参考リンク集

[^1]: [バイオインフォマティクスのツールを再現性よく実行するためのコンテナ仮想化ツール群 BioContainers](https://kazumaxneo.hatenablog.com/entry/2018/10/02/112232)