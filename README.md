# oil-refining

**製油所・装置構成・稼働率・停止・歩留まりを扱うと宣言した actor の descriptor と、
その書き込みを止める deny-by-default gate。製油所の実データも、それを読むグラフも、
ここには無い。実装ではない。**

**そして、この actor は既に後継に置き換えられている。**
`cloud-itonami/kamado`（竈）の `manifest.edn` が
`:actor/supersedes ["oil-refining"]` を宣言しており、
この repo の [MIGRATION-NOTES.md](MIGRATION-NOTES.md) は
`legacy · superseded-not-deleted · do not extend` と自称している。
**新しい仕事をここに載せない。** 行き先は `orgs/cloud-itonami/kamado`。

`oil-*` は 7 本ある。うち 6 本（`upstream` / `midstream` / **`refining`** /
`trading` / `shipping` / `distribution`）が segment の実体を扱う側で、`oil-coverage`
だけがそれらを上から測る meta actor。**この repo は 6 本のうち、原油を製品に変える
区間**を担当すると宣言している。そして **7 本のうち `MIGRATION-NOTES.md` を持ち、
後継が名指しで存在するのはこの 1 本だけ**（2026-08-09 実測）。

| | ここにあるか |
|---|---|
| actor が**何を名乗り、何を要求し、どの pipeline を持つと宣言しているか** | **ある**（`actor-manifest.jsonld` 8,553 B / `.well-known/did.json` 730 B） |
| **gate**（attestation が 7 つ揃わなければ effect を 1 つも出さない判断） | **ある**（`src/oil_refining/murakumo.cljc`、226 行 / 8,832 B） |
| 製油所を数えるグラフ、cron を撃つ scheduler、XRPC を受ける server | **無い** |
| 製油所・装置・停止の実データ | **無い** |
| **歩留まり（yield）のデータモデル** | **無い**（名前と散文にしか存在しない。後述） |

**ここには動くサービスは無い。** `cell-plan` が返すのは「書くとしたら何をどこに
書くか」という**計画**であって、書き込みそのものではない。`:effects` は
`{:op :mst/put-record ...}` という data であり、それを実行する者はこの repo に居ない。

経緯は [docs/adr/0001-superseded-descriptor-snapshot.md](docs/adr/0001-superseded-descriptor-snapshot.md)。
手順は [docs/operator-quickstart.md](docs/operator-quickstart.md)。

## 後継はどこにあり、この repo は何を保持しているのか

| | この repo（`oil-refining`） | 後継（`cloud-itonami/kamado` 竈） |
|---|---|---|
| 正本の substrate | RisingWave / Cypher（`MATCH (r:Refinery) …`） | kotoba Datom log（`graph.write` を禁じる不変条件 `N6`） |
| 実装 | 無い（descriptor + gate のみ） | `src/kamado/methods/` に 7 本の `.cljc`、`test/` あり |
| 成熟度（2026-08-09 の計測） | **414 bp** | **5,518 bp** |
| supersede 関係の宣言 | `MIGRATION-NOTES.md`（散文） | `manifest.edn` `:actor/supersedes ["oil-refining"]`（機械可読） |

**関係は片側だけの主張ではなく相互に宣言されている。** kamado 側で
`oil-refining` に言及するファイルは **9 本**（`manifest.edn` / `wire/manifest.jsonld` /
`CLAUDE.md` / `README.md` / `data/seed.edn` / `src/kamado/methods/analyze.cljc` /
`src/kamado/methods/ingest.cljc` / `test/kamado/methods/test_ingest.cljc` /
`wire/ingest/legacy-oil-refining-export.sample.json`）で、いずれも
「legacy `oil-refining` の kotoba-native な後継」という位置づけで書かれている。

### ⚠ `MIGRATION-NOTES.md` 自身が古い

置いたまま直していないが、**そこに書かれた実行手順は現在のどのパスにも当たらない**
（2026-08-09 実測）:

