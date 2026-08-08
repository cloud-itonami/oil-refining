# operator quickstart

**この repo で「動かせる」ものは 1 つだけ**（手順 4 の gate）。残りは、宣言と
現実がどれだけずれているかを **自分の端末で確かめる**ための手順である。

**先に読むこと: この actor は superseded である**（手順 1）。ここで測るのは
「動かし方」ではなく「何が残っていて、後継がどこにあるか」。新しい仕事を
載せる先は `orgs/cloud-itonami/kamado`。

所要 5 分。必要なのは `git` / `jq` / `curl` / `dig` / `nbb`。
下に貼ってある出力はすべて 2026-08-09 に実際に実行した結果の写しで、手打ちではない。
**手順 1〜3 は `..` を引く** ので、`oil-*` 7 本と **`kamado`** が同じ親ディレクトリに
checkout されている前提（superproject の `orgs/cloud-itonami/`）。無ければ
`printf '%s\n' kamado | xargs west update --fetch smart` で取る。

---

## 手順 0 — checkout が gate を持っているか確かめる

`src/` が無ければ、west の pin が `2563dbf` より前を指している。

```bash
ls src/oil_refining/murakumo.cljc && git log --oneline -1
```

```
src/oil_refining/murakumo.cljc
2563dbf Merge pull request #1 from etzhayyim/rescue/murakumo-wip-20260718
```

（この写しは `2563dbf` 時点のもの。**`2563dbf` 以降ならどの commit でもよい** ——
この文書自体が入った commit を含む。1 行目の `src/…` が出ることだけが条件。）

`src/` が無い場合は superproject 側で pin を進める。**west.yml は生成物なので手で
編集せず、サーバ側 single-entry commit を使う**（詳細は skill `west-pin-advance`）:

```bash
nbb --classpath ".:scripts/nbb_compat" scripts/west-pin-put.cljs oil-refining HEAD --dry-run
nbb --classpath ".:scripts/nbb_compat" scripts/west-pin-put.cljs oil-refining HEAD
printf '%s\n' oil-refining | xargs west update --fetch smart
```

（`xargs` は必須。`printf … | west update` は**引数ゼロ = 全 4,100 project 更新**になる。）

---

## 手順 1 — 後継が実在することを確かめる（この repo で最初に確かめるべきこと）

`MIGRATION-NOTES.md` は「kamado に置き換えられた」と書いている。**その主張が
片側だけのものでないことを、後継側から確かめる。**

```bash
head -3 MIGRATION-NOTES.md
grep -n 'actor/supersedes' ../kamado/manifest.edn
```

```
# oil-refining — SUPERSEDED by kamado 竈 (ADR-2606051500)

**Status**: legacy · superseded-not-deleted · do not extend.
11: :actor/supersedes ["oil-refining"]
```

kamado 側の言及は 1 箇所ではない（**9 ファイル**）:

```bash
grep -rl 'oil-refining' ../kamado --include='*.edn' --include='*.md' \
  --include='*.cljc' --include='*.jsonld' --include='*.json' | sort
```

```
../kamado/CLAUDE.md
../kamado/README.md
../kamado/data/seed.edn
../kamado/manifest.edn
../kamado/src/kamado/methods/analyze.cljc
../kamado/src/kamado/methods/ingest.cljc
../kamado/test/kamado/methods/test_ingest.cljc
../kamado/wire/ingest/legacy-oil-refining-export.sample.json
../kamado/wire/manifest.jsonld
```

### ⚠ ただし `MIGRATION-NOTES.md` の実行手順はもう踏めない

notes は `20-actors/kamado/methods/ingest.py` を走らせろと書いている。**その
ファイルは存在せず、kamado に Python は 1 行も無い**（`bb`→`nbb`/`cljc` 移行
＝ ADR-2607173000 のあと）:

```bash
grep -n 'ingest.py\|data/ingest/' MIGRATION-NOTES.md | head -4
find ../kamado -name '*.py' -not -path '*/.git/*' | wc -l
ls ../kamado/src/kamado/methods/ ../kamado/wire/ingest/
```

