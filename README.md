# RGraphQL

GraphQLのRust版。既存のGraphQL実装(`async-graphql`/`juniper`/
`graphql-parser`等)のコードを一切流用せず、GraphQL仕様(2021年10月版)を
一から実装するプロジェクト(RFrontEndエコシステム傘下)。

## これは何か

- **コアはフレームワーク非依存の純粋ライブラリ**: GraphQLクエリ言語の
  字句解析(`lexer`)・構文解析(`parser`)・AST(`ast`)を、`poem`/`tokio`等の
  サーバーフレームワークに依存せず提供する(パーサーだけ使う場合に不要な
  重さを背負わせない)。
- **HTTP公開層は`poem`フィーチャに分離**: RPoem/Poem(tokio/hyper)経由で
  GraphQLエンドポイントを公開する統合はアダプタ層(`poem_adapter`)に置く。
  `async-graphql`がコアとWeb統合を別クレートに分けているのと同じ構造。
- **`unsafe`コード不使用・外部パースクレート非依存**(`unsafe_code = "deny"`)。

## v0.1.0のスコープ(正確性優先・性能は後回し)

- GraphQLクエリ言語のトークナイザ(区切り子・名前・整数/浮動小数・
  文字列/ブロック文字列・コメント`#`・無視トークンのカンマ)。
- 単純な`query`操作のパーサー: フィールド選択・ネストした選択集合・
  引数・エイリアス。引数値は変数参照・数値・文字列・真偽値・null・
  列挙値・リスト・入力オブジェクトに対応。

## v0.2.0で追加したスコープ

- `mutation`操作のパース(`query`と同じ文法で操作種別のみ異なる)。
- 変数定義(`query Foo($id: ID!, $limit: Int = 10)`)と型参照
  (名前付き型・リスト`[T]`・非null`T!`のネスト)。クエリ内での変数参照
  (`field(arg: $var)`)は既存の`Value::Variable`をそのまま利用。
- フラグメント定義(`fragment Name on Type { ... }`)・フラグメント
  スプレッド(`...Name`)・インラインフラグメント
  (`... on Type { ... }` / 型条件省略の`... { ... }`)。
- ディレクティブ(`@include(if: $x)` / `@skip(if: $x)`等)の構文解析。
  操作・フィールド・フラグメント定義・フラグメントスプレッド・
  インラインフラグメントの各所に付与できる。実行時評価(条件による
  フィールド除外)はvalidation/execution層の範囲であり未実装。

## v0.3.0で追加したスコープ

- `validation`モジュール: パース済み`Document`を検証する
  `validate(&Document) -> Vec<ValidationError>`。スキーマ定義言語(SDL)/
  型システムがまだ存在しないため、ドキュメント自体だけで判定できる
  構造的ルールのみが対象。実装したルール:
  - フラグメントスプレッド先の存在確認(未定義フラグメントの参照)。
  - フラグメントの循環参照検出(直接・間接どちらも)。
  - 未使用フラグメント定義の検出(他フラグメント経由の間接参照も追跡)。
  - 無名操作は文書内で唯一の操作である場合のみ許可。
  - 操作名の一意性(重複した名前付き操作を検出)。
  - 変数の定義/使用整合性(未定義変数の使用・未使用の宣言変数、
    フラグメントスプレッド越しの使用も追跡)。
  - フィールド選択マージ(同一選択集合直下でのエイリアス/引数衝突)の
    部分集合。
  **対象外(正直な開示)**: 型システム依存のルール一式(SDL未実装のため)、
  フラグメントスプレッド越しのフィールド選択マージ判定、ソース位置
  (span)情報の付与(ASTがトークン位置を保持しないため)。詳細は
  `src/validation.rs`のモジュールドキュメントを参照。

### 未対応(次段階 v0.4.0以降)

- `subscription`操作の実際のパース(型としては存在するが未対応)。
- スキーマ定義言語(SDL)のパーサー。
- 実行エンジン(execution/resolver)。validationは上記の通りv0.3.0で
  第一段を実装済み。
- `poem`フィーチャの本実装(現状はアダプタ層のスタブと設計メモのみ)。