| MIGRATION-NOTES.md の記述 | 現在 |
|---|---|
| `20-actors/kamado/methods/ingest.py` | `orgs/cloud-itonami/kamado/src/kamado/methods/ingest.cljc`。**kamado に `.py` は 1 本も無い** |
| `20-actors/kamado/data/ingest/legacy-oil-refining-export.sample.json` | `wire/ingest/legacy-oil-refining-export.sample.json` |
| `cd 20-actors/kamado/methods && python3 ingest.py …` | **走らない**（monorepo 時代のパス。kamado は独立 repo になっている） |

つまり「後継が在る」という主張は正しく、「そこへ移る手順」は**もう踏めない**。
ADR-2607173000（script host は nbb のみ）に沿った `bb`→`nbb`/`cljc` 移行が
kamado 側で先に済んでおり、この notes がそれに追随していない。
**この repo は superseded 側なので、notes を書き換えるのではなく古いと明示する**
（正しい手順を持つのは kamado 側であり、そちらを直すのはこの repo の権限ではない）。

## 名前が 2 つあり、解決するのは片方だけ

この repo は自分を 2 通りに名乗っている。**同じ actor の別表記ではなく、
一方は DNS に存在しない。**

| 出所 | 名乗り | 2026-08-09 実測 |
|---|---|---|
| `actor-manifest.jsonld` の `@id`<br>`src/oil_refining/murakumo.cljc` の `actor-did` | `did:web:oil-refining.etzhayyim.com` | **解決しない**。`oil-refining.etzhayyim.com` に A/AAAA レコードが無く、`curl` は `000`（接続前に失敗） |
| `.well-known/did.json` の `id` | `did:web:etzhayyim.com:actor:oil-refining` | **解決する**。`https://etzhayyim.com/actor/oil-refining/did.json` が `200` |

**gate が名乗るのは解決しない方**である（`murakumo.cljc:6`）。effect の
`:actor` フィールドに載るのもそちら。これは既知の不整合で、直していない —
どちらを正とするかは etzhayyim 側の identity 決定であって、この snapshot が
勝手に決めてよいことではない。

### repo の `.well-known/did.json` は live の写しではない

`.nojekyll` が置かれているので GitHub Pages 配信を意図していたと読めるが、
2026-08-09 時点で `etzhayyim.github.io/com-etzhayyim-oil-refining` も
`cloud-itonami.github.io/oil-refining` も **404**。

解決する方の URL が実際に返す文書は、**この repo の中の did.json とは別物**である
（`jq -S` して `diff` すると 5 か所相違、`diff` は exit 1）:

| | repo の `.well-known/did.json` | live（`etzhayyim.com/actor/oil-refining/did.json`） |
|---|---|---|
| crypto suite context | `ed25519-2020` | `jws-2020` |
| `alsoKnownAs` | 4 件（at:// · github · rad: · github.io） | **空配列** |
| `verificationMethod` | **フィールド自体が無い** | 空配列（`_meta` に「ERC725 mirror pending」と注記） |
| PDS endpoint | `https://pds.etzhayyim.com` → **530**（`/xrpc/_health`） | `https://pds.aozora.app` → `/xrpc/_health` が **200** |
| 2 つ目の service | `#aozora`（AozoraAppView、`https://aozora.app`） | `#xrpc-libp2p`（`/dnsaddr/etzhayyim.com/p2p/12D3KooW…`） |

### ⚠ live の identity 面は、この actor が superseded であることを一切示さない

live の `_meta` を後継と並べると**区別が付かない**（2026-08-09 実測）:

```
oil-refining   tier-b  r0  com.etzhayyim.oil-refining
kamado         tier-b  r0  com.etzhayyim.kamado
```

`oil-*` 7 本 + `kamado` の 8 本すべてが `tier-b` / `r0` を返す。**したがって
`MIGRATION-NOTES.md` の「Not in the Tier-B roster」を live の DID document は
支持していない。** これは notes が誤っているという意味ではなく、
`_meta.kind` が apex resolver の付ける値であって「Tier-B roster」という別の台帳とは
出所が違う、ということしか実測からは言えない。**どちらが正かは決めない。**
言えるのは「**identity 面だけを見て supersede を検出することはできない**」。

