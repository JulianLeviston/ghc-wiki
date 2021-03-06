


# Roles, Abstraction & Safety



GHC 7.8 introduced a new mechanism, roles, for implementing `GeneralizedNewtypeDeriving` safely. Roles solves a big issue with GND, type-safety. Previously, GND could be used to generate an `unsafeCoerce` function, which can easily segfault a program.



Please also refer to the main [Roles page](roles).



However, GND had a second issue, its ability to break module boundaries. How this should be handled with the new roles infrastructure and what the default should be was a major point of discussion before GHC 7.8 and after.



The design chosen settled on enabling easier use of GND over enforcing module boundaries. This document tries to summarize the situation and propose alternatives for future GHC versions.



A major focus is on improving the situation of Roles & GND for Safe Haskell. We'll start with solutions, but please read the rest of the document to understand the problem better.


## Roles & Safe Haskell



Roles are an unfortunate mechanism for control right now. Since
representational is the default role for most type constructors, to enforce
invariants on abstract data types, library authors need to set their type
constructors to have nominal roles.



This requires that library authors understand roles to enforce what they expect
to happen according to Haskell2010 semantics and opens a very easy to forget vulnerability in code.



Due to this, Safe Haskell so far has disallowed both `Data.Coerce` and GND.


## Approaches to the Problem



1) **Do Nothing** -- Keep Roles & GND unchanged, keep them unsafe in Safe Haskell.



2) **Accept as Safe** -- Keep Roles & GND unchanged, accept them as safe in Safe Haskell and warn users that they need `nominal` role annotations on ADTs.



**RAE** What do you mean by *warn* here? Is this warning done via social outreach, or from GHC itself? If from GHC, when would the warning trigger? Adding a "you're potentially violating abstraction" warning has been very tempting, but we've never been able to figure out quite the right trigger for it. 



Separately, I'll add the observation that if roles were widely understood by library-writers, this would be the "obvious" option. The only reason roles are considered unsafe is that library-writers might accidentally omit a needed annotation. If roles were widely understood, then omitting a role annotation would be as much the library-writer's fault as omitting an export list. To pursue this further, it might be interesting to see what percentage of packages with reverse dependencies on Hackage have role annotations. If this percentage is quite high, then it's an argument that library-writers are with the program and know of the need to protect abstractions.
**End RAE**



3) **In-scope constructor restriction for lifting instances** -- The newtype constructor restriction for unwrapping instances could be extended to both data types, and the lifting instances of `Data.Coerce`. This is, GND & Coercing under a type constructor is allowed if (a) all involved constructors are in scope, or (b) the constructors involved have been explicitly declared to allow coercion without them being in scope. I.e., (b) allows library authors to opt-into the current GHC behavior.  This would require new syntax, probably just an explicit `deriving Coercible` statement.



**RAE** This seems straightforward to implement. We just need to decide on a syntax. `deriving Coercible` is one such syntax. Another is simply the presence of a role annotation. **End RAE**



4) **Change default role to nominal** -- This will prioritize safety over GND, and the belief is that it may break a lot of code. Worse, that it will be an ongoing tax as role annotations will be needed to enable GND.



5) **Nominal default when constructors aren't exported** -- When a module doesn't export all the constructors of a data type, then the type parameters of the data type should default to nominal. This heuristic seems to capture somewhat the intention of the user, but given the practice of defining an `Internal` module that exports everything, it seems of limited use.



6) **Nominal default in future** -- Add a new extension, `SafeNewtypeDeriving` that switches the default role to nominal, but continue to provide a deprecated `GND` extension to help with the transition. The claims in support of representational roles as default though believe that nominal by default has an ongoing, continuous tax, not just a transition cost. So it isn't clear that any scheme like this satisfies that argument.



**RAE** This doesn't make sense to me. The roles are set at definition time, yet that extension looks like it would happen at the use site. Perhaps I'm misunderstanding. **End RAE**



7) **Safe Haskell Specific** -- Many of the above approaches could be adopted in a Safe Haskell specific manner. This isn't ideal as it makes safe-inference harder and Safe Haskell less likely to remain viable going forward. [
Richard suggests one such idea](https://mail.haskell.org/pipermail/haskell-cafe/2015-April/118999.html).



8) **Warn when representational and constructors not exported** -- This would be similar to 5, but rather than switch a types default for roles to nominal when its constructors aren't exported, we simply warn the user.