```
39:The ETL is implemented (watari `ingest.py` precedent):
42:20-actors/kamado/methods/ingest.py            # legacy node export → kotoba EAVT + kg.ingest_batch
43:20-actors/kamado/data/ingest/legacy-oil-refining-export.sample.json   # Cypher-node-shaped sample
50:python3 ingest.py --export ../data/ingest/legacy-oil-refining-export.sample.json
       0
../kamado/src/kamado/methods/:
analyze.cljc
carbon_balance.cljc
decommission_robot.cljc
feedstock_guard.cljc
ingest.cljc
social.cljc
substrate.cljc

../kamado/wire/ingest/:
legacy-oil-refining-export.sample.json
```

**「後継が在る」は正しく、「そこへ移る手順」は古い。** 2 つを区別すること。

成熟度の差も測れる（superproject 側の生成物、`414 bp` vs `5518 bp`）:

```bash
grep -o '{[^{}]*"orgs/cloud-itonami/\(oil-refining\|kamado\)"[^{}]*}' \
  ../../../90-docs/system-dynamics/itonami-maturity.datoms.edn \
  | grep -o ':repo/path "[^"]*"\|:maturity/own-bp [0-9]*'
```

```
:repo/path "orgs/cloud-itonami/kamado"
:maturity/own-bp 5518
:repo/path "orgs/cloud-itonami/oil-refining"
:maturity/own-bp 414
```

---

## 手順 2 — descriptor の形を数える

README の表の数は、すべてこの 1 コマンドから出ている。

```bash
jq -r '{pipelines:(.pipelines|length), actors:(.actors|length),
        capabilities:(.capabilities|length),
        subscribeRepos:(.triggers.subscribeRepos.collections|length),
        nanoid:.nanoid}' actor-manifest.jsonld
```

```json
{
  "pipelines": 8,
  "actors": 4,
  "capabilities": 5,
  "subscribeRepos": 5,
  "nanoid": "01lr3f1n"
}
```

兄弟 6 本と比べる。**`actors` / `pipelines` は同じで、購読数と `nanoid` が違う**
（`oil-coverage` だけ形からして別物）:

```bash
(cd .. && for n in oil-upstream oil-midstream oil-refining oil-trading \
                   oil-shipping oil-distribution oil-coverage; do
  printf "%-17s " $n
  jq -r '"actors=" + ((.actors|length)|tostring) +
         " pipelines=" + ((.pipelines|length)|tostring) +
         " subs=" + ((.triggers.subscribeRepos.collections|length)|tostring) +
         " nanoid=" + .nanoid' $n/actor-manifest.jsonld
done)
```

```
oil-upstream      actors=4 pipelines=8 subs=5 nanoid=01lupstr
oil-midstream     actors=4 pipelines=8 subs=5 nanoid=01lm1dst
oil-refining      actors=4 pipelines=8 subs=5 nanoid=01lr3f1n
oil-trading       actors=4 pipelines=8 subs=4 nanoid=01ltrad3
oil-shipping      actors=4 pipelines=8 subs=6 nanoid=01l5h1p0
oil-distribution  actors=4 pipelines=8 subs=4 nanoid=01ld1str
oil-coverage      actors=6 pipelines=5 subs=12 nanoid=011c0v3r
```

**7 本のうち `MIGRATION-NOTES.md` を持つのはこの repo だけ**である:

```bash
(cd .. && for n in oil-upstream oil-midstream oil-refining oil-trading \
                   oil-shipping oil-distribution oil-coverage; do
  printf "%-17s MIGRATION-NOTES=%s\n" $n "$([ -f $n/MIGRATION-NOTES.md ] && echo yes || echo no)"
done)
```

```
oil-upstream      MIGRATION-NOTES=no
oil-midstream     MIGRATION-NOTES=no
oil-refining      MIGRATION-NOTES=yes
oil-trading       MIGRATION-NOTES=no
oil-shipping      MIGRATION-NOTES=no
oil-distribution  MIGRATION-NOTES=no
oil-coverage      MIGRATION-NOTES=no
```

---

## 手順 3 — trigger と、実際に触るグラフラベルを見る

