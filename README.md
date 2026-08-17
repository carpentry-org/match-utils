# match-utils

is a collection of utility macros around `match`.

## Usage

```clojure
; let-match takes bindings, a then-branch, and an else-branch.
; Every match branch has the same type, so the whole form is well-typed
; regardless of the body type.
(MatchUtils.let-match [(Maybe.Just x) (Array.nth &[@"hi"] 0)]
  (println* x)  ; => "hi" if the nth succeeded
  (println* "nothing"))

; defn-match allows you to write pattern-matching functions.
(deftype Num
  (I [Int])
  (D [Double]))

(use Num)
(MatchUtils.defn-match add-num
  [(I x) (I y)] (I (+ x y))
  [(D x) (D y)] (D (+ x y))
  [(D x) (I y)] (D (+ x (from-int y)))
  [(I x) (D y)] (D (+ (from-int x) y)))

; A row of variables is a catch-all for whatever the rows above it miss.
(MatchUtils.defn-match pair-or
  [(Maybe.Just a) (Maybe.Just b)] (+ a b)
  [x              y]              0)

(pair-or (Maybe.Just 1) (Maybe.Just 2))   ; => 3
(pair-or (Maybe.Just 1) (Maybe.Nothing))  ; => 0

; defn-match also supports multi-arity. When rows differ in length,
; each arity becomes its own first-class defn named `name-N`, and
; `name` itself is a dispatcher macro for call sites.
(MatchUtils.defn-match bump
  []       0
  [x]      (+ x 1)
  [x y]    (+ x y)
  [x y z]  (+ (+ x y) z))

; call sites:
(bump)          ; => 0
(bump 10)       ; => 11
(bump 3 4)      ; => 7
(bump 1 2 3)    ; => 6

; per-arity first-class use:
(Array.endo-map &bump-1 [10 20 30])  ; => [11 21 31]
```

## Notes

Prefer the qualified form `MatchUtils.let-match` and `MatchUtils.defn-match`
over `(use MatchUtils)`. A Carp quirk makes `use`-imported macros that expand
to `match` forms fail pattern-variable resolution when the call is the direct
body of a `defn` (the pattern's binder gets looked up as a value). Qualifying
the call sidesteps it.

`defn-match` tries rows in source order, and the first row that matches every
column wins. A variable pattern is a catch-all for its column, and it does not
hide later rows that the catch-all row itself fails to match, so a trailing
`[x y] fallback` row picks up everything the rows above it miss.

A row whose patterns are *all* variables does make every later row
unreachable, since it matches any input. Unreachable rows are dropped rather
than compiled, so they no longer constrain the argument types; if that leaves
an argument unconstrained, annotate it at the call site.

A bare symbol in pattern position is a pattern variable, with one exception:
a qualified symbol such as `Maybe.Nothing` is a nullary constructor, because
Carp cannot bind a dotted symbol anyway. An unqualified nullary constructor
is indistinguishable from a variable at macro-expansion time, so under
`(use Maybe)` write it as `(Nothing)` — a bare `Nothing` is a catch-all
variable and will shadow every row below it.

Row sets are not checked for exhaustiveness. An input that no row covers
prints `Unhandled case in name` and aborts.

In multi-arity mode, the public name (e.g. `bump`) becomes a dispatching
macro rather than a function. Macros cannot be passed as higher-order
values, so if you need HoF use, reference the per-arity name directly
(`bump-1`, `bump-2`, etc.). Each per-arity defn is a plain first-class
function with its own concrete type.

<hr/>

Have fun!
