## Rationale



`unsafeCoerce#` allow unsafe coercions between different types.
According to the [
documentation in GHC.Prim](http://hackage.haskell.org/package/ghc-prim-0.3.1.0/docs/GHC-Prim.html#v:unsafeCoerce-35-), it is safe only for following uses:


- Casting any lifted type to `Any`

- Casting `Any` back to the real type

- Casting an unboxed type to another unboxed type of the same size (but not coercions between floating-point and integral types)

- Casting between two types that have the same runtime representation. One case is when the two types differ only in "phantom" type parameters, for example `Ptr Int` to `Ptr Float`, or `[Int]` to `[Float]` when the list is known to be empty. 

- Casting between a newtype of a type `T` and `T` itself.  (**RAE:** This last usecase is subsumed by `Data.Coerce.coerce`, at least when the newtype constructor is in scope.)


However GHC doesn't check if it's safe to use `unsafeCoerce#` as a result bugs can appear, see [9035](http://gitlabghc.nibbler/ghc/ghc/issues/9035).
In order to solve this problem a solution was proposed by Simon in [9122](http://gitlabghc.nibbler/ghc/ghc/issues/9122), quote:


>
>
> I think it would be a great idea for Core Lint to check for uses of `unsafeCoerce` that don't obey the rules. It won't catch all cases, of course, but it would have caught [\#9035](http://gitlabghc.nibbler/ghc/ghc/issues/9035). 
>
>


This proposal is about implementation of the task.


## Progress



Current progress could be found on [
D637](https://phabricator.haskell.org/D637)([
https://phabricator.haskell.org/D637](https://phabricator.haskell.org/D637)). It implements
proposed checks modulo few questions mentioned in this proposal. 



The solution introduces following changes in the core specification, `docs/core-spec/CoreLint.ott` in the source tree ([
PDF here](https://github.com/ghc/ghc/blob/master/docs/core-spec/core-spec.pdf)):


```wiki
G |-ty t1 : k1
G |-ty t2 : k2
((k1 <: OpenKind) /\ (k2 <: OpenKind)) \/ (compatibleUnBoxedTys t1 t2)
------------------------------------------------------------------------ :: UnivCo
G |-co t1 ==>!_R t2 : t1 ~R k2 t2
```


'compatibleUnBoxedTys' consists of following rules:


1. Coercions between lifted and unboxed are not allowed.  Reason: casting between pointers and non-pointers is likely to cause seg-faults if the garbage collector happens to run.


**SPJ** Actually it should be *unboxed* not *unlifted*.  It's wrong to cast between `Array# Int` and `Int#` because the former is a pointer and the latter is not.


1. If types are unlifted then their *size* should be equal, see `primRepSizeW` in [source:compiler/types/TyCon.hs](/trac/ghc/browser/compiler/types/TyCon.hs)[](/trac/ghc/export/HEAD/ghc/compiler/types/TyCon.hs)

1. If types are unlifted then they either should be both floating or both integral.  Reason: on many architectures, floating point values are held in special registers.


   


1. No coercions between vector ([wiki:SIMD](simd) SIMD pages) types are allowed at all. (Reason: there is no correct rules for such coercions)

1. If types are unboxed tuples then tuple `(# A_1,..,A_n #)` can be coerced to `(# B_1,..,B_m #)` if `n=m` and for each pair `A_i, B_i` rules 1-5 holds.


If any of those rules are violated then linter should produce a warning.
Checks are applied to both errors produced by user code and GHC itself. (see "User programs" section for additional discussion). 


## Possible Extensions


### Support tuples with different length



In some cases it may be reasonable coerce tuples with different length, or tuples to value `(# A, B #)` -\> `C` if they have
equal size.


### User programs



Should those check be applied only to internal GHC's transformations, for user programs should also be
checked?



**SPJ** Both ideally.  Emit a warning for user programs with visible problems.  And check in Lint.  Start with the latter.



**RAE** So that means that a warning would be issued, followed by a CoreLint failure. This violates the invariant that CoreLint catches only GHC's mistakes. I loosely agree with this approach here (because I want to allow users to do terrible things if they really want to), but we'll have to be careful about wording the error message that CoreLint spits out. This also implies that users who are actively trying to shoot themselves in the foot will have to avoid `-dcore-lint`, which is slightly dissatisfying. Maybe add a flag asking whether or not CoreLint should perform these checks? I guess my tension stems from the fact that we want to protect most users from mistakes and want to detect mistakes in GHC, while still allowing crazy things to happen. (Like still exporting [
this function](https://github.com/haskell/bytestring/blob/2530b1c28f15d0f320a84701bf507d5650de6098/Data/ByteString/Internal.hs#L599).)
 
**SPJ** Let's not make the best the enemy of the good.  Make the whole lot into warnings or something, if that would reassure you.  Mostly we are trying to identify smelly code; some of it might just possibly work.  On a particular processor, when the sun is shining.  


## Implementors



implementation: Alexander Vershilov / Qnikst



advisor: Richard Eisenberg / goldfire / RAE


## Questions


### Vectors



Coercion between vectors are should be clarified (under vectors I mean 'VecRep').



By "vector" we mean the (new) data types in GHC over which architecture-supported SIMD instructions work.  For example Intel SSE or ARM Neon instructions.   All this is documented on [the SIMD pages](simd).



Is it possible to coerce between vectors with elements of different length, but when vector size matches,
for example `Vector 4 Word8ElemRep` to `Vector 2 Word16ElemRep`?  **SPJ**: for now, at any rate, NO.  It's all very architecture-specific, and I have no idea what the rules are.  No is a simple answer!


---


## Old discussions


### Size of value



**SPJ** I can't make head or tail of this section, and I am pretty sure that this section is all wrong.  But we need Geoff Mainland to help us out.  



I think that the difference between "active size" and "real size" (concepts whose very existence I dispute) is caused by `VecRep`, which in turn is Geoff Mainland's support for vector instructions; I think the most up to date description is [SIMD](simd).  So we can have a type for a 4-vector of 32-bit quantities.  But these things may well be held in special registers, a bit like `Float`, and are probably not inter-coercible with anything else. 



For now, do something simple and conservative
 
**End of SPJ**



**Qnikst** I see your point and going to use this approach as a first step in implementing this proposal. However I want  to clarify my concerns in order to ask if they are valid.
Here is a concrete example: on 64bit platform word aligned size of `Int#` and `Int64#` sizes are equal (so we can coerce back and forth), this means that 
actual value of the `unsafeCoerce# ((a::Int#) +# (b::Int#)) :: Int64#` will depend on memory and register state, so it this call have to referential transparency.
And I don't know if it's the issue, if it is then we need to have some notion of "actual" size or "used" size, if not then plain and simple check of word size will be enough.
**End Qnikst**



**SPJ** I'm really sorry but I still don't understand.  Perhaps you are trying to say that there might be a *platform-specific* coercion between `Int64#` and `Int#`, which is valid on some platforms but not on others.  Perhaps.  Maybe.  But then you'd need to consult platform flags to figure it out. If you want to do that, go ahead.  But (a) it's nothing to do with alignment; just that `Int#` is simply identical to `Int64#` on a 64-bit platform.  and (b) the "active size" and "real size" terminology is quite confusing (do not use it). **End SPJ**



GHC has 2 different sizes: word aligned size of values, and active size in bytes that actually used.  **SPJ**: where do you see these two different sizes in GHC's source code?



Term 'active size' is used to describe number of bytes that value actually use, at this moment such numbers are used
in Vectors, see `primElemRepSizeB` in ([source:compiler/types/TyCon.hs](/trac/ghc/browser/compiler/types/TyCon.hs)[](/trac/ghc/export/HEAD/ghc/compiler/types/TyCon.hs)). The reasons about forbidding coercions between
values with a different active size is that in the rest bytes there will be garbage:



Hypothetical example for a Word16 on machine with 4-byte word size:


```wiki
   [A|A|W|W]
    0 1 2 3 
```


the real size of this value will be 1 word (4 bytes), active size will be (2 bytes), bytes 2,3 will contain garbage.
So if we will check *real size* to coerce from Word16 to Word32, this coercion will be allowed, even if resulting
value will have no sense. 



However equality between `active size` looks like overestimation, as it's completely safe to coerce from larger value
(`Word32`) to lesser (`Word16`). (**Qnikst**: is it really true in all cases signed/unsigned ints and floating values?)



The question is if we need to allow coercion between values with same word size, but different active size.
(Qnikst. current implementation forbids it, as values with different active size can contain garbage, however coercion from value with bigger active size to value with smaller potentially should be fine).


### Unboxed Tuples



A big question is how to treat unboxed tuples if they have same size, can we coerce between `(# Int, Int64 #)` and \`(\# Int64, Int \#)'?



**SPJ**: I think it should be ok to coerce from `(# a, b #)` to `(# c,d #)` if it's safe to coerce from `a` to `c`, and ditto `b` to `d`.  The tuples must have the same length. 
**Qnikst**: Do I understand correctly that it should not be possible to coerce `(# a, b, c #)` to `(# a, d #)` where `size b + size c == size d`?   **SPJ** Correct. Like I say "the tuples must have the same length". **Qnikst**: ok, then. I've asked because rule "it should be ok to coerce from `(# a, b #)` to `(# c,d #)` if it's safe to coerce from `a` to `c`, and ditto `b` to `d`" is more restictive then "tuples must have the same length". Now this question is cleared for me.
 



How to check is value is floating in this case?  **SPJ** I don't understand the question.
**Qnikst**: 'Coercion between unboxed ints and floats.' so we need to specify how it works for tuples.  **SPJ** I'm sorry I still don't understand what the issue is.  Can you just give a concrete example? 
**Qnikst**: assuming that size of Double = 2\*size of Float on target arch, can we coerce `(# Float#, Float#, Int# #)` to `(# Double#, Int# #)`? To \`(\# Float\#, Int\#, Float\# \#)? What is the easiest way to express this rule?  **SPJ** No no no.  Let me say it once again, as clearly as I can: **you cannot coerce between unboxed tuples of different arities**.  No. No. No. **End SPJ**.



As far as I understand coerce from `(# a, b #)` to `(# c, d #)` should not be allowed if either `a`, `b` or `c`,`d` violates rule.
However if `(# a, b, c #)` could be converted to  `(# a, d #)` then what should be the rules for `b`, `c` -\> `d`, as far as I understand
it's correct if `b`,`c`,`d`  are ints and not correct in any other case. 



(Qnikst. current implementation allow such coercions and doesn't check "floatiness" of values)