```bash
jq -r '.pipelines[] | .trigger.type + " " +
       (.trigger.cron // .trigger.nsid // ((.trigger.collections//[])|join(","))) +
       "  steps=" + ((.steps|length)|tostring)' actor-manifest.jsonld
```

```
cron 0 */8 * * *  steps=5
subscribeRepos com.etzhayyim.apps.oilShipping.cargo  steps=1
xrpc com.etzhayyim.apps.oilRefining.registry.getRefinery  steps=1
xrpc com.etzhayyim.apps.oilRefining.registry.listRefineries  steps=1
xrpc com.etzhayyim.apps.oilRefining.registry.listUnits  steps=1
xrpc com.etzhayyim.apps.oilRefining.analytics.getYieldPressure  steps=1
xrpc com.etzhayyim.apps.oilRefining.health  steps=1
cron 0 */6 * * *  steps=3
```

読み書きするラベルは 4 つしか出てこない（**エッジ型は 1 つも無い**）:

```bash
jq -r '.pipelines[].steps[] | (.args.sql // .args.template // empty)' actor-manifest.jsonld \
  | grep -oE '\([a-z]+:[A-Za-z]+' | sort | uniq -c | sort -rn
```

```
   6 (r:Refinery
   2 (u:RefineryUnit
   1 (o:RefineryOutage
   1 (c:ActorCoverageSnapshot
```

### 「Yield Intelligence」を名乗るが、歩留まりに触る query は 1 本も無い

manifest 全体で `yield` を含む JSON leaf を列挙する。**11 個あるが、Cypher の中は
1 個も無い** —— 唯一 pipeline に現れるのは `agent.chat` に渡す散文プロンプト:

```bash
jq -r 'paths(scalars) as $p | [($p|join(".")), (getpath($p)|tostring)] | @tsv' \
  actor-manifest.jsonld | grep -i yield
```

```
actors.2.description	Crude-to-product yield pressure and output balance.
actors.2.displayName	Yield Analytics
actors.2.path	analytics:yield
convoSystemPrompt	You are the Oil Refining agent. Track refinery assets, unit configuration, throughput, outages and product yields. Explain how crude inputs, refinery complexity and unit availability affect gasoline, diesel, jet and fuel oil output.
description	Refinery registry, unit configuration, throughput and utilization monitoring, outage tracking, and crude-to-product yield intelligence.
displayName	Oil Refining & Product Yield Intelligence
pipelines.0.steps.3.args.message	Oil refining report.\n\nRefinery stats:\n$refineryStats.rows\n\nUnit stats:\n$unitStats.rows\n\nOutages:\n$outages.rows\n\nSummarize refinery concentration, unit bottlenecks and likely product yield pressure.
pipelines.5.trigger.nsid	com.etzhayyim.apps.oilRefining.analytics.getYieldPressure
profile.capabilities.2	yield-analysis
profile.displayName	Oil Refining & Product Yield Intelligence
triggers.subscribeRepos.collections.3	com.etzhayyim.apps.oilRefining.yieldSnapshot
```

`getYieldPressure` が実際に発行する query と、step 全体が触るプロパティ:

```bash
jq -r '.pipelines[] | select((.trigger.nsid // "") | test("getYieldPressure"))
       | .steps[0].args.sql' actor-manifest.jsonld
jq -r '.pipelines[].steps[] | (.args.sql // .args.template // empty)' actor-manifest.jsonld \
  | grep -oE '\b[a-z]\.[a-z_]+' | sort | uniq -c | sort -rn
```

```
MATCH (r:Refinery {country_code: $input.countryCode}) RETURN r.refinery_code AS refinery, r.throughput_bpd AS throughput, r.status AS status LIMIT 50
   4 r.status
   3 r.throughput_bpd
   2 r.vertex_id
   2 r.repo
   2 r.refinery_code
   2 r.country_code
   2 r.collection
   2 o.status
   2 n.repo
   2 n.collection
   1 u.vertex_id
   1 u.unit_type
   1 u.status
   1 u.repo
   1 u.collection
   1 o.unit_type
   1 o.refinery_code
   1 n._seq
```

返るのは **処理能力**（`throughput_bpd`）であって歩留まりではない。18 種類の
プロパティに製品・収率に相当するものは無い。

