



I have a free week in 2018 May that I hope to spend revisiting the architecture of the [\`coxswain\` plugin](plugins/type-checker/row-types/coxswain). This page contains my notes.


## Using type families as syntax



One core idea is to use closed empty type families as syntax. The plugin instead of instances provides their semantics.


```wiki
infix 3 .=

data Col (kl :: Type) (kt :: Type)

type family (l :: kl) .= (t :: kt) = (col :: Col kl kt) | col -> l t where {}

data Row (kl :: Type) (kt :: Type)

type family Row0 :: Row kl kt where {}

infixl 2 .&
type family (p :: Row kl kt) .& (col :: Col kl kt) = (q :: Row kl kt) where {}
```


I don't really see an alternative to this.


### Blocking SEQSAME et al



Unfortunately, type families as syntax interferes with the GHC constraint simplifier. For example, the ambiguity check for the `DLacks` dctor fails.


```wiki
data B

data DLacks p l where
  DLacks :: (p ~ (q .& B .= ()),Lacks p l) => DLacks p l
```


The constraints look like


```wiki
given
  [G] $d~_a4oC {0}:: p[sk:2] ~ p[sk:2] (CDictCan)
  [G] $d~~_a4oD {0}:: p[sk:2] ~~ p[sk:2] (CDictCan)
  [G] $dLacks_a4on {0}:: Lacks p[sk:2] l[sk:2] (CDictCan)
  [G] co_a4oy {0}:: (B .= ()) ~# fsk_COL[fsk:0] (CFunEqCan)
  [G] co_a4oA {0}:: (q[sk:2] .& fsk_COL[fsk:0]) ~# fsk_ROW[fsk:0] (CFunEqCan)
  [G] co_a4oB {1}:: fsk_ROW[fsk:0] ~# p[sk:2] (CTyEqCan)
derived
wanted
  [WD] hole{co_a4oM} {3}:: (alpha[tau:2] .& fsk_COL[fsk:0]) ~# p[sk:2] (CNonCanonical)
untouchables [fsk_ROW[fsk:0], fsk_COL[fsk:0], l[sk:2], q[sk:2], p[sk:2]]
touchables (alpha[tau:2], alpha[tau:2])
```


(I opened [\#15147](http://gitlabghc.nibbler/ghc/ghc/issues/15147) because the `fsk` is supposedly unexpected in a
Wanted.)



Via `-ddump-tc-trace`, we see that the Wanted was a `CTyEqCan` before it
was unflattened and zonked prior to being handed to the plugin.


```wiki
  {[WD] hole{co_a4oM} {3}:: (s_a4oE[fmv:0] :: Row * *)
                            GHC.Prim.~# (p[sk:2] :: Row * *) (CTyEqCan)}
```


The simplification rule `SEQSAME` from Fig.23 of jfp-outsidein would
fire for `co_a4oB` and the (canonical) `hole{co_a4oM}` if both equalities
were swapped such that `p` were on the left. Note that `Note [Canonical orientation for tyvar/tyvar equality constraints]` says "If
either is a flatten-meta-variables, it goes on the left" and also "If
one is a flatten-skolem, put it on the left so that it is substituted
out"; so that explains the orientation we see.



Because we're using "type families as syntax", this impedance of `SEQSAME` will be common.



And it affects other rules too.


```wiki
f :: (p ~ (u .& A .= ()),p ~ (v .& B .= ())) => Proxy p -> Proxy u -> Proxy v
f _ _ = Proxy
```


When simplifying the givens for `f`, the orientation similarly blocks the `EQSAME` interaction rule from Fig.22 for `co_a4qk` and `co_a4qt`.


```wiki
  [G] co_a4qq {0}:: (B .= ()) ~# (fsk_a4qp[fsk:0]) (CFunEqCan)
  [G] co_a4qh {0}:: (A .= ()) ~# (fsk_a4qg[fsk:0]) (CFunEqCan)
  [G] co_a4qs {0}:: (v_a4p4[sk:2] .& fsk_a4qp[fsk:0]) ~# (fsk_a4qr[fsk:0]) (CFunEqCan)
  [G] co_a4qj {0}:: (u_a4p3[sk:2] .& fsk_a4qg[fsk:0]) ~# (fsk_a4qi[fsk:0]) (CFunEqCan)
  [G] co_a4qk {1}:: (fsk_a4qi[fsk:0]) ~# (p_a4p2[sk:2]) (CTyEqCan)
  [G] co_a4qt {1}:: (fsk_a4qr[fsk:0]) ~# (p_a4p2[sk:2]) (CTyEqCan)
```

### Equal Untouchable Variables



The plugin is only responsible for *non-trivial* row/set equalities. The example here demonstrates that a var-var equality is still trivial (i.e. GHC will completely handle it) even if both variables are untouchable.



If the givens contain an equality for two untouchable variables, the other givens will only contain one of those variables; GHC (8.4.1) will have replaced the rest. (GHC does this "the hard way" via the `EQSAME` interaction rule instead of via unification because two variables, as untouchables, cannot be unified.) For example,


```wiki
f :: C (a,b) => a :~: b -> ()
f Refl = ()
```


for some class `C` with no instances gives


```wiki
========== 3 ==========
given
  [G] $dC_a1uk {0}:: C (a_a1ub[sk:2], a_a1ub[sk:2]) (CDictCan)
  [G] co_a1uf {0}:: (b_a1uc[sk:2] :: *)
                    ~# (a_a1ub[sk:2] :: *) (CTyEqCan)
given_uf_tyeqs
  ([G] co_a1uf {0}:: (b_a1uc[sk:2] :: *)
                     ~# (a_a1ub[sk:2] :: *) (CTyEqCan),
   b_a1uc[sk:2],
   a_a1ub[sk:2])
derived
wanted
```


Note that `_a1uk` is now `C (a,a)` instead of the `C (a,b)` from the original source.



GHC will also have applied the renaming to any wanteds via `SEQSAME`.


## Generating "Hidden" Givens While Solving Wanteds



If I recall correctly, new givens cannot be generated while simplifying wanteds (the comment on [
\`TcInteract.runTcPluginsWanted\`](https://github.com/ghc/ghc/blob/57858fc8b519078ae89a4859ce7588adb39f6e96/compiler/typecheck/TcInteract.hs#L269) sounds like I'm recalling correctly). I think this should be allowed, if only as a convenience to plugin authors. They could at least use it to memoize any forwarding chaining/reasoning done by their plugin as it solves wanteds. (For my particular plugin, it might also be nice if these hidden givens could be accompanied by similarly hidden universally quantified variables; in other words, binding this evidence might reveal an existentially quantified variable. This seems like a bigger ask...)



It might also enable the plugin to cache state specific to the givens, based on an otherwise unused constraint like `MyStateCacheIdentifier 42`. (This is probably only wise for memoization.) And if that cached state contains type variable uniques, then another otherwise unused constraint `Rename '[v1,v2,...]`, would let the plugin track how the uniques on the type variables `v1,v2,...` during GHC's turn. Uniques on evidence variables could not be similarly tracked, which is problematic. That deficiency of this workaround suggests that reporting updated uniques to a plugin deserves genuine support in the API.


## Set Types



I'm pursuing "set types" instead of "row types". There's ways to embed `Set k` into `Row k Void` or some such, but it seems more delicate (eta-contraction issues with lifted types?) than I'd like to try at first. And one of my prime interests is a view of algebraic data types as named summands, which I think will be less redundant via type-indexed-sums than via row-based variants.



This is my current sketch for the "syntax" types.


```wiki
data Set (n :: Nat) (k :: *)
type family Set0 :: Set 0 k
type family (s :: Set n k) .& (e :: k) :: Set (1+n) k where {}
```


I need a better word than "set" --- it's way to overloaded. But it's so temptingly short...



The kind for future row types might have a set index, as in `Row (Set n kl) kt`.


## Simplifying `Lacks` constraints



**This section can be read as background for The Monotonic Knowledge Store section below, but that section supercedes this one.**



There are two fundamental simplifications to `Lacks` constraints, based on the operationally-useful semantics of a `Lacks x b` dictionary being the number of elements in `x` that are "less than" `b`.


```wiki
[Lacks0] (d : Lacks Set0 b) = 0
[Lacks1] (d : Lacks (Set0 .& a) b) = 1    if a < b
[LacksE] (d : Lacks (x .& a) b) = (d1 : Lacks x b) + (d2 : Lacks (Set0 .& a) b)
```


And here is a slightly less obvious rule, which emphasizes that singleton `Lacks` constraints are declarations of inequality where the type promises that `a` and `b` are unequal while only the evidence indicates the concrete relative ordering.


```wiki
[Lacks-] (d : Lacks (Set0 .& a) b) = 1 - (d1 : Lacks (Set0 .& b) a)
```


(\[Lacks-\] + \[Lacks1\] handles the `a > b` case that you may have been expecting.)



By the orderless semantics of set forms, the `[LacksE]` rule generalizes to decompositions like the following.


```wiki
(d : Lacks (x .& u .& v .& w) b) = (d1 : Lacks (x .& u .& w) b) + (d2 : Lacks (Set0 .& v) b)
(d : Lacks (x .& u .& v .& w) b) = (d1 : Lacks (x .& v .& w) b) + (d2 : Lacks (Set0 .& u) b)
(d : Lacks (x .& u .& v .& w) b) = (d1 : Lacks (x .& v) b)      + (d2 : Lacks (Set0 .& u .& w) b)
(d : Lacks (x .& u .& v .& w) b) = (d1 : Lacks (x .& v) b)      + (d2 : Lacks (Set0 .& u) b       + (d3 : Lacks (Set0 .& w) b)
(d : Lacks (x .& u .& v .& w) b) = (d1 : Lacks x b) + (d2 : Lacks (Set0 .& u .& v .& w) b)
...
```


Note the multi-modality of the numbers in (both sides of) those equations: knowing all but one number determines the other, using addition and/or subtraction.



Also note that, in an otherwise empty context, `(Lacks (Set0 .& u) b,Lacks (Set0 .& v) b)` is a stronger constraint than `Lacks (Set0 .& u .& v) b`: the evidence of the second says 0-2 of `u` and `v` are less than `b`, while the evidence of the first tells us the exact relationship of `u` to `b` and of `v` to `b`.



Because we can't solve eg `Lacks (Set0 .& u) b` from `Lacks (Set0 .& u .& v) b`, we must not decompose wanteds prematurely; the decomposition must be driven by what evidence is already available and/or already needed.


## Monotonic Knowledge Store



A quick query or two didn't turn up the title, but I'm pretty sure Gundry and McBride published a paper about incrementally and monotonically refining a knowledge store. That's my new approach to the plugin. (It's also a way to view the core of the outsidein-jfp paper, especially Section 7.) Type Families As Syntax pretty much prevents GHC (or even other plugins, almost surely) from having anything to contribute about set forms -- irreducible type family applications are essentially opaque. Except to our plugin, of course.



Thus the nub of it is to incrementally build a knowledge store from the givens and then query it to simplify/solve the wanteds. The knowledge store's representation ought to be canonical (e.g. no redundant information, etc). My current plan is to do build the knowledge store in two alternating phases until fixed point. One phase processes set equality constraints and the other processes `Lacks` constraints.


### Equality Phase



In this phase, we process given equalities at kind `Set n k` to build an equality knowledge store, which is ultimately a substitution from skolem variables (eg type variables from the source code type signature) to set forms.



One major wrinkle here is implicit variables: the constraint `x .& B ~ y .& A` implies the existence of some implicit variable `alpha` such that `x ~ alpha .& A` and `y ~ alpha .& B`. The difficulty is that -- as far as I can tell -- plugins do not have any means of introducing such skolem variables to the givens. (GHC eventually panicked if I did that and left them there, IIRC.) After thinking about it, though, I'm pretty sure all occurrences of these variables will be such that the plugin, once it's done every simplification/solving it can, will be able to replace the set forms involving those implicit variables with a set form that doesn't. E.G. every occurrence of `alpha` will be in a set form that includes `A` or `B` so that it can be rewritten in terms of `x` or `y` or will be in a `Lacks` constraint where either `A` or `B` can be added to the extension. Given `Lacks x B`, for example, we would at some point simplify to `Lacks alpha B` since we know `A < B`. However, we'll either be able to discharge that constraint or reintroduce the `A` in order to eliminate the occurrence of the `alpha`.



The basic task is to add a given equality `lr .& le0 .& le1 .& ... .& leN ~ rr .& re0 .& re1 .& ... .& reN` to the equality store. We do this in a few steps.


1. We apply the store's current substitution to `lr` and `rr`. (I do not plan for V1 to support sets inside of sets, so I'm not substituting into the elements.) I presume the substitution is idempotent: it yields a new root that is not in the domain of the substitution -- this is an invariant I work to maintain in the store. So we could have a new root and some additional elements, but the form of our equality is the same: a root and maybe some elements on both sides. Note that the new root may be an implicit variable.
1. We then drop elements that occur on both sides. E.G. if `le3` and `re2` are equivalent (according to `eqType` e.g.), then we remove them from both sides. (I do not yet have a general plan for recognizing the user-error where a type occurs multiple times on a single side. However, as currently worded, this step might make that user-error harder to spot.)
1. If some of the elements could eventually unify to be equivalent to one another (e.g. `[a]` and `[b]` might eventually be equivalent if `a` and `b` were to be unified) then we cannot (yet) process it. It's not a contradiction, but we can't add it to the knowledge store until we're sure the two elements will never unify.
  Note that querying the `Lacks` knowledge store might evidence that `a` and `b` cannot be equivalent.
