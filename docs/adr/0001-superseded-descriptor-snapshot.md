# ADR-0001 — この repo は superseded な descriptor snapshot であって executor ではない

- status: accepted
- date: 2026-08-09
- 対象: `cloud-itonami/oil-refining`
- 上位: superproject ADR-2608080000（成熟度を 1 段ずつ上げる loop）/
  ADR-2608052000（7 軸の測り方）/ ADR-2606231200（sovereign actor repo pilot）
- 関連: `cloud-itonami/kamado` の ADR-2606051500（この actor の後継）
- 先例: `cloud-itonami/oil-midstream` の ADR-0001（同じ形の repo、1 周前）

## 文脈

superproject の成熟度 loop（skill `itonami-maturity-improve`）が、この repo を
`axis-docs`（`own = 0.041` = 414 bp、README 0 バイト）で名指しした。同点の兄弟が
5 本並んでおり（`:ranking-is-flat? true`）、前周は `oil-midstream` を同じ軸で
処理している。

この repo は 2026-06-24 に etzhayyim monorepo の `20-actors/oil-refining` から
**descriptor だけを写した** snapshot として起こされた。commit は 4 本しか無い:

| commit | 日付 | 内容 |
|---|---|---|
| `16e3fc8` | 2026-06-24 | snapshot（manifest / did.json / NOTICE / test.ts / **MIGRATION-NOTES.md**） |
| `fc616a7` | 2026-07-02 | did:web を `etzhayyim.com` scheme へ移行 |
| `3e5c3a5` | 2026-07-18 | murakumo WIP の rescue（`src/oil_refining/murakumo.cljc`） |
| `2563dbf` | 2026-07-27 | 上の rescue branch を main へ merge |

## 問題

README が 0 バイトだったため、この repo を読む者は次の 4 つを区別できなかった:

1. **宣言**（manifest が「こう動く」と書いていること）
2. **判断**（gate が実際に実行できること）
3. **実装**（どこにも無いもの）
4. **この actor が既に後継に置き換えられていること**

4 が先例（`oil-midstream` / `oil-distribution`）に無い、この repo 固有の問題である。
`MIGRATION-NOTES.md` は最初の snapshot から在り `do not extend` と書いているが、
**README が無いので GitHub の repo 画面には何も表示されない** —— 訪問者はファイル
一覧を見て `actor-manifest.jsonld` を開き、`runtime: k8s-langserver` /
`heartbeatRequired: true` / cron trigger を読む。そこまでで「稼働中の actor」と
読むのに十分な材料が揃い、`MIGRATION-NOTES.md` をクリックする理由は無い。

## 決定

### 1. README は「後継が在る」から書く

`axis-docs` は README のバイト数を測るが、**バイト数を目的にしない**。この repo で
読み手が最初に知るべきことは「動くサービスは無い」ではなく
**「ここは終わっていて、続きは `kamado` に在る」**であり、それを冒頭 12 行に置く。

### 2. supersede の主張は、必ず後継側から裏を取ってから書く

`MIGRATION-NOTES.md` は当事者の自己申告なので、それだけでは「本当に後継が在る」
根拠にならない。後継側を実測した:

- `kamado/manifest.edn` に **`:actor/supersedes ["oil-refining"]`**（機械可読、相互宣言）
- `oil-refining` に言及する kamado のファイルは **9 本**
- kamado は `src/kamado/methods/` に 7 本の `.cljc` と `test/` を持つ実装
- 成熟度は **5,518 bp**（この repo は 414 bp）

**片側の散文ではなく、両側の機械可読な宣言が噛み合っていることを確かめてから
「superseded」と書く。**

### 3. `MIGRATION-NOTES.md` は「古い」と明示するが、書き換えない

notes の**主張**（後継は kamado）は正しいが、**手順**は 1 つも踏めない:

