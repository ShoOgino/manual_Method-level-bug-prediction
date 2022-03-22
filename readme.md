# メソッド粒度バグ予測_実験方法
## 手順書
### 1. 対象プロジェクトのGitリポジトリの粒度をファイルからメソッドへ変換
- [git-stein](https://github.com/sh5i/git-stein)を利用。



### 2. 対象プロジェクトについて、バグ修正コミットを特定
1. バグレポート一覧ファイルを取得。
    - 例: eclipseのプロジェクトであるegitを対象とする場合
        - 下記のコマンドを実行する
            - python [fetch_bugzilla.py](https://github.com/ShoOgino/szz/blob/master/script/fetch_bugzilla.py) --project egit" 
                - 実行時のディレクトリ直下に"[reports.json](2/reports.json)"が出力される。
2. コミット一覧ファイルを取得
    - 下記のコマンドを実行する
        - python3 [git_log_to_array.py](https://github.com/ShoOgino/szz/blob/master/script/git_log_to_array.py) --pathRepository method --commitFrom ${コミットID（このコミット以前のコミットを参照する）}
            - 実行時のディレクトリ直下に"[commits.json](2/commits.json)"が出力される。
3. バグ修正コミット（バグレポートIDがコミットメッセージに記述されているコミット）を特定
    - 下記のコマンドを実行する
        - python3 [find_bug_fixes.py](https://github.com/ShoOgino/szz/blob/master/script/find_bug_fixes.py) --pathCommits commits.json --pathReports reports.json
            - 実行時のディレクトリ直下に"[bugfixes.json](2/bugfixes.json)"(バグ修正コミット一覧)が出力される。



### 3. バグ混入コミットを特定
- [SZZアルゴリズムの実装](https://github.com/ShoOgino/szz/tree/master/tools/szz)を用いる。
    - 下記のコマンドを実行する
        - java -jar [szz_find_bug_introducers-0.1.jar](https://github.com/ShoOgino/szz/tree/master/tools/szz/build/libs/szz_find_bug_introducers-0.1.jar) -i bugfixes.json -r ${メソッド粒度のリポジトリフォルダ}
            - 実行時のディレクトリ直下に"results/annotations.json"(バグ混入コミット一覧)が出力される
        - python reformat.py --pathAnnotation annotations.json
            - 実行時のディレクトリ直下に"[bugs.json](3/bugs.json)"(バグ混入コミット一覧(フォーマット済))が出力される


### 4. 学習用・評価用データセットを構築
1. [AnalyzeRepository](https://github.com/ShoOgino/RepositoryAnalyzer_private)をgit clone。
2. AnalyzeRepositoryの実行可能jarを`./gradlew shadowJar`でビルド。
    - [楠本研サーバthor](https://github.com/kusumotolab/sdllog/blob/master/articles/%E3%82%B5%E3%83%BC%E3%83%90.md)(openjdk version "1.8.0_292")で動作確認済み。
3. 実験対象プロジェクトについてフォルダを作り、ディレクトリ構成を下記の通りにする。
    - ${実験対象プロジェクトについてのフォルダ}
        - repositoryMethod(メソッド粒度のリポジトリフォルダ)
        - repositoryFile(ファイル粒度のリポジトリフォルダ)
        - bugs.json
4. データセット算出タスクを定義したjsonファイルを${実験対象プロジェクトについてのフォルダ}直下に用意。
    - ${データセット算出タスク}.jsonの構造は下記の通り。
        - isMultiProcess: 並列処理をするかどうか。
        - tasks: タスク集合。
            - (task): データセット算出タスク。下記のプロパティで規定される。
                - name: String。タスクの名前。一応命名規則は${プロジェクト名}_${参照区間}_${trainまたはtest}（例: egit_R2_train)。
                - priority: int。タスクの優先度。0でOK。
                - pathProject: String。実験対象プロジェクトについてのフォルダのパス。
                - granularity: String。予測粒度。"method"でok。
                - product: List<String>。メソッドについて、なんのデータを算出するか。複数選択可
                    - 選択肢
                        - metricsProcessGiger: Gigerらが提案したプロセスメトリクスセット
                        - metricsProcessMing: Mingらが提案したプロセスメトリクスセット
                        - metricsCodeGiger: Gigerらが提案したコードメトリクスセット
                        - metricsCodeMing: Mingらが提案したコードメトリクスセット
                        - graphCommit: 個々のコミットについてのメトリクス
                - revisionTargetMethod: String。method粒度リポジトリのどの時点で存在するメソッドについて、レコードを算出するか。
                - intervalRevisionMethod_referableCalculatingMetricsIndependentOnFuture: List<String>。未来の情報に依存しないメトリクス(LOCとかnumOfCommittersとか)を算出する際に参照可能な、開発履歴の区間。method粒度リポジトリについて。
                    - intervalRevisionMethod_referableCalculatingMetricsIndependentOnFuture[0]: コミットID。参照区間の始まり。
                    - intervalRevisionMethod_referableCalculatingMetricsIndependentOnFuture[1]: コミットID。参照区間の終わり。
                - intervalRevisionMethod_referableCalculatingMetricsDependentOnFuture: List<String>。未来の情報に依存するメトリクス(isBuggyとか)を算出する際に参照可能な、開発履歴の区間。method粒度リポジトリについて。
                    - intervalRevisionMethod_referableCalculatingMetricsDependentOnFuture[0]: コミットID。参照区間の始まり。
                    - intervalRevisionMethod_referableCalculatingMetricsDependentOnFuture[1]: コミットID。参照区間の終わり。
                - revisionTargetFile": revisionTargetMethodのファイル粒度リポジトリ版。
                - intervalRevisionFile_referableCalculatingMetricsIndependentOnFuture: intervalRevisionMethod_referableCalculatingMetricsIndependentOnFutureのファイル粒度リポジトリ版
                - intervalRevisionFile_referableCalculatingMetricsDependentOnFuture: intervalRevisionMethod_referableCalculatingMetricsDependentOnFutureのファイル粒度リポジトリ版。
    - ${データセット算出タスク}.jsonの例: [egit.json](egit.json)
5. java -jar ${ビルドされたjar} ${データセット算出タスク}.json を実行。
    - 実行後のディレクトリ構成は次のようになる。
        - ${実験対象プロジェクトについてのフォルダ}
            - repositoryMethod(メソッド粒度のリポジトリフォルダ)
            - repositoryFile(ファイル粒度のリポジトリフォルダ)
            - commits
            - modules
            - output
                - ${タスクの名前}: モジュールの分析結果が、モジュールごとにjson形式で収められている
                - ${タスクの名前}.csv: 各モジュールに対するメトリクスが、csv形式で収められている
            - bugs.json
            - ${データセット算出タスク}.json

### 5. 学習用・評価用データセットに基づいて、バグ予測モデルを構築＋評価
- [GreedyBugPrediction](https://github.com/ShoOgino/MLTool)を用いる。
    - 実行環境について
        - コンテナイメージをファイル(fenrir:/home/s-ogino/bug_prediction.tar)から読み込む。
        - コンテナ内で conda activate mltool3.8 を実行する。
    - RQ2.py, RQ3.pyに記述された下記の設定を書き換えて実行する。
        - def main()内の dirDatasets: 
        - def main()内の namesProject: 
        - def init()内の setPeriod4HyperParameterSearch: ハイパーパラメータチューニングに費やす時間。秒単位。



## 実験データ等
### リポジトリ一覧
- cassandra
    - [cassandraFile](https://github.com/apache/cassandra.git)
    - [cassandraMethod](https://github.com/ShoOgino/cassandraMethod202104.git)
- egit
    - [egitFile](https://github.com/eclipse/egit.git)
    - [egitMethod](https://github.com/ShoOgino/egitMethod202104.git)
- jgit
    - [jgitFile](https://github.com/eclipse/jgit.git)
    - [jgitMethod](https://github.com/ShoOgino/jgitMethod202104.git)
- linuxtools
    - [linuxtoolsFile](https://github.com/eclipse/linuxtools.git)
    - [linuxtoolsMethod](https://github.com/ShoOgino/linuxtoolsMethod202104.git)
- realm-java
    - [realm-javaFile](https://github.com/realm/realm-java)
    - [realm-javaMethod](https://github.com/ShoOgino/realm-javaMethod)
- sonar-java
    - [sonar-javaFile](https://github.com/SonarSource/sonar-java)
    - [sonar-javaMethod](https://github.com/ShoOgino/sonar-javaMethod)
- poi
    - [poiFile](https://github.com/apache/poi.git)
    - [poiMethod](https://github.com/ShoOgino/poiMethod202104.git)
- wicket
    - [wicketFile](https://github.com/apache/wicket.git)
    - [wicketMethod](https://github.com/ShoOgino/wicketMethod202104.git)
### リリースコミット一覧
- cassandra(1, 2, 3, 4) 11ヶ年＋224
    - 最古(root): 2009/05/02 07:57:22
        - file: 1f91e99223b0d1b7ed8390400d4a06ac08e4aa85
        - method: 56157fc6d98b9a48cfbd2977d8eb757a508ddd61
    - リリース1.0.0(1): 2011/10/19 13:35:57, +5252 +2/5/17
        - file: 845af1ca3ce21fb543daba5434751cb63fd5a18d
        - method: 5c6afc827e5e365b8db887d5d6375bf28f3617c4
    - リリース2.0.0(2): 2013/09/03 16:38:09, +10283
        - file: 5ea3b153840d069b50361f38b8b7050f3c09176b
        - method: 723f268a0d42a4d44f3e628db2e17e75e21b64d3
    - リリース3.0.0(3): 2015/11/06 14:39:23, +18851
        - file: 12b8c696bfd8ff9b63bd44dbed1fcfe95bc21aab
        - method: 1b723d6a3d5e77f229857e750af67f352a25c5ec
    - 最新(head): 
        - file: e848d47ed171f20ccd8cf5e20d9e188ede85c17c
        - method: 6d97484ca948055cd306e35b9b6e760e616cead8
- egit 11ヶ年＋6日
    - root: 2009/09/29 18:18:28
        - file: dfbdc456d8645fc0c310b5e15cf8d25d8ff7f84b
        - method: 2c1b0f4ad24fb082e5eb355e912519c21a5e3f41
    - 1: 2011/06/05 23:43:46 +1864 +2/3/24
        - file: 0cc8d32aff8ce91f71d2cdac8f3e362aff747ae7
        - method: 1241472396d11fe0e7b31c6faf82d04d39f965a6
    - 2: 2012/06/13 15:46:40 +2859 
        - file: 1f07f085d6bcae0caf372fffec19583ac5615d3b
        - method: 2774041935d41453e5080f0e3cbeef136a05597d
    - 3: 2013/06/11 00:46:23 +3407
        - file: f85dc228e3e67b229c978f751a8723a5c810738b
        - method: 6512ce982bf2376ae7ad308ba3b4a8fafe233376
    - 4: 2015/06/09 07:35:30 +4325
        - file: 48712bdfa849e1afd0071b937f0b2bef04767610
        - method: 5f42c83e73b0681da81e7466ae82830d927e916f
    - 5: 2018/06/13 22:05:45 +5475
        - file: ba0bcbfe69b2803c843c19b6e800b2525b227cb2
        - method: c6debf07f32e43470bd310f7b60bc19dd78f2506
    - head: 
        - file: 9904d5ba9d7d6b024a73999b5c9b19472a266205
        - method: b459d7381ea57e435bd9b71eb37a4cb4160e252b
- jgit
    - root: 2009/09/29 16:47:03
        - file: 1a6964c8274c50f0253db75f010d78ef0e739343
        - method: 004bd909df360ac2c699073636da678699853b19
    - 1: 2011/06/05 23:26:56 +1501 +1/8/6
        - file: f65513f75397b1f4de6d063962e5cccca5a89cb6
        - method: a2bc9ac8a1ea177ca25a8410d1daae80ce34e940
    - 2: 2012/06/13 15:03:59 +2053 +2/8/14
        - file: aacd4f721bba97eda3756e2fcbd71b5e57b3673a
        - method: 730797027641e02f9f05f2481fa3b9eddfa90f91
    - 3: 2013/06/11 00:28:11 +2686
        - file: f384644774ae01823e850c862aeda5bddb4a4326
        - method: 63c202af7960e18aafa54426d66ebb013d72461c
    - 4: 2015/06/09 07:29:27 +3804
        - file: 4f221854556991e3394b3a71e77ee0b771b1500b
        - method: 436e8b67ec6e56eba12d23ab223ed91ea9cf6cd2
    - 5: 2018/06/13 21:42:40 +5975
        - file: e729a83bd24bbc25f7ac209baee01f561fe218c8
        - method: 35b02784c066ad915030c9182b9ec38945133233
    - head:
        - file: 91b2e167a2341d41212ea801943fb82a148e3b7f
        - method: 1a2f9c40ff5df27517099ceb8239d2e53d5cea4d
- linuxtools 11ヶ年＋182日
    - root: 2009/04/09 19:54:15
        - file: b7218a084ed306a498bebdc313756dc706a49830
        - method: ad8b5bf026e3d12e499a3f5acf6ea6469261d377
    - 1: 2012/05/28 10:23:40 +5849 +3/1/19
        - file: 98e11be9f43ee4b4a574fac1437efc73c88878fd
        - method: 94ead44b7658adb9c528dc2fc3f1cb677669eb46
    - 2: 2013/06/10 16:50:52 +7610
        - file: 2e244caa1707bc82b7c487cc53ea9ebc4564c6fb
        - method: 3f53ed340e9f722fac8fcc710cac60cc404873b9
    - 3:  2014/06/09 16:15:09 +8975
        - file: 1b3ccef1dabb7360d891f0460bb1f8a981aefca8
        - method: 31933bba8ed3aab62d9dcc060d271e88522c73e6
    - 4: 2015/06/12 17:16:33 +9609
        - file: 6bfe7ea1397e70b4bea038b294ba6a61940354de
        - method: 1f5314024e098934c01b7a0615b08014c84adb24
    - 5: 2016/06/07 13:34:55 +10030
        - file: 4d86c7a081b74595b925ffa64c9578e36b0f7374
        - method: 284f156522dd1e00f07ee28f1fea7f400e2debec
    - 6: 2017/06/13 16:11:14 +10316
        - file: 2392872af4a958c41f3d817ae7c3bfe67d1a013a
        - method: d071eea3243834184b8dfb8fcc50ecf0aa29adfa
    - 7: 2018/07/04 17:27:00 +10495
        - file: 94bf679091944428c89919c52bb4cef8b2d92ca5
        - method: faaa55c05aae333d25e4bd26242e8229d17c3e2f
    - head: 
        - file: 3af94afe978305f38eed0c6d8eafd7653df0d01f
        - method: 9524114075dd38ceb913a610a4b56da01d6d1bb4
- realm-java
    - root:
        - file: b03c621431fdc7e6e43566eeef505a32f5f6ce83
    - 1: 
        - file: eef492341a17351dc671b296bf7f1cd2c2ed32a4
    - 2: 
        - file: 1f19a05f820c1e43a9a0e38d8e32b0d96920df7f
    - 3: 
        - file: 66fb375b32b7db660c1d06fc7e27bf708d8cebab
    - 4: 
        - file: e26255b9c5248620861565eb3caf7c908c4f8277
    - 5: 
        - file: 8740eb6ce5bfc8536be2b480988a6212f2ce8466
    - 6: 
        - file: 8a022573a5b095c2ec887720bd73098475829766
    - 7: 
        - file: 5e1cb707bf37a4c5fa03b45cd5a4fde39fed146d
    - head: 
        - file: fd3446d65856fc46bebf1e0632dca3b8260e2d12
- sonar-java
    - root:
        - file: 5393bd3cc71e716dcd946232f5d43297d5889357
    - 1: 
        - file: 5ac4cf695248bc7385cb3377216cd86340bda0b0
    - 2: 
        - file: fb4e10dcf17447ecea699934cdef0cf2009b1a52
    - 3: 
        - file: 65396a609ddface8b311a6a665aca92a7da694f1
    - 4: 
        - file: b653c6c8640ab3d6015d036a060f58e027a653af
    - 5: 
        - file: fe9584c9f7812edc46f32d29440fc81b85a597a4
    - 6: 
        - file: 931433c0510b161974d4844679f7bf3c73bb3e37
    - head: 
        - file: a4121b68b68740704a816ef7ad6636b61ab7ca55
- poi 18ヶ年＋254日
    - 1: 古すぎてgithubに記録されていない
    - root: 2002/01/31 02:22:28
        - file: 6c234ab6c8d9b392bc93f28edf13136b3c053894
        - method: 6c234ab6c8d9b392bc93f28edf13136b3c053894
    - 2: 2004/01/26 11:17:56 +1275 +1/11/26
        - file: 0ac1311fd0a411f6cce60e48b595844c360af466
        - method: 81a0a25199ccb15c5c2e4637d0f498b0300e5b3c
    - 3: 2007/05/7 17:09:31 +1838
        - file: 8ffd79b36803be29486ab6bdeff5dd02ea901927
        - method: cc1cd7f005bc2379449439427d2c0acef7cd932a
    - 4: 2018/08/31 12:09:06 +9396
        - file: 99b9b7828cb268f62daa5b0c5639e549477b2c6f
        - method: 37a74b4a8de772cde9788a6a8c2ae3c0862d31f3
    - head: 
        - file: 382714eccd92667fc83f70115b736c64ebff9700
        - method: 449f08fc0003c64d09663715f28896a4dd010d6a
- wicket(6, 7, 8) 16ヶ年＋20日
    - 1.0.0~1.5.0はスキップすべき。
        - セマンティックバージョニングで管理されていないから。
    - 6: 2012/08/23 22:29:55 +16277 +...
        - file: 01b3c702bc33f53af45c60b8a8dc1c45a675ff2c
        - method: 80bb2dbc0f0f8b4f89ba463f9886495f23f79e9b
    - 7: 2015/07/10 09:18:39 +18820
        - file: cc6540441db12415d54c725769813248f6948098
        - method: f4058e9842bb6197f139581df19be2057935973d
    - 8: 2018/04/30 20:54:29 +20232
        - file: 167a14eef03bc1cc672135f551a091cf4fa999b5
        - method: 25905159ad2ecd911a37d56422d22160880a1066
    - head: 
        - file: 048e5681df960018722237ee6254273e839b5fd2
        - method: 2f1546f5f9ba8fc9fe71253675796f74fea5e953
### RQ2
#### Datasets
- fenrir:/home/s-ogino/MLTool/rq2/datasets
#### Hyperparameters
- fenrir:/home/s-ogino/MLTool/rq2/hyperparameters
#### Results
- fenrir:/home/s-ogino/MLTool/rq2/results
- 各プロジェクト・対象リリースについて10回ずつ実験したときの結果。
- ConfusionMatrix.png: 予測結果の混同行列
- parameter: 予測時の、モデルのパラメータ
- prediction.csv: 予測結果(予測対象メソッド, groundtruth, 予測値)
- report.json: 予測精度について("1"がバグがある方を予測した場合のrecallなど)
- RQ2.py: 予測時のスクリプト
### RQ3 
#### Datasets
- fenrir:/home/s-ogino/MLTool/rq3/datasets
#### Hyperparameters
- fenrir:/home/s-ogino/MLTool/rq3/hyperparameters
#### Results
- fenrir:/home/s-ogino/MLTool/rq3/results
- 各プロジェクト・対象リリースについて5回ずつ実験したときの結果。
- ConfusionMatrix.png: 予測結果の混同行列
- parameter: 予測時の、モデルのパラメータ
- prediction.csv: 予測結果(予測対象メソッド, groundtruth, 予測値)
- report.json: 予測精度について("1"がバグがある方を予測した場合のrecallなど)
- RQ2.py: 予測時のスクリプト