フリートに `ProductGrade` というラベル名は存在するが、**それを言う manifest は
`oil-coverage` の 1 本だけ**で、`oil-refining` は触らない:

```bash
(cd .. && grep -l 'ProductGrade' */actor-manifest.jsonld | sed 's|/actor-manifest.jsonld||'
   grep -c 'ProductGrade' oil-refining/actor-manifest.jsonld)
```

```
oil-coverage
0
```

### 購読は 5 件、handler は 1 件

```bash
jq -r '.triggers.subscribeRepos.collections[]' actor-manifest.jsonld
jq -r '.pipelines[] | select(.trigger.type=="subscribeRepos")
       | "handler: " + ((.trigger.collections//[])|join(","))' actor-manifest.jsonld
```

```
com.etzhayyim.apps.oilRefining.refinery
com.etzhayyim.apps.oilRefining.unit
com.etzhayyim.apps.oilRefining.outage
com.etzhayyim.apps.oilRefining.yieldSnapshot
com.etzhayyim.apps.oilShipping.cargo
handler: com.etzhayyim.apps.oilShipping.cargo
```

7 本すべてで同じ形である（`oil-coverage` は 12 件宣言して handler ゼロ）:

```bash
(cd .. && for n in oil-upstream oil-midstream oil-refining oil-trading \
                   oil-shipping oil-distribution oil-coverage; do
  printf "%-17s " $n
  jq -r '"declared=" + ((.triggers.subscribeRepos.collections|length)|tostring) +
         " handlers=" + (([.pipelines[]|select(.trigger.type=="subscribeRepos")]|length)|tostring)' \
    $n/actor-manifest.jsonld
done)
```

```
oil-upstream      declared=5 handlers=1
oil-midstream     declared=5 handlers=1
oil-refining      declared=5 handlers=1
oil-trading       declared=4 handlers=1
oil-shipping      declared=6 handlers=1
oil-distribution  declared=4 handlers=1
oil-coverage      declared=12 handlers=0
```

### 読むノードを書く者がフリート内に居ない（そして正規表現は両方向に間違える）

cloud-itonami の 39 本の `actor-manifest.jsonld`（`graph.write` 70 個）を全部見る。
**素朴な部分一致とラベル形の両方を出して、差を見ること**:

```bash
(cd ..
echo "--- writes: 部分一致 ---"
for f in */actor-manifest.jsonld; do
  jq -r --arg r "${f%/actor-manifest.jsonld}" '.pipelines[]?.steps[]?
    | select(.fn=="graph.write") | select((.args|tostring)|test("Refinery")) | $r' "$f" 2>/dev/null
done | sort | uniq -c
echo "--- writes: ラベル形 ---"
for f in */actor-manifest.jsonld; do
  jq -r --arg r "${f%/actor-manifest.jsonld}" '.pipelines[]?.steps[]?
    | select(.fn=="graph.write") | select((.args|tostring)|test("\\([a-z]+:Refinery")) | $r' "$f" 2>/dev/null
done | sort | uniq -c
echo "--- reads: ラベル形 ---"
for f in */actor-manifest.jsonld; do
  jq -r --arg r "${f%/actor-manifest.jsonld}" '.pipelines[]?.steps[]?
    | select(.fn=="graph.query") | select((.args|tostring)|test("\\([a-z]+:Refinery")) | $r' "$f" 2>/dev/null
done | sort | uniq -c
)
```

```
--- writes: 部分一致 ---
   1 rare-earth-coverage
--- writes: ラベル形 ---
--- reads: ラベル形 ---
   9 oil-refining
```

**部分一致の 1 件は偽陽性である。** `rare-earth-coverage` が書くのは
`(n:RareEarthActor …)` で、当たっているのは値の中の文字列:

```bash
(cd .. && jq -r '.pipelines[]?.steps[]? | select(.fn=="graph.write")
  | select((.args|tostring)|test("Refinery")) | .args.template' \
  rare-earth-coverage/actor-manifest.jsonld \
  | grep -oE "role:'[a-z-]*refinery[a-z-]*'|MERGE \(n:[A-Za-z]+" | sort -u)
```