### Subtleties of 3



Option 3 seems like a good choice that matches well with the expectations of developers. It obeys a good rule of, "Could I implement this by hand?", and `Data.Coerce` simply acts as an optimization of what is already possible. It also resolves being able to use `coerce` internally as an optimization, while disallowing it's use externally. Something not easily expressible today.



We'd want some new syntax for allowing the old behavior (Role syntax doesn't apply as this restriction or non-restriction of constructors in scope is somewhat orthogonal to roles. I.e., it applies to the polymorphic type, while roles apply to the type parameters).



However, it can get tricky. Take this [
example](https://mail.haskell.org/pipermail/glasgow-haskell-users/2013-October/024368.html):


```wiki
module A where
data S a b = S1 a | S2 b
data T a b = MkT (S a b)

module B where
import A ( {- what goes here? -} )

class C a where
  mkT :: T Bool a

instance C Int where ...
newtype Age = MkAge Int deriving C
```


What constructors should be required to be in scope to derive the `C Age` instance? If typing by hand, it would require `MkT` and `S2` but not `S1`. This matches the rule, but as you can see, it can get tricky.



A simple approach would be to over-approximate and require for each involved type, **all** constructors are in-scope, or, the type has been marked with the new syntax of allowing coercion without constructors. **RAE** Yes, please! **End RAE**



Another issue is the syntax and how it would interact with Roles. The easiest seems to be:


```wiki
data MinList a = MinList a [a] deriving Coercible
```


Which would allow clients to use the default instances of `Coercible` regardless of if the constructors are in scope or not. Syntax beyond this is interesting in its finer grained control over `Coercible`, but gets complicated quickly and conflicts with role annotations. **RAE** I vote for the simple thing until real clients start shouting **End RAE**


#### Syntax for current behaviour



**RAE**



Despite my earlier comments, there is some annoying engineering here. For good reasons, we simply can't have Coercible instances floating about. So, whether or not you say `deriving Coercible` is something that's properly recorded on the `TyCon`. But the `TyCon` is built and fixed in stone before we see `deriving Coercible`. And, the `deriving` syntax suggests that another module who has access to all of `T`'s constructors would be able to use `StandaloneDeriving` to say `deriving instance Coercible T` (what terrible syntax -- that's utterly ill-kinded; I hate it), and then it's clearly too late to affect `T`.



**End RAE**


#### Recursive Checking Required



Consider


>
>
> data A a = MkA a    -- inferred to have an R role
> data B b = MkB (A b)   -- inferred to have an R role
>
>


Suppose MkA is not in scope, but MkB is. Now, suppose we want to coerce (B Age)
to (B Int). This will check if all of B's constructors are in scope, and they
are. Then, we decompose to check if Age coerces to Int. But this is not what
we want!



Option (3) should fail above as we should also require that A is in-scope,
which gives us a big, painful recursive check and may require users import
modules just to get their datatype definitions.


#### Optimising Recursive Check (Can-Import-Capability)



The recursive check could be optimised by tracking for each module, which data
constructors it has access to through it's imports recursively. This should
resolve the recursive check immediately and does so without requiring users
import random modules (since we determined they could import a module to
satisfy the check, so the actual import isn't required).



There is an interesting question with Safe Haskell though as it creates
restrictions around imports. Let's imagine the setup: Module A, imports Module
B, imports Module C. Let's also refer to the list of constructors that a module
could possibly access as `PA(module)`.



A \<- B \<- C



Now, we have the following situations:



A) B is Safe -- then A should inherit PA(C) since if B is safe, then C must be


>
>
> Safe or Trustworthy, so A can defiantly import C.
>
>


B) B is Unsafe -- then A should inherit PA(C). Since if A can successfully


>
>
> import B, A is Unsafe, so it's regular Haskell behavior.
>
>


C) B is Trustworthy, C is Safe / Trustworthy  -- A should inherit PA(C).



D) B is Trustworthy, C is Unsafe -- A \*shouldn't\* inherit PA(C). Since A may


>
>
> not be able to import C itself.
>
>


For the non-[SafeHaskell](safe-haskell) situation, rather than do all this tracking stuff, it
just reduces to the policy that when you don't export all constructors for a
type, it gets nominal roles by default. Otherwise it gets the usual roles.