| notes の記述 | 実測（2026-08-09） |
|---|---|
| `20-actors/kamado/methods/ingest.py` | 存在しない。**kamado に `.py` は 0 本**（`ingest.cljc` に移行済み） |
| `20-actors/kamado/data/ingest/…sample.json` | `wire/ingest/…sample.json` |
| `cd 20-actors/kamado/methods` | kamado は独立 repo。monorepo 時代のパス |

ADR-2607173000（script host は nbb のみ）の `bb`→`nbb`/`cljc` 移行が kamado 側で
先に済み、notes が追随していない。**この repo は superseded 側なので notes を
書き換えず、古いことを README と quickstart に書く** —— 正しい手順を持つのは
kamado 側であり、そこを直すのはこの repo の権限ではない。

**「後継が在る」と「そこへ移れる」は別の主張である。** 前者だけを検証して後者を
書くと、踏めない手順を案内することになる。

### 4. 主張はすべて実行可能な手順に還元し、書いたあとに全部踏む

README が出す数と identity の状態は、すべて `docs/operator-quickstart.md` の
8 手順で再現できる形にした。**書いたあとに全 bash ブロックを機械的に抽出して
verbatim 実行し、貼った出力と突き合わせた。**

内訳（bash ブロック 33 本）: **29 本が貼った出力と完全一致**（gate probe の
`.cljs` を文書中の clojure ブロックから書き出して走らせたものを含む）/
2 本は貼った出力が無い（`diff` は散文で説明、`grep -c` は行内コメント）/
1 本は west pin を書き換える破壊的手順なので実行から外し、**実際の pin 前進で
別途踏んだ**（結果節）/ 1 本（`grep -rl ../kamado`）は検証用に組んだ symlink
環境で `grep -r` が symlink 先へ降りないため一致せず、**実レイアウトの
`orgs/cloud-itonami/oil-refining` から実行して 9 行一致を確認**した。

これは形式ではなく検査として働いた —— 踏んだ結果、次の 4 つが実際に間違っていた:

1. kamado の `src/kamado/methods/` の `ls` 出力を**列組みで書いていた**（実際は
   1 行 1 ファイル）。しかも実在しない `do_not_use.txt` が混ざっていた
2. `grep -n` の行番号（41/42/49/50 → 実際は 39/42/43/50）
3. kamado の言及ファイルを 8 本と書いていた（実際は 9 本、`ingest.cljc` 落ち）
4. `rare-earth-coverage` の role 値が 2 つと書いていた（実際は 3 つ、
   `role:'refinery'` 落ち）

**4 つとも「出力を書いた」ことによる誤りで、コマンドを打っていれば起きない。**
先例の ADR-0001 が「踏まなければ動かないコマンドを実行結果として貼っていた」と
書いたのと同じ失敗を、別の形（コマンドは動くが出力が違う）で再現した。

### 5. 先例（oil-midstream）を雛形として使うが、値は必ず測り直す

7 本は同じ scaffold から出ているので構造は似るが、**数と性質は違う**。実測で
違った箇所:

| | oil-midstream | **oil-refining** |
|---|---|---|
| 後継の有無 | 無い | **`kamado`（`:actor/supersedes` で相互宣言）** |
| `MIGRATION-NOTES.md` | 無い | **在る（7 本で唯一）** |
| グラフラベル | `OilPipeline` / `OilTerminal` の 2 つ | **`Refinery` / `RefineryUnit` / `RefineryOutage` の 3 つ** |
| エッジ型 | 2 種（`flowsTo` / `constrainedBy`） | **0 種** |
| 他 actor からラベルを読まれるか | `oil-coverage` 2 / `oil-shipping` 1 | **どの actor もラベル形では読まない** |
| 購読先の producer | 見かけ上 1（`port-actor`）だが実質 0 | **見かけ上も 0** |
| 名前と中身の乖離 | — | **`Yield Intelligence` を名乗るが yield の query が 0 本** |