### live の DID document は、manifest ではなく **gate** の語彙を支持している

live 側だけが持つ `_meta.primaryLexicon` は **`com.etzhayyim.oil-refining`** であり、
これは `murakumo.cljc` の `collection` 関数が組み立てる綴りと**一致する**。
manifest 側の `com.etzhayyim.apps.oilRefining.*` ではない。8 本すべて同じ形
（`com.etzhayyim.<repo 名>`。後継 kamado も `com.etzhayyim.kamado`）。

後述する「gate と manifest の語彙が交わらない」問題は、したがって
**gate 側だけの scaffold 由来の綴り崩れ**とは読めない。**どちらが正かはこの
snapshot が決めない。**

## 何を扱うと宣言しているか

`actor-manifest.jsonld` は **8 pipeline** を宣言する（cron 2 / subscribeRepos 1 / xrpc 5）。

| trigger | 何を宣言しているか |
|---|---|
| cron `0 */8 * * *`（5 step） | **報告する。** 国別の製油所数と処理能力(bpd) → 装置種別ごとの数 → planned/active の停止 を数え、`agent.chat` に要約させ、`derive:social` で社会面に 1 本流す |
| cron `0 */6 * * *`（3 step） | **自分の被覆率を書き戻す。** 自 DID のノード総数と collection 別内訳を数え、`ActorCoverageSnapshot` を MERGE する |
| subscribeRepos `oilShipping.cargo`（1 step） | cargo が来たら、稼働中の製油所を最大 20 件引く |
| xrpc `…registry.getRefinery` | 1 つの製油所を返す |
| xrpc `…registry.listRefineries` | 製油所を最大 50 件返す |
| xrpc `…registry.listUnits` | 装置を最大 50 件返す |
| xrpc `…analytics.getYieldPressure` | 国コード指定で製油所を最大 50 件返す（**歩留まりは返さない**、後述） |
| xrpc `…health` | 製油所の総数を 1 行返す |

宣言された 4 つの sub-actor（`actors[]`）は、**4 つとも対応する step がある**:
`registry:refinery`（getRefinery / listRefineries / health / 8h cron の `refineryStats`）/
`units:conversion`（listUnits / `unitStats`）/
`analytics:yield`（getYieldPressure）/
`risk:outage`（8h cron の `outages`）。

読むグラフラベルは 3 つ（`Refinery` 6 か所・`RefineryUnit` 2 か所・`RefineryOutage`
1 か所）、書くのは `ActorCoverageSnapshot` 1 か所。使う `fn` は `graph.query` 11 /
`graph.write` 1 / `agent.chat` 1 / `derive:social` 1 で、**宣言された 5 capability の
うち `agent.invoke` はどの step でも使われていない。**

**エッジ型は 1 つも無い。** 兄弟の `oil-midstream` は `flowsTo` / `constrainedBy` を
持つが、この repo の Cypher は 1 本もエッジを辿らない —— 原油→製品の変換を扱うと
宣言しながら、繋がりを表す構造が宣言の中に無い。

### ⚠ 「Product Yield Intelligence」を名乗るが、歩留まりに触る query は 1 本も無い

`displayName` は `Oil Refining & Product Yield Intelligence`、sub-actor に
`analytics:yield`（"Crude-to-product yield pressure and output balance"）、
XRPC に `analytics.getYieldPressure`、購読に `…oilRefining.yieldSnapshot`。
**manifest 全体で「yield」は 11 個の JSON leaf に現れる。そのうち Cypher の中は 0 個。**

`getYieldPressure` が実際に発行する query はこれである:

```cypher
MATCH (r:Refinery {country_code: $input.countryCode})
RETURN r.refinery_code AS refinery, r.throughput_bpd AS throughput, r.status AS status LIMIT 50
```

返すのは**処理能力**（`throughput_bpd`）であって歩留まりではない。
step 全体が参照するプロパティは 18 種類あるが、**製品・歩留まり・収率に相当する
ものは 1 つも無い**（`r.status` 4 / `r.throughput_bpd` 3 / `r.vertex_id` 2 /
`r.refinery_code` 2 / `r.country_code` 2 / `u.unit_type` 1 / `o.status` 2 …）。