One big complication here is that modules can be \*Internal\* or \*External\* when
exported from a package. This would need to be handled and could complicate the
implementation quite a lot.


### Questions to Answer


1. How common is exporting just a subset of constructors? I.e., how well will over-approximating which constructors are needed by assuming all constructors for a type work?

1. How common are `Coercible` constraints?

1. How common is it to \*not\* export all constructors for a data-type, \*not\* declare an explicit role, yet desire that the role is \*representational\*?  I.e., how much would break if we changed the above to be nominal by default when you don't export all constructors?

## Problem Pre-GHC-7.8



[GND Pre-GHC-7.8](safe-roles/pre78-gnd)



Due to both the type-safety and abstraction issues, GND was considered unsafe in Safe Haskell.


## Background Reading



Userguide:


- [
  https://downloads.haskell.org/\~ghc/latest/docs/html/users\_guide/roles.html](https://downloads.haskell.org/~ghc/latest/docs/html/users_guide/roles.html)


GHC Wiki:


- [
  https://ghc.haskell.org/trac/ghc/wiki/Roles](https://ghc.haskell.org/trac/ghc/wiki/Roles)
- [
  https://ghc.haskell.org/trac/ghc/wiki/Roles2](https://ghc.haskell.org/trac/ghc/wiki/Roles2)


Email Threads:


- "Default Roles" -- [
  https://mail.haskell.org/pipermail/glasgow-haskell-users/2013-October/024360.html](https://mail.haskell.org/pipermail/glasgow-haskell-users/2013-October/024360.html)
- "Role Signatures in Libraries" -- [
  https://mail.haskell.org/pipermail/libraries/2013-November/021707.html](https://mail.haskell.org/pipermail/libraries/2013-November/021707.html)
- "We need to add role annotations for 7.8" -- [
  https://mail.haskell.org/pipermail/libraries/2014-March/022321.html](https://mail.haskell.org/pipermail/libraries/2014-March/022321.html)


Tickets:


- "Inferring Safe mode with GeneralizedNewtypeDeriving is wrong" -- [
  https://ghc.haskell.org/trac/ghc/ticket/8827](https://ghc.haskell.org/trac/ghc/ticket/8827)
- "GeneralizedNewtypeDeriving is still not Safe" -- [
  https://ghc.haskell.org/trac/ghc/ticket/8745](https://ghc.haskell.org/trac/ghc/ticket/8745)
- "Require -XIncoherentInstances to write role annotations on class definitions" -- [
  https://ghc.haskell.org/trac/ghc/ticket/8773](https://ghc.haskell.org/trac/ghc/ticket/8773)
- "Incoherent instances without -XIncoherentInstances" -- [
  https://ghc.haskell.org/trac/ghc/ticket/8338](https://ghc.haskell.org/trac/ghc/ticket/8338)

## Roles Overview



[Roles Overview](safe-roles/roles-overview)



Consider the code:


```wiki
newtype Set a = MkSet [a]
```


This gives the following instances of `Coercible`:



First, the **unwrapping instances**:


```wiki
instance Coercible [a] b => Coercible (Set a) b
instance Coercible a [b] => Coercible a (Set b)
```


These are only available when the `Set` constructor, `MkSet`, is in scope. This was specifically done to preserve some notion of abstraction.



Secondly, you get the **lifting instances**:


```wiki
instance Coercible a b => Coercible (Set a) (Set b)
```


This instance is produced for both `data` and `newtype` types and is available for both regardless of if or if not the constructors of the type are in scope.



The only way to control the availability of the **lifting instance** is to use a role annotation:


```wiki
type role Set nominal
```

## Problem GHC-7.8+



Consider MinList:


```wiki
module MinList (
        MinList, newMinList, insertMinList,
    ) where

data MinList a = MinList a [a] deriving (Show)

newMinList :: Ord a => a -> MinList a
newMinList n = MinList n []

insertMinList :: Ord a => MinList a -> a -> MinList a
insertMinList s@(MinList m xs) n | n > m     = MinList m (n:xs)
                                 | otherwise = s
```


This is perfectly valid and reasonable code to write. However, a consumer of MinList could now write:


```wiki
module Main where

import MinList
import Data.Coerce

newtype MInt = MInt Int deriving (Eq, Show)

instance Ord MInt where
    compare (MInt a) (MInt b) = compare b a

main =
  let ints  = [MInt 1, MInt 3, MInt 5, MInt 2] :: [MInt]
      mintS = foldr insertMinList (newMinList $ MInt 3) ints
      intS  = coerce mintS :: Set Int
  in do
    print mintS
    print intS
```


Using the `coerce` function in a simple way to break invariants. 


## Why is this a problem?



It's reasonable to say that this all is fine. If library writers want to enforce invariants, then they simply add a role annotation. This is unsatisfying as:


- It breaks the expected semantics of Haskell2010. A library writer needs to understand a GHC specific extension to get the behavior that they expect.
- It sets the default to be that type-class invariants are not enforceable.
- Reasoning about an abstract data type by inspecting the export list and types is no longer enough, you need to look at the role annotations or lack of them.

## Role Subtleties



Below we'll outline some subtleties of Roles.


### Data.Coerce isn't Needed



The way that GND is implemented now, is through the `coerce` function essentially. **RAE** Not "essentially." It really *is* just the `coerce` function! **End RAE** For example:


```wiki
class Default a where
    def :: a

instance Default Int where
    def = 0

newtype MInt = MkInt Int

deriving instance Default MInt
```


The GND instance `Default MInt` is equivalent to writing out by hand the following instance:


```wiki
instance Default MInt where
    def = coerce (def :: Int)
```


Because of this, as before we can use GND to create coercion functions without an explicit import of the `Data.Coerce` module. For example,with the `MinList` example, we can simply use a typeclass as before to create a coercion function:


```wiki
class IntIso t where
    intIso :: MinList t -> MinList Int

deriving instance IntIso Mint
```


Unlike in GHC 7.6, we need to use more concrete types due to the role mechanism, but we can still derive coercion functions.


### Constructors aren't needed



Although `newtype` has the restriction (for unwrapping instances only) that the constructor must be in scope, this can be side-stepped by declaring a `Coercible` constraint in your function context, and convincing the calling module to generate the dictionary for you if it has access to the constructors.



For example, consider a `Secret` type:


```wiki
module Secret (Secret(..)) where

newtype Secret = MkS String
```


Then consider the module that imports it without access to the constructor but uses `Coercible` still:


```wiki
module UnsafeUser where

import Data.Coerce
import Secret (Secret())

runPlugin :: Coercible a String => a -> IO ()
runPlugin s = do
    let sv = coerce a
    putStrLn $ "Secret is: " ++ sv
```


A module may try to run it as below:


```wiki
module Main where

import Secret
import UnsafeUser

main :: IO ()
main = do
    let s = Secret "my secret"
    (runPlugin :: Secret -> IO ()) s
```


Due to how GND and `coerce` is implemented through type-classes, you end up with a error-prone situation when trying to secure code from accessing `Coercible` dictionaries due to the ability of modules to defer their creation to others.



One approach (without GHC changes), is to make sure that you have a 'shim' module between you and any untrusted code, and carefully audit the shim to ensure it doesn't have the ability to create dictionaries that are security sensitive.



Another would be to regard `Coercible` constraints as unsafe since they defer checking of safety constraints. This seems appropriate given that the `Coercible` type-class approach is a bit of a hack anyway.


### Type-classes are nominal by Default



By default, type parameters for type-classes have nominal roles. This protects against the creation of incoherent instances. This appears to be safe and the correct decision. For example:


```wiki
{-# LANGUAGE RoleAnnotations #-}
{-# LANGUAGE IncoherentInstances #-}
module ExpDicts_Sub (
    MEQ(..), normMEQ
  ) where

-- Requires we explicitly use representational
type role MEQ representational
class MEQ a where { meq :: a -> a -> Bool }

instance MEQ Char where { meq _ _ = True }

normMEQ :: MEQ a => a -> a -> Bool
normMEQ a b = a `meq` b
```


Can then be used by another module as follows:


```wiki
{-# LANGUAGE GADTs #-}
module Main where

import Data.Coerce
import ExpDicts_Sub

data C a where
  C :: MEQ a => C a

newtype CChar = CChar Char

instance MEQ CChar where
  meq _ _ = False

dictChar :: C Char
dictChar = C

dictCChar :: C CChar
dictCChar = C

dictChar' :: C Char
dictChar' = coerce dictCChar

expMEQ :: C a -> a -> a -> Bool
expMEQ C a b = a `meq` b

main :: IO ()
main = do
  print $ expMEQ dictChar  'a' 'a'                   -- True
  print $ expMEQ dictCChar  (CChar 'a') (CChar 'a')  -- False
  print $ expMEQ dictChar' 'a' 'a'                   -- False
```

### Unwrapping Instances require Constructor in Scope



Newtype introduces **unwrapping instances** as well as the standard lifting instance. Unwrapping instances however, are only available when the constructor for the newtype is in scope.



For example:


```wiki
module Sub (
    T(), mkT, castTs
  ) where

import Data.Coerce
import Sub_Evil

newtype T a = T a deriving (Eq, Ord, Show)

mkT :: a -> T a
mkT = T

castTs :: [T a] -> [a]
castTs = coerce
```


Within the module `Sub` we can coerce from `T a` to `a` as the construct, `T`, is in scope. However, in a consumer of `Sub` we cannot since the constructor isn't exported:


```wiki
module Consumer where

import Data.Coerce

import Sub

-- Can't write since T constructor not in scope!
castTs' :: [T a] -> [a]
castTs' = coerce
```


The above will fail to compile.


### Coerce Dictionary Access



One subtle point though is that this access to the `coerce` function is being controlled through type-classes, and so the dictionaries that implement then at run-time. This can be tricky to get right due to the implicit nature of them.



For example, if `Sub` imported the module `Sub_Evil` and called the following code:


```wiki
-- module Sub ...

runPlugin :: IO ()
runPlugin = plugin (T 1 :: T Int) (2 :: Int)
```


And `Sub_Evil` is defined as follows:


```wiki
{-# LANGUAGE ScopedTypeVariables #-}
module Sub_Evil (
    plugin
  ) where

import Data.Coerce

-- Can gain access to dictionary without constructor if passed in at location
-- that can access constructor.
plugin :: forall a b. (Show a, Show b, Coercible a b) => a -> b -> IO ()
plugin x y = do
  putStrLn $ "A : " ++ show x
  putStrLn $ "B : " ++ show y
  putStrLn $ "A': " ++ show (coerce x :: b)
```


Then we can coerce from `T a` to `a` without access to the constructor for `T`!



This is worrying, but appears reasonably hard to exploit as it relies on using polymorphic types in the definition of `plugin`. If we replace `plugin` with the type `plugin :: Coercible (T Int) Int => T Int -> Int -> IO ()`, then the module fails to compile as the constructor-in-scope check is enforced.



Perhaps someone smarter though can figure out how to gain access to the dictionary indirectly like this while still using concrete types.



**RAE** I'm not terribly worried about this exploit, as evidence that something strange is going on is in the type signature. **End RAE**


### GND and Super-classes



An interesting question is, what happens with GND and a type-class with a super-class? Well let's see:


```wiki
{-# LANGUAGE GeneralizedNewtypeDeriving #-}
module Main where

data T = MkT Int deriving (Show)

class C a where
  op :: (Show a) => a -> String

instance C T where
  op x = "T: " ++ show x

-- derived C uses `Show T` instance, not derived `Show TT` instance...
newtype TT = MkTT T deriving (Show, C)

main :: IO ()
main = do
  putStrLn $ op $ MkT 2
  putStrLn $ op $ MkTT $ MkT 2
```


So the GND instance for `TT` uses the `Show` instance for `T` rather than the show instance for `TT`.



**RAE** This one is a red herring -- nothing to do with GND. If I remove the `deriving C` from `TT` and instead write


```wiki
instance C TT where
  op (MkTT t) = op t
```


I get the same behaviour as the original code. Note: no `coerce`! **End RAE**


### Nominal prevents optimisations



Use roles is a global property of the type. So while it may be reasonable as a library writer of an ADT, `Set`, to want to use the `coerce` function internally, but disallow it externally, you currently can't do this.



Newtype's provide this property somewhat, but as pointed out, only for their **unwrapping instances**, the **lifting instances** are only controllable through roles.



**RAE** Yes, but you can have a `SetInternal` datatype with whatever roles you please, use that throughout your library, and then have `newtype Set x y z = MkSet (SetInternal x y z)` with more restrictive roles and without an exported constructor. This is perhaps less than ideal, but not so painful that we need to worry about inventing something better. **End RAE**


