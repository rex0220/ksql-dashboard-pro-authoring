# AI によるダッシュボード設定オーサリング手順

**VSCode + Claude Code + kSQL MCP + 設定ファイル仕様**で、
ダッシュボード設定を「実アプリのデータを見ながら AI が書き、git で管理する」ための手順。

- 最終更新: 2026-08-09
- 対応: kSQL エンジン **3.66 系** / [設定ファイル仕様](./設定ファイル仕様.md) **1.14**
- **改訂履歴は git を参照**(`git log --follow docs/AI設定オーサリング手順.md`)
- 関連: [設定ファイル仕様](./設定ファイル仕様.md)(フォーマットの正本)/
  [ダッシュボードレシピ集](./ダッシュボードレシピ集.md)(SQL の書き方)

---

## 1. 全体像

```
  ┌─ VSCode + Claude Code ────────────────────────────────┐
  │                                                       │
  │   ①  kSQL MCP ──────────► kintone(read-only)         │
  │       実アプリのフィールド定義・実データで SQL を確定    │
  │                                                       │
  │   ②  設定ファイル仕様 ──► 設定 JSON を組み立て          │
  │                                                       │
  │   ③  git ──────────────► 差分レビュー・履歴管理         │
  └───────────────────────────────────────────────────────┘
                              │
                              ▼  ④ 設定画面の「インポート」→「保存」
                        kintone アプリ
```

**REST API でプラグイン設定を投入する方法は無い**ため、④ だけはブラウザ操作が残る。

**運用中のダッシュボード設定 JSON は、プラグイン利用者側で個別に管理する。**
本リポジトリが持つのは、仕様・サンプル・検証の仕組みだけである。

---

## 2. 準備

### 2.1 MCP サーバーの登録(このリポジトリでは設定済み)

リポジトリ直下の [`.mcp.json`](../.mcp.json) が Claude Code 用の登録である。
同梱エンジンの MCP サーバーを起動する(別途インストール不要)。

```jsonc
{
  "mcpServers": {
    "ksql": {
      "command": "node",
      "args": ["node_modules/@rex0220/kintone-sql-tools/dist-mcp/ksql-mcp.js"],
      "env": {
        "KSQL_BASE_URL": "${KINTONE_BASE_URL}",
        "KSQL_TOKEN":    "${KSQL_TOKEN:-}",
        "KSQL_USERNAME": "${KINTONE_USERNAME:-}",
        "KSQL_PASSWORD": "${KINTONE_PASSWORD:-}"
      }
    }
  }
}
```

- 認証情報は**環境変数から読む。ファイルには書かない**
- `KSQL_TOKEN` を設定すればトークン認証。未設定ならユーザー名 / パスワード
  (空の `KSQL_TOKEN` を渡しても動作することは確認済み)
- Claude Code の初回起動時にサーバーの使用可否を尋ねられる。VSCode 拡張でも同じ

Claude Desktop で使う場合は同梱の MCPB(`dist-mcpb/ksql-mcp.mcpb`)を導入する。

### 2.2 read-only を担保する

設定オーサリングに書き込みは不要である。次のいずれかで担保する。

1. **API トークンを「閲覧」権限のみで発行し `KSQL_TOKEN` に設定する**(推奨)。
   サーバー側で書き込みが失敗するため確実
2. `ksql_mutate` を使わせない。このツールは `allowDml` / `confirmText` / `dmlMaxRows` の
   明示指定が無いと実行されない安全機構を持つが、**運用としては ① を優先する**

### 2.3 疎通確認

Claude Code で `ksql_show_apps` を呼び、アプリ一覧が返れば準備完了。

---

## 3. 作業手順(AI に指示する流れ)

### ステップ 1 — 対象アプリとフィールドを確定する

```
ksql_show_apps                     → アプリ ID を特定
ksql_describe_app { app: 4149 }    → フィールドコード・ラベル・型を取得
```

