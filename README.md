# Tableauアカウント登録バッチのCOREデータ切替

## 概要

**Tableauのアカウントを自動で登録するアプリ(バッチ)があり、アプリで使用している人事情報をIDshareからCOREデータに切り替えた。**

**切り替え対応を行いリリース済みなので、本バッチ処理の説明とソース修正について発表する。**

## 執筆者

* CCI テクノロジーDiv 開発第2チーム E1
* 野口 貴裕

## 背景

Tableauのアカウントは手動による運用と本バッチ処理による自動運用で管理されている

自動運用の際に本バッチ処理で社内の人事情報を元にアカウントが登録(削除・権限変更等)される

CARTAへの経営統合により、従来のIDshareのデータから全社的にKintoneの人事情報（COREデータ）利用に切り替える方針となった

そのため、本バッチ処理で利用している人事情報取得をIDshareデータからCOREデータに切り替える必要があった

また、昨年(2022年)の12月から本アプリは停止していたので、アカウント登録が必要な際に手動対応となっていた

## システム概要(lake.biにおいてTableau周り関連するところ)
### ■lake.bi
* 弊社における、広告の運用実績が蓄積されたデータ基盤プロダクト

### ■基盤EVM(onlineレポート)
* 運用実績が蓄積されたデータはTableauのデータソースで抽出され、Excel形式のレポートに利用されている
* Web画面ではTableauダッシュボードが埋め込まれていて、入力条件に沿ったレポートを提供する

### TableauServerについて(サイト・ユーザー・ロール)
#### サイト
　lake.biでアカウント管理しているサイト種類

| サイト | 用途 |
| :--: | :--- |
| agency | 社内版lake.bi |
| bxreport_for_agency | 社外版lake.bi |
| media | メディアソリューション・ディビジョン用ダッシュボード |
| cci_all | CCI社員用 |

  
#### ユーザー
- ユーザー：ESQ_ID
- サイトロール：サイトに対するロール

※ロールは本バッチ処理においては以下のサーバー管理者以外をサイト毎に付与
  
#### ロール
| 権限名 | 説明 | 権限範囲 |  
| :--: | :--- | :--: |  
| サーバー管理者 | TableauServerに対する全権限を保持 | 全体 |
| サイト管理者 | 特定のサイトに対する全権限を保持 | サイト |
| Explore(Publish可能) | 特定のプロジェクトに対して各種コンテンツを公開可能 | プロジェクト |
| Explore | TableauServerに対する全権限を保持 | プロジェクト |
| Viewer | 閲覧権限のみ | プロジェクト |

### Tableauアカウント登録バッチ
* TableauServer(WindosServer)内に配置されるバッチ処理で主な処理はWindowsバッチで記述されている
* Tableauのアカウント登録(追加・更新・削除)を行う

* 処理フロー
```
・人事情報取得・PostgreDB登録
　　以下はサイト毎
・スプレッドシートデータ取得・PostgreDB登録
・Tableau情報取得・PostgreDB登録
・ユーザー管理情報生成
・Tableau反映(ユーザー管理機能)
```