フリート側にラベルが無いわけでもない —— `oil-coverage` の backbone 集計は
`ProductGrade` を数える対象に挙げている。**それを読むのも書くのも、
`oil-refining` ではない**（`ProductGrade` を言及する manifest は
39 本中 `oil-coverage` 1 本のみ）。

唯一「yield」が pipeline の中に現れるのは、8 時間 cron の `agent.chat` に渡す
散文プロンプト（`… likely product yield pressure.`）である。**歩留まりは
LLM に推測させる対象であって、この actor が観測するデータではない。**
後継の kamado が「歩留まりではなく炭素収支（origin / process / fate）を
数える」設計に振り替えたのは、この空白と無関係ではない。

## 購読は 5 件、そのうち handler があるのは 1 件

`triggers.subscribeRepos.collections` は 5 collection を購読すると宣言するが、
`trigger.type == "subscribeRepos"` の pipeline は **1 本しか無い**
（`oilShipping.cargo`）。残り 4 件は、受け取っても実行するものが無い。
これは 7 本共通の形である:

| repo | 宣言した購読 | handler pipeline |
|---|---|---|
| oil-upstream | 5 | 1 |
| oil-midstream | 5 | 1 |
| **oil-refining** | **5** | **1** |
| oil-trading | 4 | 1 |
| oil-shipping | 6 | 1 |
| oil-distribution | 4 | 1 |
| oil-coverage | **12** | **0** |

購読 4 件のうち `…oilRefining.yieldSnapshot` は、上記のとおり**歩留まりを扱う
唯一の受け口**でありながら handler も producer も無い。

### そして、読むノードを書く者もフリート内に居ない

cloud-itonami に `actor-manifest.jsonld` を持つ repo は **39 本**あり、その中に
`graph.write` step は **70 個**ある。この 70 個を全部並べても、
`Refinery` / `RefineryUnit` / `RefineryOutage` を**ラベルとして書くものは 1 つも無い**。

読む側も **`oil-refining` の 9 step だけ**である。兄弟の `oil-midstream` は
`oil-coverage` と `oil-shipping` からも読まれていたが、**この repo のラベルは
他のどの actor もラベルとしては触らない。**

#### この計測は正規表現を 2 方向に間違える（両方とも実測した）

| | 素朴な `test("Refinery")` | ラベル形 `test("\\([a-z]+:Refinery")` |
|---|---|---|
| **偽陽性** | `rare-earth-coverage` の `graph.write` が 1 件当たる | 当たらない |
| **偽陰性** | — | `oil-coverage` の 2 step を落とす |

- **偽陽性**: `rare-earth-coverage` が書くのは `(n:RareEarthActor …)` であり、
  当たっているのは `role: 'refinery-recycler'` / `'refinery-project'` /
  `displayName: 'Eneabba Rare Earths Refinery'` という**値の中の文字列**。
  ここで止めれば「producer が居る」と書いてしまう。
- **偽陰性**: `oil-coverage` は
  `UNWIND ['OilCompany', …, 'Refinery', …] AS lbl … WHERE lbl IN labels(n)` で
  **ラベルを文字列として**数えている。これは実際の読み取りだが、ラベル形の
  正規表現には映らない。

したがって正確には: **ラベルを書く者は 0、ラベル形で読むのは `oil-refining` の
9 step、ラベル名を文字列として数えるのは `oil-coverage` の 2 step。**

購読先 5 件それぞれの供給元:

| 購読先 collection | フリート内で言及する repo | producer か |
|---|---|---|
| `…apps.oilRefining.refinery` | この repo のみ（購読側） | **無い** |
| `…apps.oilRefining.unit` | この repo のみ（購読側） | **無い** |
| `…apps.oilRefining.outage` | この repo のみ（購読側） | **無い** |
| `…apps.oilRefining.yieldSnapshot` | この repo のみ（購読側） | **無い** |
| `…apps.oilShipping.cargo` | oil-* 5 本（すべて購読側） | **無い** |