**ラベルとフィールドコードは一致しないことがある。** 例えば SFA パックの案件管理は
ラベル「顧客No.」に対してコードが `顧客No_`。**SQL にはコードを書く。**

**そして、コードとラベルが違うときは `AS` でラベルを付ける。** 表の見出し・グラフの凡例・
KPI はすべて結果の列名を出すため、コードのままだと内部名が閲覧者に見える。

```sql
SELECT 顧客No_ AS `顧客No.`, 金額 AS 受注金額 FROM APP100
```

記号・空白を含むラベルはバッククォートで囲む。ラベルが日本語だけなら結果列名もそのままだが、
**英数字が含まれると列名は小文字化**される(`` AS `顧客No.` `` → `顧客no.`)。
表示は書いたとおり出る(`displayName`)ので、**`columns` / `totals` / `mapping` の参照だけ**
小文字の側を使う。

**要件がラベルで書かれていたら、どのコードへ対応させたかを利用者に示してから進む。**
似たラベルが複数あるとき(「受注予定金額」「受注予定金額(税込)」「受注実績金額」など)は
推測しない。取り違えても**数字は出てしまう**ため、いちばん気づきにくい壊れ方をする。

型も必ず確認する。`USER_SELECT` / `CREATOR` / `CHECK_BOX` / `MULTI_SELECT` は
`=` が使えず `in` / `not in` が必要になる。

### ステップ 2 — SQL を確定する

```
ksql_validate  { sql }   → 構文チェック(kintone API を呼ばない)
ksql_explain   { sql }   → 実行計画。押し下げが効いているかを確認できる
ksql_query     { sql }   → 実データで結果を確認(列名・件数・値)
```

**`ksql_query` の列名を必ず確認する。** KPI の予約列名、グラフの `mapping`、
表の `columns` はすべて**結果の列名**を参照するため、ここがずれると表示が壊れる。

文法で迷ったら `ksql_docs`(または resources `ksql://language-reference` /
`ksql://recipes`)を引く。