先例の文言を写して数だけ差し替えると、この 7 行はすべて間違う。
gate cell 数（16）と購読数（5）だけが `oil-midstream` と同じである。

### 6. 「Product Yield Intelligence」に yield が無いことを、数えて書く

この repo で最も強い所見であり、先例には対応物が無い。

- manifest 全体で `yield` を含む JSON leaf は **11 個**
- そのうち **Cypher（`graph.query` / `graph.write` の `sql` / `template`）の中は 0 個**
- pipeline 内で唯一現れるのは `agent.chat` に渡す**散文プロンプト**
  （`… likely product yield pressure.`）
- `analytics.getYieldPressure` が実際に返すのは `r.throughput_bpd`（**処理能力**）
- step 全体が触るプロパティ 18 種類に、製品・収率に相当するものは 1 つも無い
- フリートに `ProductGrade` ラベルは在るが、言及する manifest は
  `oil-coverage` の 1 本だけで、`oil-refining` は触らない
- 購読 `…oilRefining.yieldSnapshot` には handler も producer も無い

**歩留まりは、この actor が観測するデータではなく LLM に推測させる対象**である。
後継の kamado が「歩留まり」ではなく「炭素収支（origin / process / fate）」を
数える設計に振り替えたことと、この空白は無関係ではない —— ただし
**因果として書かない**（kamado の ADR-2606051500 がそう述べているわけではない）。

### 7. producer 探索の正規表現は 2 方向に間違える。両方を出す

先例の ADR-0001 は「grep だけで止めれば producer が居ると書いてしまう」と
書いた（偽陽性）。この repo で同じ探索をやると、**偽陰性も出た**:

| | 素朴な `test("Refinery")` | ラベル形 `test("\([a-z]+:Refinery")` |
|---|---|---|
| 偽陽性 | `rare-earth-coverage` の `graph.write` 1 件 | 無し |
| 偽陰性 | — | `oil-coverage` の 2 step |

- **偽陽性**: `rare-earth-coverage` が書くのは `(n:RareEarthActor …)` で、
  当たっているのは `role:'refinery'` / `'refinery-project'` /
  `'refinery-recycler'` / `displayName:'Eneabba Rare Earths Refinery'` という
  **値の中の文字列**
- **偽陰性**: `oil-coverage` は
  `UNWIND ['OilCompany', …, 'Refinery', …] AS lbl … WHERE lbl IN labels(n)` で
  **ラベル名を文字列として**数えている。これは実際の読み取りだが、ラベル形の
  正規表現には映らない

したがって結論は「writes が空」だけでは足りず、**「ラベルを書く者は 0、ラベル形で
読むのは `oil-refining` の 9 step、ラベル名を文字列として数えるのは `oil-coverage`
の 2 step」**と 3 つに分けて書く。**片方の正規表現だけを信じない。**

### 8. 2 つの DID の不整合は「記録するが直さない」

- `actor-manifest.jsonld` の `@id` と `murakumo.cljc` の `actor-did` は
  `did:web:oil-refining.etzhayyim.com` を名乗るが、**DNS レコードが無く解決しない**
- `.well-known/did.json` の `id` は `did:web:etzhayyim.com:actor:oil-refining` で、
  **こちらは 200 で解決する**

どちらを正とするかは etzhayyim 側の identity 決定であり、descriptor snapshot が
片方に寄せてよいものではない。repo の did.json を live（5 か所相違）に合わせる
こともしない —— live 側は `_meta` に「ERC725 mirror pending」と書いており、まだ
動いている最中の文書である。

### 9. 「live の identity 面は supersede を示さない」を記録する

live の `_meta` を後継と並べると **8 本すべてが `tier-b` / `r0`** で、
`oil-refining` と `kamado` は区別が付かない。したがって:

- `MIGRATION-NOTES.md` の「Not in the Tier-B roster」を live の DID document は
  **支持していない**。ただし `_meta.kind` は apex resolver が付ける値で、notes の
  言う「roster」とは出所が違う可能性がある。**どちらが正かは決めない**