`oil-coverage` はこの repo の `…apps.oilRefining.coverageSnapshot` を購読しており、
**測られる側**としての接続だけが両方向で噛み合っている。ただしこの repo が
6 時間 cron で行うのは `MERGE (c:ActorCoverageSnapshot …)` という**グラフノードへの
書き込み**であって、PDS の repo collection への record 公開ではない ——
`subscribeRepos` が運ぶのは後者なので、**この 1 本も実際には繋がらない。**

## 兄弟 6 本との違いはどこか

`actors=4` / `pipelines=8` / `handlers=1` は 6 本すべて同じ（`oil-coverage` だけ
`actors=6` / `pipelines=5` / `handlers=0` で形からして別物）。**購読数と `nanoid`、
そして後継の有無が違う**:

| repo | nanoid | 宣言した購読 | README | 後継 |
|---|---|---|---|---|
| oil-upstream | `01lupstr` | 5 | 0 B | — |
| oil-midstream | `01lm1dst` | 5 | 16,427 B | — |
| **oil-refining** | **`01lr3f1n`** | **5** | **この文書** | **`kamado`** |
| oil-trading | `01ltrad3` | 4 | 0 B | — |
| oil-shipping | `01l5h1p0` | 6 | 0 B | — |
| oil-distribution | `01ld1str` | 4 | 12,109 B | — |
| oil-coverage | `011c0v3r` | 12 | 9,124 B | — |

購読数が同じなので、gate cell 数も `oil-midstream` と同じ 16 になる。

## gate は何を止めるか

`src/oil_refining/murakumo.cljc` は **16 cell × 7 gate** の deny-by-default。
7 つの attestation が 1 つでも欠けると `:status :blocked` で `:effects` は空になる
（実測: 6/7 揃えても `:blocked`、`all-cell-plans` は 16 cell 全部 blocked で総 effect 数 0。
7/7 揃えると 16 cell すべて `:ready` で effect 16）。

7 gate: `:council-charter-attestation` `:no-platform-held-key-baseline`
`:no-probing-baseline` `:murakumo-only-inference-baseline`
`:did-primary-baseline` `:append-only-gate-baseline`
`:kotoba-only-substrate-baseline`

16 という数は manifest から導ける: **5 xrpc + 2 `requiredCollections` +
4 `requiredLoops` + 5 `subscribeRepos` = 16**。

実行して確かめられる（[quickstart](docs/operator-quickstart.md) の手順 4）。

**この gate は superseded 宣言を知らない。** 7 つ揃えば 16 cell 分の
`:mst/put-record` 計画を出す。`MIGRATION-NOTES.md` の
`do not extend` を強制する仕組みは、この repo のコードのどこにも無い。

### ⚠ gate が書く collection は、manifest が宣言する collection と 1 つも一致しない

`murakumo.cljc` の `collection` 関数は `com.etzhayyim.oil-refining.<name>` を
組み立てる。manifest 側の語彙は `com.etzhayyim.apps.oilRefining.<name>` である。
`apps.` の有無と segment のハイフン/キャメルが違い、**16 個の交わりは空**:

```
gate     com.etzhayyim.oil-refining.refinery
manifest com.etzhayyim.apps.oilRefining.refinery
```

（上で見たとおり、live の DID document が支持しているのは **gate 側**の綴りである。）

さらに gate は 2 種類の「読むもの」を書き込み先に変えている:

- **XRPC の *メソッド* 名を collection として扱う。** `getrefinery` は manifest では
  「読む API」だが、gate では `com.etzhayyim.oil-refining.getrefinery` へ
  `:mst/put-record` する計画になる。
- **他 actor の collection を自分の名前空間に付け替える。** manifest が購読する
  `com.etzhayyim.apps.oilShipping.cargo` は、gate では
  `com.etzhayyim.oil-refining.cargo` という**自分が書く先**になる。

`requiredLoops` の 4 つ（`shinka` / `koji` / `kyumei` / `domain-knowledge`）も
同じ規則で collection 化されている。いずれも scaffold 生成器の産物と読めるが、
正しい語彙を決めるのは lexicon 側の権限なので**直していない**。