> **⚠ MCP で動いても、ペインで動くとは限らない。**
> `APP@profile` / `IMPORT` / DML の `EXPLAIN` / DML の `VALIDATE ONLY` /
> `INSERT`・`UPDATE`・`DELETE` は **プラグインではエラー**になる。
> 詳細は [設定ファイル仕様 §8](./設定ファイル仕様.md#8-sql-の制約プラグインサブセット)。
>
> **`LAPP_<名前>` が使える。**設定の
> `logicalApps`(名前 → アプリ番号の対応表)を定義すれば解決される。**環境非依存の
> 設定ファイルを作る推奨手段**なので、配布・移行前提の設定では積極的に使う
> (封筒の `sqlApps` には `logicalName` 付きで列挙する)。MCP 側はプロファイルの
> `logicalApps` で同名を定義すれば同じ SQL がそのまま検証できる。
>
> **`;` 区切りの複文と `CREATE TEMP TABLE` は 2026-07-28 から使える**。
> **描画されるのは最後に結果セットを返した文。**
>
> **SELECT 別名の英字は結果列名では小文字になる**(エンジン仕様。`AS ランクA` →
> 列名 `ランクa`)。表示(見出し・凡例)は `displayName` により書いたとおりに出るが、
> **設定の `columns` / `totals` / `mapping` が参照するのは列名(= 小文字の側)**。
> `ksql_query` が返した列名をそのまま使えば間違えない。

### ステップ 3 — 設定 JSON を組み立てる

[設定ファイル仕様](./設定ファイル仕様.md) に従う。特に:

- **必ず封筒形式で作る**(§0.1: `date` / `pluginName` / `pluginVersion` / `engineVersion` /
  `appId` / `appName` / `sqlApps` / `config`)。**素の config だけのファイルを作るのは誤り** —
  インポートは通るが、確認ダイアログが「出力元の情報がありません(旧形式または手書き)」になり、
  アプリ番号変換にアプリ名が出ない。封筒なしはインポート側の後方互換であって、生成側の選択肢ではない
- `sqlApps` には SQL が参照する全 `APP<番号>` を列挙(名前は `ksql_show_apps` から)。
  `engineVersion` は同梱エンジンの実バージョンを書く
- `config` 内は `schemaVersion: 2` / `edition: "pro"` 固定
- ペインは **12 枚まで**、`layout[].i` と `panes[].id` を **1:1 で必ず両方書く**
- グリッドは 60 カラム、`x + w ≤ 60`
- **`options` のキー名の誤りはエラーにならず無視される**

一覧 ID に依存させたくない場合は `views` ではなく **`common.view`** に書く。

雛形として [samples/sfa-案件管理-dashboard.json](./samples/sfa-案件管理-dashboard.json)
(9 ペイン・JOIN・条件付き書式・マークダウンを含む実例)をコピーするのが早い。

### ステップ 4 — 検証する

**専用の検証コマンドは作っていない。**設定がプラグインへ入る瞬間に
`importSettingsJson` が検証しており、そこが最適な位置だからである
(構造・サイズ・無償版エクスポートの封筒剥がし。
**検証に失敗しても既存設定には一切触れない**純関数)。

| 何を | どこで守られるか |
| :--- | :--- |
| 構造・サイズ | **インポート時**(`importSettingsJson`)+ 設定画面のサイズメーター |
| SQL の構文 | 設定画面の SQL 欄(0.6 秒デバウンスで `explainQuery`) |
| SQL の値 | **設定画面のプレビューは実データ**。保存前に数字を確認できる |
| CLI / MCP 専用構文の混入 | `npm run check:sql`(本リポジトリの SQL 資産を CI で検査) |

> 汎用の `npm run check:settings <file>` は **検討したが却下した**
> (2026-07-27)。①〜④ はすべて上表の重複で、⑤ 正規化出力は必要性が未確認のため。

### ステップ 5 — 反映する

**通常はインポート UI を使う。**

1. アプリの設定 → プラグイン → kSQL Dashboard Pro の **設定**
2. **インポート** で JSON を選択(検証に失敗した場合、既存設定は変更されない)
3. **保存**

保存後、一覧画面で各ペインを目視確認する。SQL の誤りはここで初めて現れる。

#### 貼り付け 1 回で反映する

ファイル選択を挟まずに投入できる。**設定画面のコンソール**で:

```js
localStorage.setItem('rex0220-kdp-debug', '1');   // 1 回だけ。以後は不要
location.reload();
```

リロード後、同じコンソールで JSON テキストを渡す。

```js
kSQLDashboardPro.saveSettings(`{ "schemaVersion": 2, … }`)
```

**インポート UI と同じ経路**(`importSettingsJson`)を通るため、
**無償版のエクスポート(封筒形式)もそのまま貼れる**。
保存 → アプリ更新 → リロードまで自動で進む。

> `saveSettings` は診断フラグを立てたときだけ生える。
> 権限の防壁ではない(`setConfig` / deploy は kintone がサーバー側で検査する)。

---

## 4. AI への指示テンプレート

```
kSQL MCP を使って、アプリ APP<番号> のダッシュボード設定を作ってください。

手順:
1. ksql_describe_app でフィールドコードと型を確認する(ラベルではなくコードを使う)
2. 各ペインの SQL を ksql_validate → ksql_query で確認し、返る列名を控える
3. docs/設定ファイル仕様.md に従って設定 JSON を書く
4. docs/samples/sfa-案件管理-dashboard.json を雛形にしてよい

制約:
- SQL にはフィールドコードを書く。**コードとラベルが違う列は AS でラベルを付ける**
  (例 `顧客No_ AS \`顧客No.\``)。記号・空白を含むラベルはバッククォートで囲む