- 実測から言えるのは **「identity 面だけを見て supersede を検出することはできない」**

観測を書くときは、その観測が何を区別できないかも一緒に書く（決定 11 と同じ規律）。

### 10. live の DID document が gate 側の語彙を支持していることを記録する

`murakumo.cljc` の `collection` は `com.etzhayyim.oil-refining.<name>` を、manifest は
`com.etzhayyim.apps.oilRefining.<name>` を使い、**16 個の交わりは空**である。

live だけが持つ `_meta.primaryLexicon` は **`com.etzhayyim.oil-refining`**、すなわち
**gate 側の綴り**だった。8 本すべてで同じ（`com.etzhayyim.<repo 名>`。後継 kamado も
`com.etzhayyim.kamado`）。

したがって「gate 側だけが崩れている」とは読めない。**それでもどちらを正とするかは
決めない** —— lexicon の決定はこの snapshot の権限ではない。記録に留める。

さらに gate は 2 種類の「読むもの」を書き込み先に変えている（XRPC メソッド名の
collection 化、他 actor の collection の自名前空間への付け替え）。これも記録のみ。

### 11. live PDS の 200 を registration の証拠として使わない

`com.atproto.repo.describeRepo` は `{"collections":[]}` を返すが、**存在しない DID
にも同じ形で 200 を返す**（対照実験を実施）。`listRecords` も架空 DID で
`{"records":[]}`。

したがって言えるのは「record は 1 件も観測できない」までで、「登録済みだが空」と
「未登録」は**この endpoint では区別できない**。

### 12. gate が supersede を強制しないことを書く

`murakumo.cljc` は 7 つの attestation が揃えば 16 cell 分の `:mst/put-record` 計画を
出す。**`MIGRATION-NOTES.md` の `do not extend` を強制する仕組みはコードのどこにも
無い。** これは欠陥の指摘ではなく境界の記述であり、**この反復では直さない**
（gate に supersede チェックを足すのは descriptor snapshot の役割を越える）。

## 測ったこと（2026-08-09、すべて実測）

| 主張 | 実測 |
|---|---|
| 後継 | `cloud-itonami/kamado`。`manifest.edn` に `:actor/supersedes ["oil-refining"]`、言及ファイル 9 本 |
| 成熟度 | oil-refining **414 bp** / kamado **5,518 bp** |
| `MIGRATION-NOTES.md` の手順 | **踏めない**（kamado に `.py` 0 本、パスは monorepo 時代のもの） |
| 7 本中 `MIGRATION-NOTES.md` を持つ repo | **1 本（この repo）** |
| pipeline 数 | 8（cron 2 / subscribeRepos 1 / xrpc 5） |
| sub-actor 数 | 4（`registry:refinery` / `units:conversion` / `analytics:yield` / `risk:outage`、**4/4 に対応する step がある**） |
| capability 数 | 5（うち `agent.invoke` はどの step でも未使用） |
| 宣言した購読 / handler | **5 / 1** |
| gate cell 数 | 16 = 5 xrpc + 2 requiredCollections + 4 requiredLoops + 5 subscribeRepos |
| gate 数 | 7、deny-by-default（6/7 でも `:blocked`、16 cell 全部 blocked で effect 0。7/7 で 16 ready / effect 16） |
| gate と manifest の collection の交わり | **空**（16 個すべて） |
| グラフラベル | `Refinery` 6 / `RefineryUnit` 2 / `RefineryOutage` 1 / `ActorCoverageSnapshot` 1 write。**エッジ型 0** |
| `yield` を含む JSON leaf / Cypher 内 | **11 / 0** |
| `getYieldPressure` が返すもの | `r.refinery_code` / `r.throughput_bpd` / `r.status`（歩留まりではない） |
| `ProductGrade` を言及する manifest | 39 本中 `oil-coverage` の 1 本のみ |
| フリート全体の書き手 | 39 manifest / 70 `graph.write` 中、`Refinery*` をラベルとして書くもの **0** |
| ラベルを読む側 | ラベル形 `oil-refining` 9 step / 文字列として `oil-coverage` 2 step |
| `oil-refining.etzhayyim.com` | A/AAAA 無し、curl `000` |
| `etzhayyim.com/actor/oil-refining/did.json` | `200` |
| repo did.json vs live | 5 か所相違（`diff` exit 1） |
| live の `_meta.kind` / `status` | `tier-b` / `r0` —— **後継 kamado と同一。8 本とも同じ** |
| live の `_meta.primaryLexicon` | `com.etzhayyim.oil-refining`（= gate 側の綴り。8 本とも同型） |
| `pds.etzhayyim.com/xrpc/_health` | `530` |
| `pds.aozora.app/xrpc/_health` | `200` |
| `describeRepo`（実 DID / 架空 DID） | どちらも `{"collections":[]}` で 200 |
| GitHub Pages（両 org） | `404` |
| `actor-manifest.test.ts` | 11 `it(` / 16 `expect(`、**走らない**（package.json も node_modules も無い） |
| `src/oil_refining/murakumo.cljc` | 226 行 / 8,832 B |
| 兄弟の形 | `actors=4` / `pipelines=8` / `handlers=1` は 6 本共通、購読数は 4〜6（`oil-coverage` のみ 6 / 5 / 12 / 0） |

