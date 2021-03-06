# Extensible Records



There seems to be widespread agreement that the current situation with regards to records is [
unacceptable](http://bloggablea.wordpress.com/2007/04/24/haskell-records-considered-grungy/), but the [
official GHC policy](http://haskell.org/haskellwiki/GHC:FAQ#Does_GHC_implement_any_kind_of_extensible_records.3F) is that there are too many good ideas to choose from - so nothing gets done!



The purpose of this page is to collect and discuss proposals for adding extensible records to GHC.


# Proposals


- [
  Wolfgang Jeltsch's records library](http://www.informatik.tu-cottbus.de/~jeltsch/research/ppdp-2010-paper.pdf) This implements records as a library rather than as a language extension.
- [
  A proposal for records in Haskell](http://research.microsoft.com/~simonpj/Haskell/records.html) (wherein [
  TRex](http://cvs.haskell.org/Hugs/pages/hugsman/exts.html#sect7.2) is rejected as having a high implementation cost)
- [
  A Polymorphic Type System for Extensible Records and Variants](http://web.cecs.pdx.edu/~mpj/pubs/polyrec.html)
- [ Scoped Labels](http://www.cs.uu.nl/~daan/download/papers/scopedlabels.pdf)
- [ Type Families](http://homepage.ntlworld.com/b.hilken/files/Records.hs)
- [ Heterogeneous Collections](http://homepages.cwi.nl/~ralf/HList/), see also [
  Keyword Arguments](http://okmij.org/ftp/Haskell/keyword-arguments.lhs)
- [
  Data.Record.hs](http://www.cs.kent.ac.uk/people/staff/cr3/toolbox/haskell/Data.Record.hs), expanded and documented version of the old Haskell prime ticket 92 attachment [
  Data.Record.hs](http://hackage.haskell.org/trac/haskell-prime/attachment/ticket/92/Data.Record.hs). (comment: my preferences would be (1) we should try to implement as many useful record operations, predicates, and invariants as we can, (2) we should try to unify the sets of operations into a coherent whole, (3) we should identify to what extent and in what form we need to have language and implementation support, and (4) users, not library providers, will decide which subsets of operations they use most; a library providing for as many common usage patterns as possible might have a chance of breaking the deadlock, and laying the groundwork for a future design that might actually have some users and experience behind it; these preferences appear to conflict with the intentions of the creator of this page)
- [
  Anonymous records](https://gist.github.com/nikita-volkov/6977841) -- "... something more like a tuple with ability to access its items by name."

# Functional References



Functional References are a cheap and cheerful technique for working with the existing (non-extensible) record system, and may be of interest to extensible record implementers. A good implementation can be found on [
Twan van Laarhoven's blog](http://twan.home.fmf.nl/blog/haskell/overloading-functional-references.details).


# Syntax



Purely for the sake of argument on this page, I propose the following syntax. Feel free to change or extend this if you can think of something better. Many of these conflict with existing Haskell operators, so can't be used in any concrete proposal.


- `{L1 = v1, L2 = v2, ...}` the constant record with field labels `L1, L2, ...` and corresponding values `v1, v2, ...`
- `{L1 :: t1, L2 :: t2, ...}` the type of the constant record with field labels `L1, L2, ...` and corresponding values of types `t1, t2, ...`
- `r . L` the value of the field labelled `L` in record `r`
- `t ::. L` the type of the field labelled `L` in record type `t`
- `r - L` the record `r` with field `L` deleted
- `t ::- L` the record type `t` with field `L` deleted
- `r + s` record `r` extended by adding all the fields of `s`. Many systems restrict to the case where `s` has constant shape.
- `t ::+ u` record type `t` extended by adding all the fields of type `u`. Many systems restrict to the case where `u` has constant shape.
- `r <- s` record `r` updated by replacing the all fields of `s`. Many type systems restrict to the case where `s` has constant shape.


By **constant shape** we mean that the field names of a record are given literally, though the values and types of the fields could be variables.


# Constant Record Types



An important difference between the various proposals is what constitutes a valid record, and similarly a valid record type. The key points are:


- Permutativity:: Are `{X :: Int, Y :: Int}` and `{Y :: Int, X :: Int}` the same type? The **Poor Man's Records** system distinguishes these two, which makes implementation much simpler, but means that any function which accepts permuted records must be polymorphic. 
- Repeated Fields:: Is `{X :: Int, X :: Int}` a valid record type? Both **Poor Man's Records** and **Scoped Labels** allow this type, but other systems consider this an error. 


Apparently the first of these is particularly controversial: see [
http://www.haskell.org/pipermail/haskell/2008-February/020177.html](http://www.haskell.org/pipermail/haskell/2008-February/020177.html)


# Label Namespace



The proposals which are implemented as libraries put labels in conid (at the value level) and tycon (at the type level). In other words they must begin with capital letters, not clash with any other constructor or type, and be declared before use. If we want to support labels as first-class objects, this is essential so that we can distinguish them from other objects.



The other proposals allow labels to be arbitrary strings, and distinguish them from other objects by context.



A third possibility (suggested on the mailing list) is to reserve a new syntactic class, such as identifiers starting with ', so that labels do not need to be declared before use.



This is related to the problem of Label Sharing: if the label `L` is declared in two different modules `M1` and `M2`, both of which are imported, do we have one label `L` or two labels `M1.L` and `M2.L`? Should there be a mechanism for identifying labels on import?



The **Heterogeneous Collections** system introduces user defined Label Namespaces. All the labels in a record must live in the same namespace, and within a namespace, labels are defined in a way which forces a linear ordering. This gives a kind of modularity which interacts with the module system in a complex way: is not clear (for example) what happens when two imported modules extend a namespace independently.


# Type Systems



The most important difference between the various record proposals seems to be the expressive power of their type systems. Most systems depend on special predicates in the type context to govern the possible forms of record polymorphism. As usual there is a trade-off between power and simplicity.


<table><tr><th>No predicates</th>
<td>You can get away without any predicates if you are prepared to allow records to have multiple fields with the same label. In the **scoped labels** system, the syntactic form of the type matches that of the record, so you can type the operators by

- `(. L) :: a ::+ {L :: b} -> b`
- `(- L) :: a ::+ {L :: b} -> a`
- `(+ {L = v}) :: a -> a ::+ {L :: b}`
- `(<- {L = v}) :: a ::+ {L :: b} -> a ::+ {L :: b}`

</td></tr></table>


<table><tr><th>Positive predicates</th>
<td>If you allow only positive information about records in the context, you can only type some of the operators. In **A proposal for records in Haskell** the condition `a <: {L1::t1, L2 :: t2, ...}` means that `a` is a record type with at least the fields `L1, L2, ...` of types `t1, t2, ...` respectively. You can type the following operators:

- `(. L) :: a <: {L :: b} => a -> b`
- `(<- {L = v}) :: a <: {L :: b} => a -> a`

</td></tr></table>


<table><tr><th>Lacks predicates</th>
<td>In order to type the other operators, we need negative information about records. In the Hugs **Trex** system the condition `a \ L` means that `a` is a record type without a field `L`. You can type all the operators if you restrict the right-hand sides to constant shape records:

- `(. L) :: a \ L => a ::+ {L :: b} -> b`
- `(- L) :: a \ L => a ::+ {L :: b} -> a`
- `(+ {L = v}) :: a \ L => a -> a ::+ {L :: b}`
- `(<- {L = v}) :: a \ L => a ::+ {L :: b} -> a ::+ {L :: b}`

</td></tr></table>


<table><tr><th>General predicates, using type families</th>
<td>The **Type Families** system uses three predicates: ` a `Contains` L ` means that `a` is a record type with a field labelled `L`; ` a `Disjoint` b ` means that `a` and `b` are record types with no fields in common; and ` a `Subrecord` b ` means that `a` and `b` are record types, and every field of `a` also occurs in `b` (with the same type). You can type all the operators:

- ` (. L) :: a `Contains` L => a -> a ::. L `
- ` (- L) :: a `Contains` L => a -> a ::- L `
- ` (+) :: a `Disjoint` b => a -> b -> a ::+ b `
- ` (<-) :: a `Subrecord` b => b -> a -> b `

</td></tr></table>


<table><tr><th>General predicates, using functional dependencies</th>
<td>The **Heterogeneous Collections** and **Poor Man's Records** systems achieve the same as the **Type Families** system, using the relational style of functional dependencies. In **Poor Man's Records**, `Select L a b` means that `a` is a record type with a field labelled `L`, and `b = a ::. L`; `Remove L a b` means that `a` is a record type with a field labelled `L`, and `b = a ::- L`; and `Concat a b c` means that `a` and `b` are record types with no fields in common, and `c = a ::+ b`. You can type the operators:

- ` (. L) :: Select L a b => a -> b `
- ` (- L) :: Remove L a b => a -> b `
- ` (+) :: Concat a b c => a -> b -> c `

</td></tr></table>



The `Subrecord` predicate and `<-` operator could easily be added. The difference between **Heterogeneous Collections** and **Poor Man's Records** is that **Poor Man's Records** makes no attempt to sort labels or remove duplicates, so for example `{X = 3, Y = 4}` and `{Y = 4, X = 3}` have different types, so are certainly not equal (the updated version of November 2007 supports record projection and permutation, among most other operations).


# Implementation and Language support



As it seems possible to implement most of the functionality in a library, there might be no need for a complex **extensible records** feature. Nevertheless, there are issues which are common to most proposals and which would best be addressed at the language and implementation level:


- type sharing: not specific to records, but crucial for record programming practice. If multiple modules introduce the "same" labels, means are needed to specify the equivalence of these types (cf [
  Haskell prime ticket 92](http://hackage.haskell.org/trac/haskell-prime/ticket/92)).

- partial evaluation of type class programs: to achieve constant time record field access. Again, this feature is not specific to records, but crucial for record programming practice.

- portability: it would be nice if extensible records libraries were portable over multiple Haskell implementations. That not only means that these implementations need to support the same features, but that they need to interpret these features in the same way (this is currently not the case for the interaction of functional dependencies and type class overlap resolution in GHC and Hugs).

- permutativity: The easiest way to implement permutativity of field labels is to sort them by some total ordering. Although this can be implemented using functional dependencies, it's complex and inefficient. Compiler support for a global order on tycons (based on fully qualified name, perhaps) would be very helpful. I have submitted a feature request [\#1894](http://gitlabghc.nibbler/ghc/ghc/issues/1894). Does this conflict with type sharing?


I have submitted another feature request, [\#2104](http://gitlabghc.nibbler/ghc/ghc/issues/2104), which would support permutativity and labels without declaration, as well as sort out sharing in the simplest possible way (all labels are global).


# Examples



Please put examples here, if possible using the above notation. The aim is to find out which features of the various systems are important in practice, so uncontrived examples which illustrate differences between the systems are wanted!



An example to show the need for extra polymorphism in unpermuted records:


```wiki
type Point = {X :: Float, Y :: Float}

norm :: Point -> Float
norm p = sqrt (p.X * p.X + p.Y * p.Y)

norm {Y = 3.0, X = 4.0}    -- this is a type error, because X and Y are in the wrong order

norm' :: (Select X a Float, Select Y a Float) => a -> Float
norm' p = sqrt (p.X * p.X + p.Y * p.Y)

norm' {Y = 3.0, X = 4.0}    -- this is OK, because norm' is polymorphic
```


Here is the example translated into `Data.Record`, and slightly expanded to demonstrate the flexibility with which one may specify precisely as much detail as wanted, without requiring a global label ordering (if that is wanted, we simply permute to the locally expected order). `norm1` has the type asked for above (no extra fields, specific order for expected fields), `norm2` has the most flexible type (only expected fields are specified), `norm3` shows a middle ground (no extra fields permitted, but order is irrelevant):


```wiki
data X = X deriving Show; instance Label X where label = X
data Y = Y deriving Show; instance Label Y where label = Y
data Z = Z deriving Show; instance Label Z where label = Z

type Point = (X := Float) :# (Y := Float) :# ()
-- a record is a list of fields, sorted to match expected order
record r = r !#& undefined
-- a point is a Point, if re-ordered
point = record ((Y := (3.0::Float)) :# (X := (4.0::Float)) :# ()) :: Point

norm1 :: Point -> Float
norm1 p = sqrt (p #? X * p #? X + p #? Y * p #? Y)

-- this is a type error, because X and Y are not in the expected order
-- test1a = norm1 ((Y := 3.0) :# (X := 4.0) :# ())
-- this works, because X and Y will be permuted into the expected order
test1b = norm1 (record $ (Y := (3.0::Float)) :# (X := (4.0::Float)) :# ())

norm2 :: (Select X Float rec, Select Y Float rec) => rec -> Float
norm2 p = sqrt ((p #? X) * (p #? X) + (p #? Y) * (p #? Y))

-- this is OK, because norm2 doesn't care about field order
test2a = norm2 ((Y := 3.0) :# (X := 4.0) :# ())
-- this is OK, because norm2 doesn't care about unused fields
test2b = norm2 ((Y := 3.0) :# (X := 4.0) :# (Z := True) :# ())

norm3 :: Project' rec Point => rec -> Float
norm3 p' = sqrt ((p #? X) * (p #? X) + (p #? Y) * (p #? Y)) where p = (record p')::Point

-- this is OK, because norm3 is doesn't care about field order
test3a = norm3 ((Y := (3.0::Float)) :# (X := (4.0::Float)) :# ())
-- this is a type error, because norm3 doesn't accept unused fields
-- test3b = norm3 ((Y := 3.0) :# (X := 4.0) :# (Z := True) :# ())
```


The more complex systems support first class labels. Here is an example using the Type Families system:


```wiki
labelZip :: ({n :: a} `Disjoint` {m :: b}) => n -> m -> [a] -> [b] -> [{n :: a, m :: b}]
labelZip n m = zipWith (\x y -> {n = x, m = y})
```

# How to proceed?



There are a lot of design decisions listed on this page, some of which inspire strongly-help opinions. It is clear that discussion will not resolve these, and we don't seem to have a lot of examples to help clarify matters. The deadlock is unchanged: we still have too many good ideas.



I believe the way forward is to implement several of the possible systems, and release them for feedback. To get users to actually try out the libraries, I think we need some concrete syntax for constant records, so I suggest we put in a feature request. Something like:


```wiki
expressions: {n1 = e1, ... , nn = en} --> mkUnderlyingRecord n1 e1 ( ... mkUnderlyingRecord nn en underlyingEmptyRecord)
types: {n1 :: t1, ..., nn :: tn} --> MkUnderlyingRecord n1 e1 ( ... MkUnderlyingRecord nn en UnderlyingEmptyRecord)
patterns: {n1 = p1, ... , nn = pn} --> (viewUnderlyingRecord n1 -> (p1, ...  ,(viewUnderlyingRecord nn -> (pn, viewUnderlyingEmptyRecord -> ())))
patterns: {n1 = p1, ... , nn = pn, ..} --> (viewUnderlyingRecord n1 -> (p1, ...  ,(viewUnderlyingRecord nn -> (pn, _)))
```


Libraries could then implement `mkUnderlyingRecord`, `underlyingEmptyRecord`, `MkUnderlyingRecord`, `UnderlyingEmptyRecord`, `viewUnderlyingRecord`  and `viewUnderlyingEmptyRecord` in whatever way is best. What do you think?


# See Also


- [ HaskellWiki](http://www.haskell.org/haskellwiki/Extensible_record)
- [
  Haskell'](http://hackage.haskell.org/trac/haskell-prime/wiki/FirstClassLabels)