![tableau_account_manager_flow](https://github.com/ta-noguchi-cci/cci_ta-noguchi/blob/main/images/tableau_account_manager_flow.png)

### IDShareデータについて

Tableauアカウント登録バッチでは人事情報としてIDShareデータ(元データはCOMPANYのデータ)を使用していた

※IDShareのデータはCOMPANYが元データでID連携サーバーからS3に配置されるファイル(AccountList.csv)

簡単にIDShareデータの取得の流れを記しておく
#### COMPANY　→　S3(cci-id-share)にCPID_xx.csv　→　ID連携サーバ(ID_SHARE)でAccountList.csv作成・S3配置　→　S3(cci-id-share)

## 解決手法

人事情報のデータはTableauサーバー内にあるPostgreDBのacmスキーマのcimテーブルに登録されている。

cimテーブルにこれまでのIDShareデータと同等の内容のデータが登録されれば、後続の処理は変更せずに済むのでCOREデータの確認と各カラムの使用有無を確認をした。

### acm.cimテーブル定義
| カラム名 (論理名) | カラム名 (物理名)| 型 | 制約 |
| :--- | :--- | :--- | :--- |
| exec日付 | exec_date | date | not null |
| esq_id | esq_id | text | not null |
| emp番号 | emp_no | text |  |
| division | division | text |  |
| team | team | text |  |
| ロール | role | text |  |
| 名前 | name | text |  |
| 名前カナ | name_kana | text |  |
| 名前english | name_english | text |  |
| mail住所 | mail_address | text |  |
| hire日付 | hire_date | date |  |

### core_dbについて

COREデータは以下の流れでCSVファイルが出力されている

#### CJK（正社員の情報）→CIM（業務委託＋その他アカウントの情報）→CSV出力→外部連携サーバ→転送サーバ→S3（core-storage01）

切り替え当初(2022年12月頃)は、S3のcore-storage01に出力されるhito.csvをCOREデータとして使用する予定であった

しかし、テクノロジー・ディビジョンの他のシステムでもCOREデータを使用するため、共通のDB(core_db)を用意することとなった

DBの用意等はアーキテクチャインテグレーションチームの方にお願いする形となった

### 直接COREデータ(S3のcore-storage01のhito.csv)を取得

core_db案が出る前までは、S3のcore-storage01のhito.csvを取得する形で実装を進めていた

#### COREデータ(hito.csv)からデータ取得⇒PostgreDBのcoreテーブル(新規作成)⇒必要なものだけPostgreDBのcimテーブルに登録

#### cimテーブルへCOREデータをマッピングした際には以下のようになる

### acm.cimテーブルへのマッピング
| 列 | 型 | 制約 | coreテーブルカラム番号 |
| :--- | :--- | :--- | :--- |
| exec_date | date | not null | - |
| esq_id | text | not null | No.2 統合ID（ESQID） |
| emp_no | text |  | No.1 社員番号 |
| division | text |  | No.6 所属予備２名称 |
| team | text |  | No.10 所属名称 |
| role | text |  | No.11 役割名称 |
| name | text |  | No.3 社員名称 |
| name_kana | text |  | No.4 社員カナ名称 |
| name_english | text |  | Coreデータなし |
| mail_address | text |  | No.18 システム使用アドレス |
| hire_date | date |  | Coreデータなし |

* Coreデータなしの部分は後続の処理で使用されていないので問題なし
* ここまでの実装はブランチを切って保存
https://ap-northeast-1.console.aws.amazon.com/codesuite/codecommit/repositories/TableauAccountManager/browse/refs/heads/LBPJ-1592?region=ap-northeast-1

## core_dbができてから

### core_dbから作成されたCSVファイルを取得する方法に変更

#### CoreData 専用テーブルDB上⇒Troccoジョブで経由⇒S3:stg-lakebi-core-data/へcore_data_lakebi.csv
#### core_data_lakebi.csvに出力する流れまではアーキテクチャインテグレーションチームの方にお願いする形となった
#### core_data_lakebi.csv取得し、PostgreDBのcimテーブルに登録
#### coreテーブルは使用なしとなるため削除

core_data_lakebi.csvに出力する内容はcimテーブルで必要な項目のみ

### core_data_lakebi.csv
| 列 | cimテーブルカラムマッピング |
| :--- | :--- |
| esq_id | esq_id |
| emp_no | emp_no |
| division | division |
| team | team |
| name | name |
| name_kana | name_kana |
| name_english | name_english |
| mail_address | mail_address |
| employee_role | role |

### ソース修正

#### ソース修正は以下になる

##### 1. COREデータ取得batファイル作成

   - IDShare取得と同じような処理なので、IDShare取得をコピーしてCOREデータ取得用に修正しbatファイル作成

   - 設定ファイル(setenv.bat)のS3接続設定の修正

##### 2. COREデータCSVに重複行が存在するので削除

   - Windows PowerShellで作成(ps1ファイル)

```
# 重複データを削除する
Get-Content -Path $input_file_path -Encoding $charset | sort -Unique | Set-Content -Encoding $charset $output_file_path
```

##### 3. cimテーブルに登録するymlファイル作成

   - cimテーブルに登録するimport_to_cim.ymlをコピーしてimport_to_core.ymlとして、COREデータ登録用に修正しymlを作成

```
in:
  type: file
  path_prefix: #PATH_PREFIX#
  parser:
    charset: #CHARSET#
    newline: #NEWLINE#
    type: csv
    delimiter: ','
    quote: '"'
    escape: '"'
    trim_if_not_quoted: false
    skip_header_lines: 1
    allow_extra_columns: false
    allow_optional_columns: false
    columns:
    - {name: esq_id, type: string}
    - {name: emp_no, type: string}
    - {name: division, type: string}
    - {name: team, type: string}
    - {name: name, type: string}
    - {name: name_kana, type: string}
    - {name: name_english, type: string}
    - {name: mail_address, type: string}
    - {name: role, type: string}
filters:
  - type: column
    add_columns:
      - {name: exec_date, type: timestamp, default: #EXEC_DATE#, format: "%Y/%m/%d"}
      - {name: hire_date, type: timestamp, default: 2022/01/01, format: "%Y/%m/%d"}
  - type: row
    condition: AND
    conditions:
      - {column: esq_id,  operator: "IS NOT NULL"}
      - {column: division,  operator: "IS NOT NULL"}
      - {column: team,  operator: "IS NOT NULL"}
out:
  type: postgresql
  host: #DATABASE_HOST#
  database: #DATABASE#
  schema: #DATABASE_SCHEMA#
  user: #DATABASE_USER#
  password: #DATABASE_PW#
  table: #TABLE#
  mode: insert
  before_load: delete from #DATABASE_SCHEMA#.#TABLE# where exec_date = '#EXEC_DATE#'
  column_options:
    exec_date: {type: 'date'}

```

## 今後の展望

### 実装面

- 削除不要なアカウントが削除されてしまうことがあるため調査が必要
- 評価と本番にアカウントやアカウントの状態に差異があり、本番を想定したテストがやりにくい状況になっているので、評価と本番の差異の把握
- 差異を把握した上で本番環境に近い環境(データ)の構築・設定
- スプレッドシートはCCIテナントのGドライブから取得しているのでCARTAテナントへの切り替え
- 検証(実現できるか)・工数次第ではあるが個人的にはJavaでの処理に変えたい

### 運用面

- こちらでは現状、本バッチ処理の実施のみのため、問い合わせ対応やアカウント追加等の運用はTableauアーキテクチャインテグレーションチームの方にお願いしている形になっているので、こちらでも対応できるようにする

## 参考

参考資料
- Tableauアカウントバッチ処理概要.xlsx

https://docs.google.com/spreadsheets/d/16J51KfOlaTyBXsVnjIT0XCYK77M4so32/edit?usp=sharing&ouid=109432511723231409224&rtpof=true&sd=true

- 設計書類

https://drive.google.com/drive/folders/1wKrXbIsxJwkIpC6u-vd_EG_7cGL6Qqhi