## 使用例

```rust
use rgraphql::parse;

let doc = parse(r#"
    query HeroName {
        empireHero: hero(id: 1000, episode: JEDI) {
            name
            friends { name }
        }
    }
"#).unwrap();

assert_eq!(doc.operations.len(), 1);
let op = &doc.operations[0];
assert_eq!(op.name.as_deref(), Some("HeroName"));
```

トークナイザ単体:

```rust
use rgraphql::{tokenize, TokenKind};

let tokens = tokenize("{ hero }").unwrap();
assert_eq!(tokens.first().unwrap().kind, TokenKind::BraceL);
```

## ビルド・テスト

```bash
cargo test                 # コアのみ
cargo test --features poem # Poem統合アダプタ(スタブ)を含む
```

外部パーシングクレートへの依存が無いため、追加のセットアップは不要。

## 位置づけ(4層4重の通信+DB)

RGraphQLは、RPoem/RCosmoを基盤とするサーバー側スタック上で
「GraphQL公開層」を担う:
`open-web-server → RPoem/RCosmo → aruaru-db/PostgreSQL → open-raid-z`。
コアはこのどの層にも依存しない純粋ライブラリとし、Poem統合(HTTP公開)は
`poem`フィーチャのアダプタ層でのみRPoem/Poemと接続する。

## 関連プロジェクト