⚠ **namespace が `oil_refining.murakumo`（アンダースコア）である。** Clojure の
慣習では `oil-refining.murakumo` と書いてファイル側を `oil_refining/` にする。
現状でも load はできるが、`(require '[oil-refining.murakumo])` は**通らない** —
`oil_refining` と綴る必要がある。

## live の PDS には record が 1 件も観測できない（ただし証拠能力は弱い）

live の did.json が指す `pds.aozora.app` に `com.atproto.repo.describeRepo` を投げると
`{"collections":[]}` が返る。**ただしこの endpoint は存在しない DID にも同じ形で
200 を返す**（対照実験: `did:web:etzhayyim.com:actor:not-a-real-actor-xyz` でも
`{"collections":[],"handleIsCorrect":false}`）。

したがってこの観測から言えるのは **「record は 1 件も観測できない」**までで、
「登録済みだが空」と「そもそも未登録」は**この endpoint では区別できない**。
200 を registration の証拠として読まないこと。

## 2026-06-24 の snapshot であること

etzhayyim monorepo の `20-actors/oil-refining` から descriptor だけを写した
snapshot。`actor-manifest.jsonld` が `runtime: k8s-langserver` /
`edge: sveltekit-proxy` / `legacyExecutionTier: T1` / `heartbeatRequired: true` と
宣言していても、**その runtime も edge も heartbeat もここには無い。**
`complianceDocs` が指す 2 本（`90-docs/rules/compliance/…` / `90-docs/platform/…`）も、
この repo には存在しない**元 monorepo のパス**である。

`actor-manifest.test.ts` は **走らない** — `package.json` も vitest も無く、
`node_modules` も無い。vitest 前提で書かれた 11 の `it(` / 16 の `expect(` が、
実行されないまま置かれている。`.ts` なのでこの workspace の nbb 経路にも載らない。

commit は 4 本だけ:

| commit | 日付 | 内容 |
|---|---|---|
| `16e3fc8` | 2026-06-24 | snapshot（manifest / did.json / NOTICE / test.ts / **MIGRATION-NOTES.md**） |
| `fc616a7` | 2026-07-02 | did:web を `etzhayyim.com` scheme へ移行 |
| `3e5c3a5` | 2026-07-18 | murakumo WIP の rescue（`src/oil_refining/murakumo.cljc`） |
| `2563dbf` | 2026-07-27 | 上の rescue branch を main へ merge |

**`MIGRATION-NOTES.md` は最初の snapshot から在った** —— つまりこの repo は
「起こされた時点で既に superseded だった」。

## 既知のギャップ（この反復で埋めていないもの）

- **test が無い。** 上の表の数（pipeline 8 / sub-actor 4 / cell 16 / gate 7 /
  label 3 / 購読 5・handler 1 / yield を含む Cypher 0）と 2 つの DID は、いま
  **この README が主張しているだけ**で、実体が変わっても赤くならない。
  `marine-insurance` は `test/…/docs_test.cljs` でこれを固定している —— 同じものが
  ここにも要る。
- **west pin が遅れていた。** superproject の pin は `fc616a7`（2026-07-02）で、
  `src/oil_refining/murakumo.cljc` を含む `2563dbf` を指していなかった。この
  README を書く時点で main に合わせている。
- **`MIGRATION-NOTES.md` を直していない**（古いパスと `python3` 手順のまま）。
  正しい手順を持つのは kamado 側であり、そこを直すのはこの repo の権限ではない。
- **supersede を機械が検出できる形にしていない。** 現状 `:actor/supersedes` は
  kamado 側にしか無く、この repo 側には散文の `MIGRATION-NOTES.md` しか無い。
  live の identity 面（`_meta.kind`/`status`）は後継と区別が付かない。
- identity の不整合、collection 語彙の不一致、購読 handler の欠落 4 件、
  歩留まりモデルの不在は、いずれも**記録のみで直していない**。
- **同型の兄弟 3 本**（`oil-upstream` / `oil-trading` / `oil-shipping`）は
  README 0 バイトのまま。今回 1 本だけ。
