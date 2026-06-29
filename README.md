# cron-clj (定期スケジュール式)

[![CI](https://github.com/kotoba-lang/cron/actions/workflows/ci.yml/badge.svg)](https://github.com/kotoba-lang/cron/actions/workflows/ci.yml)

Handle **cron expressions as EDN/Clojure data** in portable Clojure — every namespace
is `.cljc`, with **zero third-party runtime deps**, so it runs on the JVM,
ClojureScript, and Clojure-on-WASM hosts (SCI). A cron expression is a plain map you
can inspect, store, diff, or feed to a loop/scheduler; the library adds the field
parser, structural validator, and a pure next-fire-time evaluator around it. Pairs
naturally with scheduling and loop actors that need to know "when does this fire next?"

Sibling of the other reusable `*-clj` kernels in this org
([bpmn-clj](https://github.com/com-junkawasaki/bpmn-clj),
[dmn-clj](https://github.com/com-junkawasaki/dmn-clj)).

## Why a shared library (org placement)

Per the three-org rule, the **reusable** scheduling primitive lives in
**com-junkawasaki**; **public-benefit actor instances** that schedule concrete jobs live
in **etzhayyim**; any **business/private deployment** lives in **gftdcojp**. cron-clj
is the dep — it carries no scheduler runtime and no I/O (those are host-injected).

## The model: cron as EDN (`cron.model`)

A 5-field cron expression (or `@` macro shorthand) is parsed into a plain map of sets,
one per field, with `:cron/dom-restricted?` / `:cron/dow-restricted?` flags that drive
the Vixie-style day OR semantics:

```clojure
(require '[cron.model :as m])

(m/parse "*/15 * * * *")
;=> {:cron/minute #{0 15 30 45}
;    :cron/hour   #{0 1 … 23}
;    :cron/dom    #{1 … 31}   :cron/dom-restricted? false
;    :cron/month  #{1 … 12}
;    :cron/dow    #{0 … 6}    :cron/dow-restricted? false}

(m/parse "@daily")
;=> {:cron/minute #{0} :cron/hour #{0} … :cron/dom-restricted? false …}

(m/parse "0 0 13 * FRI")
;=> {:cron/dom #{13} :cron/dom-restricted? true
;    :cron/dow #{5}  :cron/dow-restricted? true …}
```

Supported field syntax: `*`, `n`, `a-b`, `a-b/s`, `*/s`, `a,b,c` (mixable).
3-letter names: `JAN`–`DEC` (month), `SUN`–`SAT` (dow) — case-insensitive.
Macro shorthands: `@yearly` / `@annually` / `@monthly` / `@weekly` / `@daily` / `@hourly`.

## Validation (`cron.validate`)

`problems` returns a vector of `{:cron/severity :cron/code :cron/id :cron/msg}`;
`valid?` is true iff there are no `:error`s (warnings are advisory):

```clojure
(require '[cron.validate :as v])

(v/valid? (m/parse "*/15 * * * *"))   ;=> true
(v/problems (assoc (m/parse "0 0 * * *") :cron/minute #{60}))
;=> [{:cron/severity :error :cron/code :field/out-of-range
;     :cron/id :cron/minute :cron/msg "minute values must be in 0..59; …"}]
```

Errors: field values out of the legal range (minute 0–59, hour 0–23, dom 1–31,
month 1–12, dow 0–6); empty field set.

## Execution (`cron.execute`)

A datetime is `{:y :m :d :hh :mm}` — a plain map, no java.time / Date.
Calendar arithmetic is done via a pure proleptic-Gregorian day-number helper
(Howard Hinnant's `days_from_civil` / `civil_from_days`).

```clojure
(require '[cron.execute :as e])

;; day-of-week helper: 0=Sun … 6=Sat
(e/weekday 2026 6 27)  ;=> 6  (Saturday)
(e/weekday 1970 1 1)   ;=> 4  (Thursday)

;; does this cron fire exactly at this datetime?
(e/fires-at? (m/parse "*/15 * * * *") {:y 2026 :m 6 :d 27 :hh 9 :mm 15})  ;=> true
(e/fires-at? (m/parse "*/15 * * * *") {:y 2026 :m 6 :d 27 :hh 9 :mm 7})   ;=> false

;; next firing time, strictly after `from`
(e/next-fire (m/parse "*/15 * * * *") {:y 2026 :m 6 :d 27 :hh 9 :mm 7})
;=> {:y 2026 :m 6 :d 27 :hh 9 :mm 15}

(e/next-fire (m/parse "@daily") {:y 2026 :m 6 :d 27 :hh 0 :mm 30})
;=> {:y 2026 :m 6 :d 28 :hh 0 :mm 0}
```

### Day OR semantics

When both `dom` and `dow` fields are non-`*` (i.e. both restricted), Vixie cron fires
when **either** condition is satisfied:

```clojure
;; "0 0 13 * FRI" fires on the 13th of any month OR any Friday
(e/fires-at? (m/parse "0 0 13 * FRI") {:y 2026 :m 6 :d 13 :hh 0 :mm 0})  ;=> true  (13th)
(e/fires-at? (m/parse "0 0 13 * FRI") {:y 2026 :m 6 :d 19 :hh 0 :mm 0})  ;=> true  (Friday)
(e/fires-at? (m/parse "0 0 13 * FRI") {:y 2026 :m 6 :d 14 :hh 0 :mm 0})  ;=> false
```

`next-fire` is bounded to ~3 years (~1,576,980 minute steps) and returns `nil` if no
firing is found within that window (e.g. an impossible expression like `"0 0 31 2 *"`).

## Test

```
clojure -X:test
```