- [RFrontEnd](https://github.com/aon-co-jp/RFrontEnd) 傘下の各プロジェクト
  (RHTML/RCSS/RTypeScript/RJSON/RReact) — 本クレートはこのエコシステムの
  「既存実装を流用せず一から再実装する」方針を共有する。
- [RPoem](https://github.com/aon-co-jp/RPoem) — サーバー側実行基盤。
  `poem`フィーチャの本実装で接続予定。

## ライセンス

Apache-2.0 OR MIT

## API 方針: REST 全廃・gRPC / Connect へ (2026-11-04)

### 日本語
- **REST は全廃する。新しい API は REST で作らない。** ブラウザや `curl` から呼ぶ API は **Connect の単純呼び出し**(`POST /<パッケージ>.<Service>/<Method>`、`content-type: application/json`、本文は JSON)、サービス間は **gRPC / gRPC-web**(RPoem の管理API `open_runo.admin.v1.Admin` と同じ流儀)。ストリーミングは、必要になってから。
- **既存の REST は、触るときに順に移す。** 移したら旧 REST の経路は消す。移行が終わるまで残るが、**新規には追加しない**。
- **JSON は RS-JSON に統一**する(読み込みは `from_str_secure` / `parse_secure` でサイズ・深さ・重複キーの上限つき、書き出しは `rust_json::fast`)。ブラウザの WASM に `prost` は入れない(サイズが増えるため)。
- **例外(自分たちの API ではなく、相手の規格が決めているもの)**: ACME、OAuth/OIDC のコールバック、第三者(GitHub など)からの Webhook、ヘルスチェック、静的ファイル配信、OTLP など。これらは規格どおりの HTTP を受ける。
- 参考実装: `RPoem`(`open_runo.admin.v1.Admin`)、`open-easy-web`(`server/src/connect.rs`、`proto/open_easy_web/admin/v1/admin.proto`)。
- ※ このリポジトリのコードが既に方針どおりかどうかは、各リポジトリの現状による。この節は方針の記録であり、コードの変更を意味しない。

### English
- **REST is being abolished. New APIs are not built as REST.** APIs called from browsers or `curl` use **Connect unary calls** (`POST /<package>.<Service>/<Method>`, `content-type: application/json`, JSON body); service-to-service calls use **gRPC / gRPC-web** (the same style as RPoem's admin API `open_runo.admin.v1.Admin`). Streaming only when it is actually needed.
- **Existing REST endpoints are migrated one by one, when they are touched.** The old REST route is removed once migrated. Until migration is finished they remain, but **no new ones are added**.
- **JSON is unified on RS-JSON**: reading with `from_str_secure` / `parse_secure` (size, depth and duplicate-key limits), writing with `rust_json::fast`. `prost` is not put into browser WASM (it would grow the binary).
- **Exceptions (shapes dictated by someone else's standard, not our own API)**: ACME, OAuth/OIDC callbacks, webhooks from third parties (e.g. GitHub), health checks, static file serving, OTLP, and so on. These accept plain HTTP as the standard requires.
- Reference implementations: `RPoem` (`open_runo.admin.v1.Admin`) and `open-easy-web` (`server/src/connect.rs`, `proto/open_easy_web/admin/v1/admin.proto`).
- Note: whether the code in a given repository already follows this policy depends on that repository. This section records the policy; it does not mean the code has been changed.

### Deutsch (KI-Übersetzung, noch nicht von Muttersprachlern geprüft)
- **REST wird abgeschafft. Neue APIs werden nicht als REST gebaut.** APIs, die aus dem Browser oder mit `curl` aufgerufen werden, nutzen **Connect-Unary-Aufrufe** (`POST /<Paket>.<Service>/<Methode>`, `content-type: application/json`, JSON-Body); zwischen Diensten gilt **gRPC / gRPC-web** (wie bei der Admin-API von RPoem, `open_runo.admin.v1.Admin`). Streaming erst, wenn es wirklich nötig ist.
- **Bestehende REST-Endpunkte werden einzeln migriert, sobald man sie anfasst.** Nach der Migration wird die alte REST-Route entfernt. Bis dahin bleiben sie bestehen, es werden aber **keine neuen hinzugefügt**.
- **JSON wird auf RS-JSON vereinheitlicht**: Lesen mit `from_str_secure` / `parse_secure` (Grenzen für Größe, Tiefe und doppelte Schlüssel), Schreiben mit `rust_json::fast`. `prost` kommt nicht in Browser-WASM (die Datei würde größer).
- **Ausnahmen (Formate, die ein fremder Standard vorgibt, nicht unsere eigene API)**: ACME, OAuth/OIDC-Callbacks, Webhooks von Dritten (z. B. GitHub), Health-Checks, statische Dateien, OTLP usw. Diese nehmen schlichtes HTTP entgegen, wie es der Standard verlangt.
- Referenzimplementierungen: `RPoem` (`open_runo.admin.v1.Admin`) und `open-easy-web` (`server/src/connect.rs`, `proto/open_easy_web/admin/v1/admin.proto`).
- Hinweis: Ob der Code eines Repositorys diese Richtlinie schon befolgt, hängt vom jeweiligen Repository ab. Dieser Abschnitt hält die Richtlinie fest; er bedeutet nicht, dass der Code bereits geändert wurde.

### Русский (машинный перевод, носителями языка ещё не проверен)
- **REST полностью отменяется. Новые API не создаются как REST.** API, вызываемые из браузера или через `curl`, используют **унарные вызовы Connect** (`POST /<пакет>.<Service>/<Method>`, `content-type: application/json`, тело — JSON); между сервисами — **gRPC / gRPC-web** (так же, как административный API RPoem, `open_runo.admin.v1.Admin`). Потоковая передача — только когда она действительно нужна.
- **Существующие REST-эндпоинты переносятся по одному, когда до них доходят руки.** После переноса старый REST-маршрут удаляется. До завершения переноса они остаются, но **новые не добавляются**.
- **JSON унифицируется на RS-JSON**: чтение через `from_str_secure` / `parse_secure` (ограничения на размер, глубину и повторяющиеся ключи), запись через `rust_json::fast`. `prost` не добавляется в браузерный WASM (размер вырос бы).
- **Исключения (форматы, заданные чужим стандартом, а не наш собственный API)**: ACME, обратные вызовы OAuth/OIDC, вебхуки сторонних сервисов (например, GitHub), проверки работоспособности, раздача статических файлов, OTLP и т. п. Они принимают обычный HTTP, как требует стандарт.
- Эталонные реализации: `RPoem` (`open_runo.admin.v1.Admin`) и `open-easy-web` (`server/src/connect.rs`, `proto/open_easy_web/admin/v1/admin.proto`).
- Примечание: соответствует ли код конкретного репозитория этой политике, зависит от самого репозитория. Этот раздел фиксирует политику и не означает, что код уже изменён.
