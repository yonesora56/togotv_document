# togotv_document

Zennとの連携用リポジトリ

## 参考

- <https://zenn.dev/zenn/articles/install-zenn-cli>
- <https://zenn.dev/zenn/articles/markdown-guide>

## ドキュメント：CWLの作成環境をVScodeの｢Dev Containers｣の機能を使って構築する

## [CWL document 1](./articles/cwl-document_1.md)

### 20231117
- Codespacesから直にリポジトリを開く例を追加して完成
- 脚注を追加

### 20240612

- 脚注を削除

&nbsp;

## [CWL document 2](./articles/cwl-document_2.md)

- workflowの実行例の説明をもう少し追加して完成
- <https://view.commonwl.org/workflows/github.com/manabuishii/cwl-samples/blob/bff3770d52404b12d2d0f12b45c4c0edd3213dba/grep-and-count-for-figure-1.cwl>

&nbsp;

## [CWL document 3](./articles/cwl-document_3.md)

- hintsフィールドとrequirementsフィールドの説明を追加する
- 脚注を追加予定

&nbsp;

## 丹生さんへのリサーチ(20230928)

1. 全部ドキュメントにするのではなく､気になったらリンクがあることが重要!

2. 書いたファイルのリポジトリのリンクを最後と最初においておく
    - リンク集を最後に作成

3. __なぜCWLを扱うのかについて記述__
    - (記載を追加済み)

4. hintsフィールドとrequirementsフィールド(https://www.commonwl.org/v1.0/CommandLineTool.html#Requirements_and_hints)
5. CWLをなんのエンジンで実行するかで`requirements`フィールドの実行の可否が決まる
    - エンジンによってはサポートしていない事例も
    - バッジで準拠を確認できる(implementation)
6. argumentsにブチ込んで書くのがが楽っぽい
    - defaultはargumentsを書くほうが便利
    - __?がつくようなやつはargumentから消えるので､inputBindingをつけなきゃいけない__
7. shebangを書く? 書かない?
    - どっちでも大丈夫(シバンで書くと ./で実行できる)
    - ベストプラクティスにも書かれていない
    - zatsuをもっと活かそう
    - <https://www.commonwl.org/v1.2/CommandLineTool.html#Executing_CWL_documents_as_scripts>
8. どのツールで実行する?(cwltool? runner? toil-runner?)
    - Javaコマンド(OpenJDK,AzureJDK)を提供している
    - C-コンパイラ
    - なんでも実行したいならcwl-runner
    - チュートリアルなので､触れなくても良いかも
9.  zatsu-cwl-generator --help を書く
    - Javaコマンド(OpenJDK,AzureJDK)を提供している
    - C-コンパイラ
    - なんでも実行したいならcwl-runner
    - (チュートリアルなので､触れなくても良いかも)

10.  __動画の広げ方について__
    - ~~GitHub pagesで公開しよう? (公開手法：ddbj/workflow-registoryのように)~~
      - Zennになりました
    - <https://ddbj.github.io/workflow-registry-browser/#/workflows/65bc3bd4-81d1-4f2a-8886-1fbe19011d81/versions/1.0.0>
    - いろんな環境で実行しよう

11. zatsu-cwl-generator
    - __動画でzatsu使って､記事ではもっと詳しく__
    - `-c`オプションメインでガンガン書いていく
    - biocontainers（コンテナ）を使うケースはおまけにしておく
    - __ホストでガンガン入れている人向けに作るので､ローカルに入れているツールでCWLを書いて､その後にdockerの事例を紹介__
      - その予定
    - zatsu をもっと記事にいかす
12. Codespacesの違いを強調する
    - __クローンしなくても大丈夫な例を後ろに挿入__

13. bentenが表示されない?
    - issueに自分も書いておく
    - <https://github.com/rabix/benten/issues/122>
14. devcontainerの書き換え
    - お願いしました

15. cwlversionについて
    - どちらでもOK!



