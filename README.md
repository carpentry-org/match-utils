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
; Dispatch is optimized by column scoring, list patterns before literals.
(deftype Num
  (I [Int])
  (D [Double]))

(use Num)
(MatchUtils.defn-match add-num
  [(I x) (I y)] (I (+ x y))
  [(D x) (D y)] (D (+ x y))
  [(D x) (I y)] (D (+ x (from-int y)))
  [(I x) (D y)] (D (+ (from-int x) y)))

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

`defn-match` compiles each row independently and does not implement default
rows. A row with a variable binding at some column will shadow later rows
that have concrete patterns at the same column, since the variable matches
anything at that position. Write concrete rows first and catch-alls last.

In multi-arity mode, the public name (e.g. `bump`) becomes a dispatching
macro rather than a function. Macros cannot be passed as higher-order
values, so if you need HoF use, reference the per-arity name directly
(`bump-1`, `bump-2`, etc.). Each per-arity defn is a plain first-class
function with its own concrete type.

<hr/>

Have fun!