```
MERGE (n:RareEarthActor
role:'refinery'
role:'refinery-project'
role:'refinery-recycler'
```

**ラベル形は逆に偽陰性を出す。** `oil-coverage` はラベル名を**文字列として**
数えており（`lbl IN labels(n)`）、この正規表現には映らない:

```bash
(cd .. && jq -r '.pipelines[]?.steps[]? | select(.fn=="graph.query")
  | select((.args|tostring)|test("Refinery")) | .id' oil-coverage/actor-manifest.jsonld)
```

```
backboneCounts
backbone
```

正確には: **ラベルを書く者は 0、ラベル形で読むのは `oil-refining` の 9 step、
ラベル名を文字列として数えるのは `oil-coverage` の 2 step。**

---

## 手順 4 — gate を実際に走らせる（この repo で唯一動くもの）

`/tmp/probe-refining.cljs` を作る:

```clojure
(ns probe-refining (:require [oil_refining.murakumo :as m]))

(def all-gates (set m/common-gates))

(defn summarise [label atts]
  (let [p (m/cell-plan :getrefinery
                       {:attestations atts :request-id "probe-1"
                        :computed-at "2026-08-09T00:00:00Z"})]
    (println (str label "  status=" (:status p)
                  "  effects=" (count (:effects p))
                  "  missing=" (count (:missing-gates p))))))

(println "cells:" (count m/cell-specs) " gates:" (count m/common-gates))
(summarise "none      " #{})
(summarise "6-of-7    " (disj all-gates :kotoba-only-substrate-baseline))
(summarise "all 7     " all-gates)
(println "gate collection:" (m/collection "refinery"))
(println "actor-did:" m/actor-did)
(let [plans (m/all-cell-plans {:attestations #{}})]
  (println "blocked:" (count (filter #(= :blocked (:status %)) (vals plans)))
           "/" (count plans)
           " total effects:" (reduce + 0 (map #(count (:effects %)) (vals plans)))))
(let [plans (m/all-cell-plans {:attestations all-gates})]
  (println "all-7 ready:" (count (filter #(= :ready (:status %)) (vals plans)))
           "/" (count plans)
           " total effects:" (reduce + 0 (map #(count (:effects %)) (vals plans)))))
```

```bash
nbb --classpath "src:/tmp" /tmp/probe-refining.cljs
```

```
cells: 16  gates: 7
none        status=:blocked  effects=0  missing=7
6-of-7      status=:blocked  effects=0  missing=1
all 7       status=:ready  effects=1  missing=0
gate collection: com.etzhayyim.oil-refining.refinery
actor-did: did:web:oil-refining.etzhayyim.com
blocked: 16 / 16  total effects: 0
all-7 ready: 16 / 16  total effects: 16
```

**確かめるべきは「6 つ揃えても通らない」ことである** —— 部分的な attestation で
effect が 1 つでも出るなら、それは deny-by-default ではない。

**そして「7 つ揃えば通る」ことも確かめるべきである** —— この gate は
`MIGRATION-NOTES.md` の `do not extend` を知らない。superseded 宣言を強制する
仕組みはコードのどこにも無い。

`gate collection:` の行と、manifest 側の綴りを見比べる。**交わりが無い**:

```bash
jq -r '.triggers.subscribeRepos.collections[]' actor-manifest.jsonld
```

```
com.etzhayyim.apps.oilRefining.refinery
com.etzhayyim.apps.oilRefining.unit
com.etzhayyim.apps.oilRefining.outage
com.etzhayyim.apps.oilRefining.yieldSnapshot
com.etzhayyim.apps.oilShipping.cargo
```

16 cell の全 collection を出すと、**XRPC のメソッド名も、他 actor の collection も
自分の名前空間に付け替わっている**ことが見える（`getrefinery` / `cargo`）:

```bash
cat > /tmp/probe-cols.cljs <<'EOF'
(ns probe-cols (:require [oil_refining.murakumo :as m]))
(doseq [[k v] (sort-by key m/cell-specs)]
  (println (str (name k) "\t" (first (:collections v)))))
EOF
nbb --classpath "src:/tmp" /tmp/probe-cols.cljs
```

