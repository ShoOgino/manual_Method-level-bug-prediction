# メソッド粒度バグ予測手順書
## 概要
1. リポジトリをファイル粒度からメソッド粒度へ変換
2. bugfixコミット・bugintroコミットを特定(手順4のため)
3. メソッドについて各種メトリクスを算出。
4. メトリクスを整形して、データセットを構築する。
5. データセットをもとに、バグ予測モデルを構築＋評価

リネーム追跡が関わる操作は3だけ。だから、章良は3で利用するツールを書き換えてコンパイル後することになる。3以外の操作は手順に沿うだけで良いはず。

## 詳細
### 1. リポジトリをファイル粒度からメソッド粒度へ変換
- リネーム追跡と関係ないから、俺が変換したやつを使えばOK。
    - 本ファイル下部の「対象プロジェクトのリポジトリ一覧」を参照。


### 2. bugfixコミット・bugintroコミットを算出
- リネーム追跡と関係ないから、俺が算出したやつを使えばok。
    - https://github.com/ShoOgino/bugs.git


### 3. メソッドについて各種メトリクスを算出（データセットを作る）
1. [AnalyzeRepository](https://github.com/ShoOgino/AnalyzeRepository.git)をgit clone。
2. プロジェクトフォルダを作り、ディレクトリ構成を下記の通りにする。
    - ${プロジェクトフォルダ}
        - repositoryMethod(メソッド粒度のリポジトリフォルダ)
        - repositoryFile(ファイル粒度のリポジトリフォルダ)
        - bugs.json
3. AnalyzeRepositoryのリネーム追跡部分を書き換え。
4. AnalyzeRepositoryの実行可能jarを`./gradlew shadowJar`でビルド。
    - [楠本研サーバthor](https://github.com/kusumotolab/sdllog/blob/master/articles/%E3%82%B5%E3%83%BC%E3%83%90.md)(openjdk version "1.8.0_292")で動作確認済み。
5. java -jar ${ビルドされたjar}<br>
    --pathProject ${プロジェクトフォルダのパス}<br>
    --idCommitHead ${headコミットid(メソッド粒度の方)}<br>
    --commitEdgesMethod ${コミットid(メソッド粒度の方)。対象期間の始端を表す} ${コミットid(メソッド粒度の方)。対象期間の終端を表す} ${コミットid。このコミットまでに実行されたbugfixコミットを参照する(メソッド粒度の方)}<br>
    --commitEdgesFile ${コミットid(ファイル粒度の方)。対象期間の始端を表す} ${コミットid(ファイル粒度の方)。対象期間の終端を表す}<br>
    --calcMetrics<br>
    を実行。
    - 俺の研究と同様の予測(release-by-release方式の予測)をする場合
        - リリースコミットorルートコミットから次のリリースコミットまでを対象期間とする。
            - 対象プロジェクトのリリースコミットについては本ファイル下部の各種コミット一覧を参照。
        - 対象期間AについてデータセットAを構築し、対象期間BについてデータセットBを構築し、データセットAをモデル訓練用、データセットBをテスト用に用いる。
            - 対象期間Aは対象期間Bより過去の期間とする（駄目な例:対象期間A＝R1～R2, 対象期間B＝root～R1）。
        - コマンド例: egitについて、root～R1を学習用、R1～R2をテスト用とする時
            - 学習用データセットを算出
                - java -jar ${ビルドされたjar}<br>
                --pathProject ${プロジェクトフォルダのパス}<br>
                --idCommitHead b459d7381ea57e435bd9b71eb37a4cb4160e252b<br>
                --commitEdgesMethod 2c1b0f4ad24fb082e5eb355e912519c21a5e3f41 1241472396d11fe0e7b31c6faf82d04d39f965a6<br>
                --commitEdgesFile dfbdc456d8645fc0c310b5e15cf8d25d8ff7f84b 0cc8d32aff8ce91f71d2cdac8f3e362aff747ae7<br>
                --calcMetrics<br>
            - テスト用データセットを算出
                - java -jar ${ビルドされたjar}<br>
                --pathProject ${プロジェクトフォルダのパス}<br>
                --idCommitHead b459d7381ea57e435bd9b71eb37a4cb4160e252b<br>
                --commitEdgesMethod 1241472396d11fe0e7b31c6faf82d04d39f965a6 2774041935d41453e5080f0e3cbeef136a05597d<br>
                --commitEdgesFile 0cc8d32aff8ce91f71d2cdac8f3e362aff747ae7 1f07f085d6bcae0caf372fffec19583ac5615d3b<br>
                --calcMetrics<br>
    - 結果として、下記のようなディレクトリ構成になっているはず。
        - ${プロジェクトフォルダ}
            - repositoryMethod(メソッド粒度のリポジトリフォルダ)
            - repositoryFile(ファイル粒度のリポジトリフォルダ)
            - datasets
                - ${対象期間のメソッドについてのデータセット}.csv
            - bugs.json

### 4. データセット前処理
[MLTool](https://github.com/ShoOgino/MLTool.git)を用いる。
#### 実行手順
1. [MLTool](https://github.com/ShoOgino/MLTool.git)をgit clone。
2. python MLTool/actions/src/splitDataset.py <br>
--dataset4train ${訓練用データセット.csvのパス} <br>
--dataset4test ${テスト用データセット.csvのパス} <br>
--destination ${前処理の終わったデータセットを保存するディレクトリ}<br>
を実行。
    - 結果として、${前処理の終わったデータセットを保存するディレクトリ}に下記のファイルが生成される。
        - train0.json
        - train1.json
        - train2.json
        - train3.json
        - train4.json
        - valid0.json
        - valid1.json
        - valid2.json
        - valid3.json
        - valid4.json
        - test.json

### 5. データセットをもとに、バグ予測モデルを構築＋評価
- [MLTool](https://github.com/ShoOgino/MLTool)を用いる。MLToolのReadmeを参照のこと…
    - 実行環境は[楠本研サーバthor](https://github.com/kusumotolab/sdllog/blob/master/articles/%E3%82%B5%E3%83%BC%E3%83%90.md)上のdockerコンテナ(id: 578a599c9b0b, name: mltool)上に構築してある。`sudo docker exec -it mltool ash`で中に入れる。
    - ハイパーパラメータ探索に費やす時間は1時間くらいでいい。

## 対象プロジェクトのリポジトリ一覧
- cassandra
    - [cassandraFile](https://github.com/apache/cassandra.git)
    - [cassandraMethod](https://github.com/ShoOgino/cassandraMethod202104.git)
- eclipse.jdt.core
    - [eclipse.jdt.coreFile](https://github.com/eclipse/eclipse.jdt.core.git)
    - [eclipse.jdt.coreMethod](https://github.com/ShoOgino/eclipse.jdt.coreMethod202104.git)
- egit
    - [egitFile](https://github.com/eclipse/egit.git)
    - [egitMethod](https://github.com/ShoOgino/egitMethod202104.git)
- jgit
    - [jgitFile](https://github.com/eclipse/jgit.git)
    - [jgitMethod](https://github.com/ShoOgino/jgitMethod202104.git)
- linuxtools
    - [linuxtoolsFile](https://github.com/eclipse/linuxtools.git)
    - [linuxtoolsMethod](https://github.com/ShoOgino/linuxtoolsMethod202104.git)
- lucene-solr
    - [lucene-solrFile](https://github.com/apache/lucene-solr.git)
    - [lucene-solrMethod](https://github.com/ShoOgino/lucene-solrMethod202104.git)
- poi
    - [poiFile](https://github.com/apache/poi.git)
    - [poiMethod](https://github.com/ShoOgino/poiMethod202104.git)
- wicket
    - [wicketFile](https://github.com/apache/wicket.git)
    - [wicketMethod](https://github.com/ShoOgino/wicketMethod202104.git)
## 各種コミット一覧
- cassandra(1, 2, 3, 4) 11ヶ年＋224
    - 最古(root): 2009/05/02 07:57:22
        - file: 1f91e99223b0d1b7ed8390400d4a06ac08e4aa85
        - method: 56157fc6d98b9a48cfbd2977d8eb757a508ddd61
    - リリース1.0.0(1): 2011/10/19 13:35:57
        - file: 845af1ca3ce21fb543daba5434751cb63fd5a18d
        - method: 5c6afc827e5e365b8db887d5d6375bf28f3617c4
    - リリース2.0.0(2): 2013/09/03 16:38:09
        - file: 5ea3b153840d069b50361f38b8b7050f3c09176b
        - method: 723f268a0d42a4d44f3e628db2e17e75e21b64d3
    - リリース3.0.0(3): 2015/11/06 14:39:23
        - file: 12b8c696bfd8ff9b63bd44dbed1fcfe95bc21aab
        - method: 1b723d6a3d5e77f229857e750af67f352a25c5ec
    - 最新(head): 
        - file: e848d47ed171f20ccd8cf5e20d9e188ede85c17c
        - method: 6d97484ca948055cd306e35b9b6e760e616cead8
- eclipse.jdt.core 19ヶ年＋127日
    - 1: 古すぎてgithubに記録されていない
    - root: 2001/06/05 16:17:58
        - file: be6c0d208933ac936a6ccb6c66b03d3da13e3796
        - method: a7fce940d379c2e5e244d9ddaf1acb77c5df6fe5
    - 2: 2002/06/26 17:54:29
        - file: d0dd6c20e4958b2e8f8c4f7f60f4a15fff6ca500
        - method: ba370a9c7ffb041448d1d6f1b3ed0bcf4b2f36f5
    - 3: 2004/06/24 23:13:09
        - file: c5f94c1be0604760b768518fd2d7014d6ae18052
        - method: 7c3c5b149a5b757dafc9eee38cc2d25bd109519c
    - 4: 2010/07/27 02:37:24
        - file: 3b4f046b547174509da38e0b0a4f7bf6125e51ec
        - method: ebc23690a9bf0afac24abe2147261af0fbe9fa10
    - head: 
        - file: 145fe3a737c63e3d079ddf2fc46dc2640a129635
        - method: 4cc601fa3a7ae1b39957dc3e4ff408a522b0c323
- egit 11ヶ年＋6日
    - root: 2009/09/29 18:18:28
        - file: dfbdc456d8645fc0c310b5e15cf8d25d8ff7f84b
        - method: 2c1b0f4ad24fb082e5eb355e912519c21a5e3f41
    - 1: 2011/06/05 23:43:46
        - file: 0cc8d32aff8ce91f71d2cdac8f3e362aff747ae7
        - method: 1241472396d11fe0e7b31c6faf82d04d39f965a6
    - 2: 2012/06/13 15:46:40
        - file: 1f07f085d6bcae0caf372fffec19583ac5615d3b
        - method: 2774041935d41453e5080f0e3cbeef136a05597d
    - 3: 2013/06/11 00:46:23
        - file: f85dc228e3e67b229c978f751a8723a5c810738b
        - method: 6512ce982bf2376ae7ad308ba3b4a8fafe233376
    - 4: 2015/06/09 07:35:30
        - file: 48712bdfa849e1afd0071b937f0b2bef04767610
        - method: 5f42c83e73b0681da81e7466ae82830d927e916f
    - 5: 2018/06/13 22:05:45
        - file: ba0bcbfe69b2803c843c19b6e800b2525b227cb2
        - method: c6debf07f32e43470bd310f7b60bc19dd78f2506
    - head: 
        - file: 9904d5ba9d7d6b024a73999b5c9b19472a266205
        - method: b459d7381ea57e435bd9b71eb37a4cb4160e252b
- jgit
    - root: 2009/09/29 16:47:03
        - file: 1a6964c8274c50f0253db75f010d78ef0e739343
        - method: 004bd909df360ac2c699073636da678699853b19
    - 1: 2011/06/05 23:26:56
        - file: f65513f75397b1f4de6d063962e5cccca5a89cb6
        - method: a2bc9ac8a1ea177ca25a8410d1daae80ce34e940
    - 2: 2012/06/13 15:03:59
        - file: aacd4f721bba97eda3756e2fcbd71b5e57b3673a
        - method: 730797027641e02f9f05f2481fa3b9eddfa90f91
    - 3: 2013/06/11 00:28:11
        - file: f384644774ae01823e850c862aeda5bddb4a4326
        - method: 63c202af7960e18aafa54426d66ebb013d72461c
    - 4: 2015/06/09 07:29:27
        - file: 4f221854556991e3394b3a71e77ee0b771b1500b
        - method: 436e8b67ec6e56eba12d23ab223ed91ea9cf6cd2
    - 5: 2018/06/13 21:42:40
        - file: e729a83bd24bbc25f7ac209baee01f561fe218c8
        - method: 35b02784c066ad915030c9182b9ec38945133233
    - head:
        - file: 91b2e167a2341d41212ea801943fb82a148e3b7f
        - method: 1a2f9c40ff5df27517099ceb8239d2e53d5cea4d
- linuxtools 11ヶ年＋182日
    - root: 2009/04/09 19:54:15
        - file: b7218a084ed306a498bebdc313756dc706a49830
        - method: ad8b5bf026e3d12e499a3f5acf6ea6469261d377
    - 1: 2012/05/28 10:23:40
        - file: 98e11be9f43ee4b4a574fac1437efc73c88878fd
        - method: 94ead44b7658adb9c528dc2fc3f1cb677669eb46
    - 2: 2013/06/10 16:50:52
        - file: 2aa4892cece1fc393f8bec3083a6303912d17f16
        - method: 3f53ed340e9f722fac8fcc710cac60cc404873b9
    - 3:  2014/06/09 16:15:09
        - file: 1b3ccef1dabb7360d891f0460bb1f8a981aefca8
        - method: 31933bba8ed3aab62d9dcc060d271e88522c73e6
    - 4: 2015/06/12 17:16:33
        - file: 6bfe7ea1397e70b4bea038b294ba6a61940354de
        - method: 1f5314024e098934c01b7a0615b08014c84adb24
    - 5: 2016/06/07 13:34:55
        - file: 4d86c7a081b74595b925ffa64c9578e36b0f7374
        - method: 284f156522dd1e00f07ee28f1fea7f400e2debec
    - 6: 2017/06/13 16:11:14
        - file: 2392872af4a958c41f3d817ae7c3bfe67d1a013a
        - method: d071eea3243834184b8dfb8fcc50ecf0aa29adfa
    - 7: 2018/07/04 17:27:00
        - file: 94bf679091944428c89919c52bb4cef8b2d92ca5
        - method: faaa55c05aae333d25e4bd26242e8229d17c3e2f
    - (8): 新しすぎ: 2020/09/09 13:48:40
        - file: c5dd5836277c944f82072489e3f394053d603499
        - method: 4fd7436b9b5fbecaa5155757738d976e208b4435
    - head: 
        - file: 3af94afe978305f38eed0c6d8eafd7653df0d01f
        - method: 9524114075dd38ceb913a610a4b56da01d6d1bb4
- lucene-solr(1, ..., 8) 19ヶ年＋32日
    - 1: 古すぎてgithubに記録されていない。
    - root: 2001/09/11 21:44:36
        - file: a0e7ee9d0d12370e8d2b5ae0a23b6e687e018d85
        - method: a0e7ee9d0d12370e8d2b5ae0a23b6e687e018d85
    - 2: 2006/05/26 16:50:37
        - file: 191558261298d2dcf24fdbd91b1d1727e69ea99c
        - method: b3f6ea5a814f05e9e8551b965e18567de814b62b
    - 3: 2009/11/22 13:59:47
        - file: 2e244caa1707bc82b7c487cc53ea9ebc4564c6fb
        - method: deecaac786bf460424c56eae72ac0d7f0305077d
    - 4: 2012/08/06 10:33:48
        - file: 4dc925bf4e198487ec455c5881d0c14030f8dd71
        - method: 0fc2819690aa412322019e18c052bcff41e5d94d
    - 5: 2015/02/14 02:37:34
        - file: 429588097cdd0ec86dbf960d49bb1c0ac5d78b72
        - method: 51b92ffe30c50bb08699200b62c40420378ac3df
    - 6: 2016/04/01 05:40:50
        - file: cf7967cc467d9d697d520fcdf92fcdb52f7ddd4e
        - method: 9bd00e9af390947b2751706503ba0d2f0b28da6d
    - 7: 2017/09/08 17:39:12
        - file: a5402f68631768bae57d923613211128de077982
        - method: 14162b1f8f2266547b5e1059f37c2efcf8981ea2
    - 8: 2019/05/08 21:40:06
        - file: 8c6e30536562413639eaf8bab1087da700733b33
        - method: ce6ac2f8144bf1a8fc35c555924357dd8efefc54
    - head: 
        - file: 4bbf2391da3dc23a7fade0d194853a3b82ac30dc
        - method: 2d062b9b9bc669298da80d215e455cf53377c8ce
- poi 18ヶ年＋254日
    - 1: 古すぎてgithubに記録されていない
    - root: 2002/01/31 02:22:28
        - file: 6c234ab6c8d9b392bc93f28edf13136b3c053894
        - method: 6c234ab6c8d9b392bc93f28edf13136b3c053894
    - 2: 2004/01/26 11:17:56
        - file: 0ac1311fd0a411f6cce60e48b595844c360af466
        - method: 81a0a25199ccb15c5c2e4637d0f498b0300e5b3c
    - 3: 2007/05/7 17:09:31
        - file: 8ffd79b36803be29486ab6bdeff5dd02ea901927
        - method: cc1cd7f005bc2379449439427d2c0acef7cd932a
    - 4: 2018/08/31 12:09:06
        - file: 99b9b7828cb268f62daa5b0c5639e549477b2c6f
        - method: 37a74b4a8de772cde9788a6a8c2ae3c0862d31f3
    - head: 
        - file: 382714eccd92667fc83f70115b736c64ebff9700
        - method: 449f08fc0003c64d09663715f28896a4dd010d6a
- wicket(6, 7, 8) 16ヶ年＋20日
    - 1.0.0~1.5.0はスキップすべき。
        - セマンティックバージョニングで管理されていないから。
    - 6: 2012/08/23 22:29:55
        - file: 01b3c702bc33f53af45c60b8a8dc1c45a675ff2c
        - method: 80bb2dbc0f0f8b4f89ba463f9886495f23f79e9b
    - 7: 2015/07/10 09:18:39
        - file: cc6540441db12415d54c725769813248f6948098
        - method: f4058e9842bb6197f139581df19be2057935973d
    - 8: 2018/04/30 20:54:29
        - file: 167a14eef03bc1cc672135f551a091cf4fa999b5
        - method: 25905159ad2ecd911a37d56422d22160880a1066
    - head: 
        - file: 048e5681df960018722237ee6254273e839b5fd2
        - method: 2f1546f5f9ba8fc9fe71253675796f74fea5e953