- 要件がラベルで書かれていたら、**どのコードへ対応させたかを先に示す**。似たラベルが
  複数あるときは推測せず確認する
- 出力は必ず封筒形式(date / pluginName / pluginVersion / engineVersion / appId / appName /
  sqlApps / config)。素の config だけで出力するのは誤り
- ペインは最大 12 枚、layout[].i と panes[].id を 1:1 で両方書く
- x + w <= 60
- APP@profile / IMPORT / DML の EXPLAIN / DML の VALIDATE ONLY /
  INSERT・UPDATE・DELETE は使わない(ペインで動かない)
- SELECT 別名の英字は結果列名では小文字になる。columns / totals / mapping の参照は
  ksql_query が返した列名(小文字の側)をそのまま書く
- ユーザー選択・複数選択フィールドには = ではなく in を使う
  (ドロップダウン・ラジオボタン・ステータスは = / != でも自動で押し下がる。
  ただし定義に無い値・空文字は押し下がらない)
- 期間の絞り込みは相対日付。JOIN でも使えるが、LEFT/RIGHT JOIN・アプリをまたぐ OR・
  CTE を JOIN の入力にした形は不可
- LEFT/RIGHT JOIN は取得上限に掛かりやすい。押し下げが効かず相手のアプリを丸ごと取るため、
  保持されない側が上限に達するとエラーになる。使うなら取得上限を候補集合より大きくする
- 集計を含む SELECT に、集計もグループ化もしていない列を書かない(エラーになる。
  MIN()/MAX() で包むか GROUP BY に足す。ANY_VALUE は無い)
- UNION は両辺の列数を揃える(不一致はエラー。足りない側へ '' などを足す)
- 値が 0 個のときの MIN/MAX は 0 ではなく空文字。0 件判定・除数ガードは = '' も見る
- サブテーブルの中の項目は APPn$テーブル から選ぶ(親から SELECT すると全行が空になる)
- SELECT * は使わず列を明示する(設定が列名を参照するため)

使えるもの(積極的に使ってよい):
- ; 区切りの複文・CREATE TEMP TABLE・SET・DECLARE・ASSERT
  → 構成比は CREATE TEMP TABLE → SET @total = (SELECT …) → 最後に SELECT で書く
  → 描画されるのは「最後に結果セットを返した文」
- LAPP_<名前>(論理アプリ名)+ config.logicalApps — 配布・環境移行前提の設定はこちらで書く
- 変数コントロール(view/タブの controls)— 閲覧者が値を切り替えるバー。DECLARE は書かない
  - type "date": WHERE 日付列 = @<変数>(kintone 関数が入る)、任意区間対応は
    WHERE 日付列 >= @<変数>_from AND 日付列 < @<変数>_to_next(半開区間)
  - type "string" / "number": 供給されるのは @<変数> 1 本だけ(範囲形は date のみ)。
    options で固定リスト、optionsSql で動的リスト(1 列目 = 値 / 2 列目 = 表示名)
  - optionsSql は WHERE がなければ対象アプリを全件読む(DISTINCT はエンジン側なので
    取得は減らない)。マスタアプリから取るか、WHERE 作成日時 >= LAST_YEAR() で絞る
  - 値と表示名が違う固定リストは、リテラルの UNION ALL で書く(FROM なし = 取得ゼロ)。
    SELECT 'A' AS 値, 'Aプラン' AS 表示名 UNION ALL SELECT 'B', 'Bプラン'
  - KLIKE の右辺に空文字は渡せないので、string を KLIKE に使うなら default を必ず書く
  - ユーザー選択フィールドの絞り込みには使わない(照合は code で、存在しない値は
    0 件ではなく kintone API エラーになる)
