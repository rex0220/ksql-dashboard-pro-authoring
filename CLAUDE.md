# kSQL Dashboard Pro — 設定ファイルのオーサリング環境

このリポジトリは、**要件から kSQL Dashboard Pro の設定 JSON を AI が生成・変更する**ための環境です。
生成した JSON は、プラグイン設定画面の **ツール → インポート / ビューへ取り込む** で反映します。

## 正本ドキュメント(必ずこの順で参照)

1. [docs/設定ファイル仕様.md](docs/設定ファイル仕様.md) — 設定 JSON の形式の正本
2. [docs/ダッシュボードレシピ集.md](docs/ダッシュボードレシピ集.md) — SQL の書き方と設計原則
3. [docs/AI設定オーサリング手順.md](docs/AI設定オーサリング手順.md) — 作業手順の詳細
4. [docs/samples/sfa-案件管理-dashboard.json](docs/samples/sfa-案件管理-dashboard.json) — 雛形

## 作業の流れ

要件は `requirements/` のファイル、またはチャットで受け取る。

1. 対象アプリの番号を特定する — 要件に番号があればそれを使い、`ksql_describe_app` で名前と
   一致するか確かめる。名前しか無ければ `ksql_show_apps` で探すが、**同名・類似名の候補が
   複数あるときは推測せず、番号(またはアプリの URL)を利用者に確認してから進める**
2. `ksql_describe_app { app: N }` → **フィールドコードと型**を確認する(ラベルではなくコードを SQL に書く)
3. 各ペインの SQL を `ksql_validate` → `ksql_query` で実データ確認し、**返る列名を控える**
4. **件数チェック** — 集計・グラフのペインは、同じ WHERE 条件で
   `SELECT COUNT(*) FROM APPn WHERE …` を実行して対象件数を確認する(単発 GET で安価)。
   対象件数が取得上限を超えるペインは**実行時に「正しい集計値を出せません」エラーになる**
   (明細の表は打ち切り表示)。上限は既定 10,000・設定で 500〜50,000。対処は優先順に:
   ① WHERE(期間など)で絞る ② `options.maxRecords` を件数に余裕を持たせて上げる
   ③ 50,000 でも収まらないなら設計を変える(集計軸や期間の見直し)。
   `maxRecords` を既定より**下げる**ときは、対象件数に十分な余裕があることを必ず確認する
5. docs/設定ファイル仕様.md に従って設定 JSON を組み立てる
6. **封筒形式で保存する**(素の config だけだと、インポートの確認ダイアログに
   「出力元の情報がありません」と表示され、アプリ番号変換にアプリ名が出ない):

   ```jsonc
   {
     "date": "<生成日時 YYYY-MM-DD HH:MM:SS>",
     "pluginName": "kSQL Dashboard Pro",
     "pluginVersion": "1",
     "engineVersion": "3.39.0",           // node_modules の @rex0220/kintone-sql-tools の version
     "appId": <対象アプリ番号>,
     "appName": "<対象アプリ名>",
     "sqlApps": [                          // SQL が参照する APP<番号> をすべて列挙(名前は ksql_show_apps から)
       { "appId": 4149, "appName": "案件管理" }
     ],
     "config": { /* schemaVersion 2 の設定本体 */ }
   }
   ```

7. 保存先は**利用者が指定したファイル名に従う**(ファイル名の体系は利用者の管理)。
   指定が無ければ `settings/<アプリ名>-<ダッシュボード名>.json` を提案する。
   既存ダッシュボードの変更は**必ず同じファイルへの上書き**(新規ファイルを増やさない。
   履歴は git の diff で追う)。新規か更新か曖昧な指示のときは先に確認する
8. 反映方法(インポートかビューへ取り込むか)と確認ポイントを利用者に伝える

既存設定の変更は、利用者がエクスポートした JSON(封筒形式)を受け取り、`config` の中身だけを編集し、
`date` を更新して同じ形式で返す。`sqlApps` は SQL の参照アプリを変えたときだけ更新する。

## 制約(ペインで動かない・壊れる書き方)

- ペインは 1 タブ最大 12 枚。`layout[].i` と `panes[].id` を 1:1 で両方書く。`x + w <= 60`
- APP@profile / IMPORT / DML(INSERT・UPDATE・DELETE)とその EXPLAIN / VALIDATE ONLY は使わない
- **SELECT 別名の英字は結果列名では小文字になる**(`AS ランクA` → 列名 `ランクa`。表示は
  書いたとおりに出る)。`columns` / `totals` / `mapping` の参照は `ksql_query` が返した列名
  (小文字の側)をそのまま書く
- ユーザー選択・作成者・更新者・チェックボックス・複数選択には `=` ではなく `in` / `not in`
- ドロップダウン・ラジオボタンも `in` を使う(`=` は押し下げされず全件取得になる)
- `SELECT *` は使わない(設定が列名を参照するため列を明示)
- 期間の絞り込みは相対日付関数(THIS_MONTH() 等)。LEFT/RIGHT JOIN 内・アプリをまたぐ OR・
  CTE を JOIN の入力にした形では使えない
- LEFT/RIGHT JOIN は取得上限に掛かりやすい(押し下げが効かず相手アプリを丸ごと取る)

## 使えるもの(積極的に使ってよい)

- `;` 区切りの複文・CREATE TEMP TABLE・SET・DECLARE・ASSERT
  (構成比は CREATE TEMP TABLE → SET @total = (SELECT …) → 最後に SELECT。描画は「最後に結果セットを返した文」)
- **`LAPP_<名前>`(論理アプリ名)+ `config.logicalApps`** — 配布・環境移行前提の設定はこちらで書く
  (封筒の `sqlApps` には `logicalName` 付きで列挙。MCP 側はプロファイルの `logicalApps` に同名を定義)
- **期間コントロール**(view / タブの `controls`: `type: "date"`)— 閲覧者の期間切り替え。
  ペイン SQL は `WHERE 日付列 = @<変数>`(kintone 関数が入る)、任意区間対応は
  `WHERE 日付列 >= @<変数>_from AND 日付列 < @<変数>_to_next`(半開区間)。`DECLARE` は書かない
- UNION の枝の COUNT(*) はラベル列つきでも単発 GET — アプリ件数一覧に最適
- VALIDATE APP<番号>(データ品質ペイン)
- ウィンドウ関数 ROW_NUMBER / RANK / DENSE_RANK(`OVER(…)` と `AS 別名` は必須)
- 取得上限・一時テーブル上限は最大 50,000(既定 10,000。`options.maxRecords` / `options.tempTableMaxRows`)

## このリポジトリのルール

- **封筒なし(素の config だけ)のファイルを作るのは誤り。** 生成・変更とも必ず封筒形式で出力する。
  封筒なしの受け入れはインポート側の後方互換であって、生成側の選択肢ではない
- kintone への書き込みは行わない(`ksql_mutate` は使用禁止。ログインユーザー認証で
  書き込み権限があっても、このリポジトリでは読み取り専用で扱う)
- 認証情報をファイルに書かない(.env はコミット対象外)
- 生成 JSON は保存前に必ず `ksql_validate` 済みの SQL だけを含める