1. If only one of the sides is now just a root variable with no extension, then we can use that to extend the substitution (if the root is a explicit variable) or update the substitution (if the root is an implicit variable).
1. If both roots are the empty set, then it's either a contradiction (e.g. `Set0 .& A ~ Set0`), a derived equivalence (`Set0 .& a ~ Set0 .& b` becomes `a ~ b`), or an equality we cannot process (`Set0 .& x .& y ~ Set0 .& a .& b`).
1. If both roots are variables, then we "articulate" `x` and `y`: we create a new implicit variable and extend the substitution with corresponding mappings for the two root variables (`x .& B ~ y .& A` becomes `x := alpha .& A` and `y := alpha .& B` for a fresh `alpha`).

### `Lacks` Phase



In addition to the equality store, we also build a canonical representation of the `Lacks` constraints. In order to explain the invariants, we'll need to understand arithmetic on `Lacks` dictionaries. A `Lacks x b` dictionary evidences that a type `b` is not an element of `x`, but the evidence for that contains more information than is strictly necessary. In particular it is a non-negative integer that supports an efficient implementation of records/variants etc. The semantics of `d :: Lacks x b` is that exactly `d`-many elements in `x` are "less than" `b`. This details of this ordering are arbitrary, but it is total for types with no free variables. (I implement it using a partial deterministic variation on [
Type.nonDetCmpTypeX](https://github.com/ghc/ghc/blob/21e1a00c0ccf3072ccc04cd1acfc541c141189d2/compiler/types/Type.hs#L2177).)



Instead of a substitution like the equality store, `Lacks` constraints are represented as a set of nested maps. A constraint `d :: Lacks (x .& u .& v) b`, becomes a sequence of mappings `b := m0` such that `m0` includes `x := m1` such that `m1` includes `{u,v} := d`. The key invariant for this store is that the recorded mappings cannot be simplified (compare to the idempotency of the equality store). In particular, if `b , x , {u,v} , d1` is in the store, then `b , x , {u,v,w} , d2` cannot be in the store, but `b , Set0 , {w} , (d2 - d1)` should be instead. These are the possible simplifications that the store must be canonical wrt:


```wiki
[Lacks0] (d : Lacks Set0 b) = 0
[Lacks<] (d : Lacks (Set0 .& e) b) = 1    if e < b
[LacksI] (d : Lacks (Set0 .& e) b) = 1 - (d1 : Lacks (Set0 .& b) e)
[LacksE] (d : Lacks (x .& e) b) = (d1 : Lacks x b) + (d2 : Lacks (Set0 .& e) b)
```


Note that the orderless semantics of set forms means that `[LacksE]` generalizes to a sum of any set partition of the extension elements with the additional restriction that either the root `x` is the empty set or it occurs in exactly one of the partition's equivalence classes (possibly that of the empty extension), as demonstrated in the `{u,v}` and `{u,v,w}` example in the previous paragraph.



We thus process a `Lacks` constraint `b , x , es , d` via the following steps:


1. Apply the equality store substitution to `x` in order to reduce `x` and perhaps extend `es`.
1. Use `[Lacks<]` and `[LacksI]` to remove all elements `e` from `es` for which `e < b` or `e > b`. The simplified constraint is `b , x , (es - {e}) , (d - 1)` or `b , x , (es - {e}) , d`, respectively.
1. Simplify the constraint by appeal to any existing constraints `b , Set0 , es0 , d0` where `es0` is a subset of `es`. The simplified constraint is `b , x , (es - es0) , (d - d0)`.
1. Simplify the constraint by appeal to any existing constraints `b , x , es0 , d0` where `es0` is a subset of `es`. The simplified constraint is `b , Set0 , (es - es0) , (d - d0)`.
1. Otherwise, insert it into the store, which is its own process described next.


To insert a fully simplified new constraint into the store:


1. Discharge the new constraint as redundant if it exactly matches an old constraint.
1. Use `[LacksE]` "backwards" to simplify any old constraints in the store by appeal to the new constraint. Any old constraint being simplified is removed from the store, simplified, and then added to the worklist of fully simplified constraints to add.
1. After simplifying all matching old constraints, insert the new constraint into the store's maps and then start again for the next item in the worklist.


(Contrary to the implication of the above steps, the implementation should accumulate a difference `dd` instead of continually updating `d` during simplification. This makes it possible to bind `d` to `dd` when discharging a redundant constraint (for which we `d - dd = 0`).)


### Alternate and Iterate the Phases



It may be possible to process an equality constraint only by appealing to a `Lacks` constraint that differentiates two elements on either side of an equality (`(x .& a ~ y .& b,Lacks (Set0 .& a) b)`). And the substitution of the equality store is the first step when simplifying a given `Lacks` constraints. We therefore should alternate: extend the equality store as much as possible, then extend the `Lacks` constraints as much as possible, then back to equality store, and so on as long as the stores are gaining information.



I have not considered how to update the `Lacks` store for updates to the equality store's substitution, but it will require some care. Maybe just remove all affected mappings, expand their root, and reprocess them "from scratch"?


### Implementation Note: There are two different orderings on types!



There are two relevant orderings on types.


1. O1 `x < y` is deterministic. It is therefore partial when there are free variables in `x` and/or and `y`, because free variables have no invariant information that could determine their order. But note that "partial" does not mean "undefined for all arguments": for example, `[a]` and `Int` are related (I don't know which --- and it ultimately doesn't matter -- but we have either `[a] < Int` or `Int < [a]` by O1.).

1. O2 `x < y` is "non-deterministic" and total. Because it's total, it can relate free variables. This makes it non-deterministic because free variables have no invariant information that could determine their order. In particular, GHC compares free variables by their `Unique`, which is unpredictable. In particular, the unique for the "same" variable changes unpredictably between each invocation of a typechecker plugin to the next but should be consistent ("effectively deterministic") during a single invocation.


I mentioned O1 rather explicitly in the `Lacks` Phase section above. My implementation also relies on O2 to implement the "set notation" (e.g. `{u,v}`) that I used without explanation also in the `Lacks` Phase section above via `Data.Set`.


### Querying the Store to Simplify/Solve Wanteds



Wanted constraints are handled in much the same way as givens, excluding the steps that involve updating the store.


### Converting the Store Back to Constraints



When simplifying givens, for example, the idea is to process the constraints to create a store and if processing performed any simplifications, replace the processed constraints with the new constraints that represent the content of the store. This is somewhat complicated for two reasons.


1. The implicit variables occur in both stores, because both store builders apply the substitution. So these will have to be removed.
1. The "simplified" constraints will be much larger than necessary: again because both apply the substitution, a constraints would include `(x ~ y .& v,Lacks (x .& u .& v) b)` instead of the more compact `(x ~ y .& v,Lacks (y .& u) b)`.


The implicit variables must be removed. The "inflation" issue is just a quality of life concern and there's no clear right answer: should inferred contexts and error messages put as much information as possible into each constraint (thereby risking redundancies) or as little (compact but less transparent)?



For the equality store, my rough plan is to handle both issues (preferring compactness over transparency) in one step: before writing out the equalities in the store, for each implicit variable root, I'll compact that root's equality constraints by finding a minimum spanning tree of the weighted fully connected graph in which each node is a variable in the substitution domain whose mapping extends that root and each weight is the total `typeSize` of the union of their extension minus the intersection of their extension and emit the MST's edges as the simplified equality constraints. For explicit variable roots and the empty set root, no processing is strictly necessary, but the same algorithm would compact their representation, I think.



I haven't considered the `Lacks` store yet. I haven't convinced myself that a greedy algorithm would find the minimal way to express a set form.