```
cargo	com.etzhayyim.oil-refining.cargo
domain-knowledge	com.etzhayyim.oil-refining.domain-knowledge
getrefinery	com.etzhayyim.oil-refining.getrefinery
getyieldpressure	com.etzhayyim.oil-refining.getyieldpressure
health	com.etzhayyim.oil-refining.health
koji	com.etzhayyim.oil-refining.koji
kyumei	com.etzhayyim.oil-refining.kyumei
listrefineries	com.etzhayyim.oil-refining.listrefineries
listunits	com.etzhayyim.oil-refining.listunits
outage	com.etzhayyim.oil-refining.outage
refinery	com.etzhayyim.oil-refining.refinery
shinka	com.etzhayyim.oil-refining.shinka
shinkaevolution	com.etzhayyim.oil-refining.shinkaevolution
shinkaknowledge	com.etzhayyim.oil-refining.shinkaknowledge
unit	com.etzhayyim.oil-refining.unit
yieldsnapshot	com.etzhayyim.oil-refining.yieldsnapshot
```

cell が 16 ある理由も manifest から導ける:

```bash
jq -r '((.pipelines|map(select(.trigger.type=="xrpc"))|length) + (.requiredCollections|length)
        + (.requiredLoops|length) + (.triggers.subscribeRepos.collections|length))' actor-manifest.jsonld
```

```
16
```

---

## 手順 5 — 2 つの DID のうち、どちらが解決するか確かめる

```bash
dig +short oil-refining.etzhayyim.com A          # ← 何も返らない
curl -s -o /dev/null -w '%{http_code}\n' https://oil-refining.etzhayyim.com/.well-known/did.json
curl -s -o /dev/null -w '%{http_code}\n' https://etzhayyim.com/actor/oil-refining/did.json
```

```
000
200
```

`000` は「HTTP status が無い」＝ 接続の前段で失敗した、という意味である。
**manifest と gate が名乗る方の DID が、この解決しない側。**

repo の did.json が live の写しでないことも見ておく:

```bash
curl -s https://etzhayyim.com/actor/oil-refining/did.json > /tmp/live-did-refining.json
diff <(jq -S . .well-known/did.json) <(jq -S . /tmp/live-did-refining.json) | head -30
```

`ed25519-2020` vs `jws-2020`、`alsoKnownAs` 4 件 vs 空、`verificationMethod` の
有無、PDS の宛先、2 つ目の service —— 5 か所ずれる（`diff` は exit 1）。

### live の identity 面は superseded を示さない

**後継 kamado と並べても区別が付かない**:

```bash
for n in oil-upstream oil-midstream oil-refining oil-trading \
         oil-shipping oil-distribution oil-coverage kamado; do
  printf "%-18s " $n
  curl -s --max-time 12 https://etzhayyim.com/actor/$n/did.json \
    | jq -rc '[._meta.kind, ._meta.status, ._meta.primaryLexicon] | @tsv'
done
```

```
oil-upstream       tier-b	r0	com.etzhayyim.oil-upstream
oil-midstream      tier-b	r0	com.etzhayyim.oil-midstream
oil-refining       tier-b	r0	com.etzhayyim.oil-refining
oil-trading        tier-b	r0	com.etzhayyim.oil-trading
oil-shipping       tier-b	r0	com.etzhayyim.oil-shipping
oil-distribution   tier-b	r0	com.etzhayyim.oil-distribution
oil-coverage       tier-b	r0	com.etzhayyim.oil-coverage
kamado             tier-b	r0	com.etzhayyim.kamado
```

2 つのことが同時に読める:

1. **`MIGRATION-NOTES.md` の「Not in the Tier-B roster」を、live の DID document は
   支持していない。** ただし `_meta.kind` は apex resolver が付ける値で、notes の
   言う「roster」とは出所が違う可能性がある。**どちらが正かは決めない。**
2. **`primaryLexicon` は 8 本すべて `com.etzhayyim.<repo 名>`** で、これは
   manifest の `apps.` 語彙ではなく **gate 側の綴り**である。

宛先の生死:

```bash
for u in https://pds.etzhayyim.com/xrpc/_health https://pds.aozora.app/xrpc/_health; do
  printf "%-46s -> " $u; curl -s -o /dev/null -w '%{http_code}\n' --max-time 12 $u
done
```