- VALIDATE APP<番号>(入力規則違反の洗い出し。データ品質ペイン)
- GENERATE_SERIES(start, stop [, step]) — WITH の CTE 本体にだけ書ける整数・日付系列。
  取引の無い日・月を 0 で並べる推移は、系列を左辺にした LEFT JOIN で書く(レシピ D20)。
  step は '1 day' / '1 month' / '1 year' 形式(月・年 step の start は月初・1 月 1 日限定)、
  生成上限 10,000 行、取得ゼロ
- CROSS JOIN — 明示の直積(日付系列 × マスタで「日付 × 製品」の格子 → 製品別 0 埋め。
  レシピ D20)。直積も上限 10,000 行(WHERE / LIMIT で免除されない)。ON 1=1・カンマ結合は不可
- WITH RECURSIVE(再帰 CTE。3.66.0〜)— 部品表・組織図など深さがデータ次第の階層を展開
  (レシピ D21)。seed UNION ALL 再帰項の 2 枝・自己参照 1 回・INNER 等値 JOIN 1 本に限る。
  参照アプリは全件実体化(WHERE が押し下がらないため、アプリ全体が取得上限に収まること)。
  深さ 100・累積 10,000 行・中間展開 100,000 の三境界超過は部分結果なしのエラー。
  循環しうるデータは CYCLE 句で経路単位に打ち切る。固定深さの階層なら自己 JOIN で足りる
- ウィンドウ関数 — 順位系(ROW_NUMBER / RANK / DENSE_RANK)・集計ウィンドウ
  (SUM/COUNT/AVG/MIN/MAX … OVER。累積残高)・LAG / LEAD(前月比は月次集約 → LAG → 比率の 3 段)。
  OVER(…) と AS 別名は必須。集計・GROUP BY と同じ SELECT には書けない(CTE で分ける)。
  集計ウィンドウ・LAG/LEAD は取得上限に達すると打ち切りではなくエラー
- 取得上限・一時テーブル上限は最大 50,000(既定 10,000。大きくするなら options.maxRecords /
  options.tempTableMaxRows)

作りたい内容: <ここに要件>
```

---

## 5. git 管理

**運用中の設定 JSON はプラグイン利用者側で個別管理する。**
本リポジトリには仕様・サンプル・検証の仕組みのみを置く。

利用者側で git 管理する場合の注意:

| 項目 | 方針 |
| :--- | :--- |
| 差分の安定性 | エクスポートは整形済み JSON だが、**キー順は挿入順**。GUI 編集とエクスポートを繰り返すと差分がノイズ化しうる。正規化(キーのソート)出力があると安定するが、**差分ノイズが出た記録が無い**ため見送っている(案 A-⑤) |
| 機密情報 | 認証情報は含まれない。ただし**アプリ番号・フィールド名・業務用語**は含まれるため、公開リポジトリは避ける |
| 正の所在 | 「git が正」と決めるなら、GUI で直した内容をエクスポートして戻す運用ルールが要る |

---

## 6. 確認済みの環境情報

| 項目 | 内容 |
| :--- | :--- |
| MCP サーバー | `ksql-mcp` **3.66.0**(エンジンパッケージ同梱。Node.js 20 以上)。接続時の instructions 1 行目に版数が出る。**`ksql_docs` を引数なしで呼ぶと稼働中の版数を返す**(3.56.3〜)。エンジン更新後は MCP クライアントの再読み込みが必要(常駐プロセスは `npm install` では差し替わらない) |
| ツール | 13 個 — `ksql_show_apps` / `ksql_describe_app` / `ksql_app_metadata` / `ksql_validate` / `ksql_explain` / `ksql_query` / `ksql_docs` / `ksql_mutate` / 保存クエリ 5 種 |
| resources | `ksql://language-reference` / `ksql://recipes` |
| 認証 | `KSQL_BASE_URL` +(`KSQL_TOKEN` または `KSQL_USERNAME`/`KSQL_PASSWORD`) |
| 疎通確認 | `initialize` → `tools/list` → `ksql_query` / `ksql_describe_app` の実行まで確認済み |
| テンプレートギャラリー | **13 種** |