## 結果

- `README.md` / `docs/operator-quickstart.md` / この ADR を追加した。
- quickstart の全 bash ブロックを機械抽出して verbatim 実行し、貼った出力と
  一致することを確認した（決定 4）。
- superproject の west pin を `fc616a7`（2026-07-02）→ `2563dbf` に進めた。旧 pin は
  2026-07-18 の rescue より前を指しており、**`src/oil_refining/murakumo.cljc`
  （226 行の gate）が checkout に現れていなかった**。成熟度 scan は pin ではなく
  checkout を読むので、この repo の substrate は 0 と測られていた —— pin を正した
  副産物として、その過小評価も解消される（`axis-substrate` は loop の目標軸ではない）。
  **これは 4 周連続で同じ形である**（`oil-coverage` / `oil-distribution` /
  `oil-midstream` / `oil-refining` とも pin が rescue merge より前だった）。

## やっていないこと

- **test を書いていない。** README が主張する数（pipeline 8 / sub-actor 4 / cell 16 /
  gate 7 / label 3 / 購読 5・handler 1 / yield を含む Cypher 0）と 2 つの DID を
  実体と突き合わせるものが無く、manifest が変わっても README は赤くならない。
  `marine-insurance` の `test/marine_insurance/docs_test.cljs` が先例で、同型が要る。
  1 反復 1 軸なので次周（`axis-test`）に残した。
- **`MIGRATION-NOTES.md` を直していない**（決定 3）。
- **supersede を機械が検出できる形にしていない。** `:actor/supersedes` は kamado 側に
  しか無く、この repo 側は散文だけ。live の identity 面も後継と区別が付かない
  （決定 9）。**フリート全体で「superseded な actor」を列挙する手段が無い**ことを
  意味するが、その台帳をどこに置くかはこの repo の決定ではない。
- identity の不整合（決定 8）、collection 語彙の不一致（決定 10）、購読 handler の
  欠落 4 件、歩留まりモデルの不在（決定 6）、gate が supersede を強制しないこと
  （決定 12）は、いずれも記録のみ。
- namespace の `oil_refining`（アンダースコア）は直していない —— 直すと
  `require` の綴りが変わる。
- `actor-manifest.test.ts` を nbb の `test/` に書き直していない。
- **同型の兄弟 3 本**（`oil-upstream` / `oil-trading` / `oil-shipping`）は
  README 0 バイトのまま。今回 1 本だけ。