```
https://pds.etzhayyim.com/xrpc/_health         -> 530
https://pds.aozora.app/xrpc/_health            -> 200
```

repo の did.json が指す方（`pds.etzhayyim.com`）が落ちていて、live が指す方
（`pds.aozora.app`）が生きている。GitHub Pages 側は両 org とも 404:

```bash
for u in https://etzhayyim.github.io/com-etzhayyim-oil-refining/.well-known/did.json \
         https://cloud-itonami.github.io/oil-refining/.well-known/did.json; do
  printf "%s -> " $u; curl -s -o /dev/null -w '%{http_code}\n' --max-time 12 $u
done
```

```
https://etzhayyim.github.io/com-etzhayyim-oil-refining/.well-known/did.json -> 404
https://cloud-itonami.github.io/oil-refining/.well-known/did.json -> 404
```

---

## 手順 6 — live の PDS に record があるか（そして、なぜこれが弱い証拠か）

```bash
DID="did:web:etzhayyim.com:actor:oil-refining"
curl -s --max-time 15 "https://pds.aozora.app/xrpc/com.atproto.repo.describeRepo?repo=$DID"
```

```json
{"did":"did:web:etzhayyim.com:actor:oil-refining","handle":"handle.invalid","collections":[],"handleIsCorrect":false}
```

**この 200 を「登録されている」と読んではいけない。** 存在しない DID でも同じ形が
返る:

```bash
curl -s --max-time 15 "https://pds.aozora.app/xrpc/com.atproto.repo.describeRepo?repo=did:web:etzhayyim.com:actor:not-a-real-actor-xyz"
```

```json
{"did":"did:web:etzhayyim.com:actor:not-a-real-actor-xyz","handle":"handle.invalid","collections":[],"handleIsCorrect":false}
```

`listRecords` も同じで、架空 DID・gate 語彙・manifest 語彙のいずれでも
`{"records":[]}` が返る:

```bash
for c in com.etzhayyim.apps.oilRefining.refinery com.etzhayyim.oil-refining.refinery; do
  printf "%-42s " $c
  curl -s --max-time 12 "https://pds.aozora.app/xrpc/com.atproto.repo.listRecords?repo=$DID&collection=$c"
  echo
done
```

```
com.etzhayyim.apps.oilRefining.refinery    {"records":[]}
com.etzhayyim.oil-refining.refinery        {"records":[]}
```

言えるのは **「record は 1 件も観測できない」**までで、「登録済みだが空」と
「未登録」はこの endpoint では区別できない。

---

## 手順 7 — `.ts` テストが走らないことを確かめる

```bash
ls package.json node_modules
```

```
ls: node_modules: No such file or directory
ls: package.json: No such file or directory
```

`actor-manifest.test.ts` は vitest を import しているが、その vitest がここには
無い。**この 11 の `it(` は一度も実行されていない。**

```bash
grep -c 'it(' actor-manifest.test.ts     # 11
```

---

## ここから先に進みたい場合

**まず `orgs/cloud-itonami/kamado` を見る。** この repo を「動かす」方向には
進まないこと（`do not extend`）。

それでもこの descriptor を動かす必要があるなら、少なくとも次の 5 つが repo の外に要る:

1. `Refinery` / `RefineryUnit` / `RefineryOutage` を持つグラフ
   （**そのノードを書く者はフリート 39 本のどこにも居ない**、手順 3）
2. cron を撃つ scheduler と XRPC を受ける server（`runtime: k8s-langserver`）
3. 7 つの attestation を発行する主体（無ければ gate は永久に `:blocked`）
4. `com.etzhayyim.apps.oilRefining.*` と `com.etzhayyim.oil-refining.*` の
   どちらを lexicon の正とするかの決定（live の DID document は後者を支持している、手順 5）
5. **歩留まりのデータモデルそのもの**（`yield` は名前と散文にしか無く、Cypher に
   1 個も無い、手順 3）—— 後継の kamado はここを「歩留まり」ではなく
   「炭素収支」に置き換えている

いずれもこの repo は持っていないし、持っていると主張してもいない。
