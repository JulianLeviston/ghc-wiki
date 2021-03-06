## Representing closure-converted types as indexed types



The idea is to use a class


```wiki
class CC a where
  data CConv a        -- closure converted 'a'
  to :: a -> CConv a
  fr :: CConv a -> a
```


The most interesting instance is that for functions, which reads


```wiki
data Clo a b = forall e. Clo (e -> a -> b) e

class (CC a, CC b) => CC (a -> b) where
  data CConv (a -> b) = CCArrow (Clo (CConv a) (CConv b))
  to f = CCArrow (Clo (\_ -> f) ())
  fr (CCArrow (Clo f e)) = f e
```


For all basic types, we want something like the following:


```wiki
instance CC Int where
  newtype CConv Int = CCInt Int
  to = CCInt
  fr (CCInt x) = x
```


In fact, we want something like that for all types that are defined in modules that are not closure-converted.  Unfortunately, associated *data* types cannot have defaults.  (Also note that we use here a `newtype instance` for a `data` family.  Something, which should be easy to add to GHC's implementation of associated types, but isn't allowed right now.)


### User-defined types



When we encounter a data type definition for `T a1 .. an` in a closure converted module, we need to generate a `CC (T a1 .. an)` instance of `CC`.  The right hand side of `CConv (T a1 .. an)` will have the same form as that of `T`, but with all data constructors renamed and with all types `t` replaced by `CConv t`.  If no arrows occur directly or indirectly in the definition of `T`, we could instead use the same *default* instance as outlined for basic types above, as an optimisation.


### Import and export



To export and import the data constructors of `CConv` instances, we can just export and import `CConv(..)`.


### Higher-kinded types



During our brain storming, we thought that we might need `CConv` on higher-kinds, too - i.e., if we have `CConv (T a1 .. an)`, we also want `CConv T`.  The idea was that if we have 


```wiki
data U t = forall a. U a (a -> t a)
data T a = ...
```


the translation of `U T` requires `CConv T`.



However, I am not so convinced anymore that we really need it (at least as long as disregarding the interaction with non-closure-converted code).  After all, we need `CConv` essentially in two places:


1. If we have `f :: tau`, we produce `f' :: CConv tau`.
1. If we have `data T = MkT tau`, we produce `data CConv T = CCMkT (CConv tau)`.


In both cases, we only apply `CConv` to tau types.  (In GHC's sense of tau types.)



The only place, where that's not the case that I can see at the moment is when the closure-converted module defines (or uses?) constructor classes.  Then, we need a corresponding constructor class for `CConv C` (where the `C` is the type constructor).  This doesn't seem to be a feature that we need to cover in the near future.



In any case, we might be able to use a family of classes (such as with `Typeable` and we also discussed some tricks we might be able to play with associated synonyms).


### Contexts: 1. dealing with signature contexts



Something we didn't think about much so far is how to closure convert functions that have a class context; so, if we have


```wiki
inc :: Num a => a -> a
inc = (+1)
```


what is the signature of the closure converted version?  We can't use


```wiki
inc :: CConv (Num a => a -> a)
```


as associated types don't support that.  We probably want


```wiki
inc :: Num a => CConv (a -> a)
-- OR
inc :: Num (CConv a) => CConv (a -> a)
```


As closure conversion happens on Core code, we need to be careful, as the context is not visible as such in the *core view* of types.  (An alternative view point is that as closure conversion happens on Core, we don't want to treat dictionaries specially.  However, interface files contain the *source view*  of a signature, so we need to deal with closure conversion of contexts in some way, independent of whether we use associated types...unless we manage to keep closure conversion entirely out of interface files.)


### Contexts: 2. implicit `CC` contexts



Our main problem right now is the following.  If we have 


```wiki
foo :: tau
foo = e
```


we want to generate


```wiki
foo = fr fooCC

fooCC :: CConv tau
fooCC = <convert> [[e]]
```


Unfortunately, this is not type correct if `tau` contains type variables; e.g.,


```wiki
id :: a -> a
id = \x -> x
```


generates


```wiki
id = fr idCC

idCC :: CConv (a -> a)
idCC = ...
```


as we now have `id :: CC a => a -> a`.


### Additional notes



When we closure convert a module `B`, importing functions from a module `A`, we may be able to completely ignore whether `B` was closure converted itself (that would simplify matters enormously!)  The idea is that if `B` wants to use a function `A.foo` in closure converted code, it simply uses `(to A.foo)`.  If `A` is closure converted, we will have `A.foo = fr A.fooCC`.  Now a simple rewrite rule can eliminate the back and forth conversion (after inlining):


```wiki
{-# RULES
"to/fr" forall f. to (fr f) = f
 #-}
```


We still have to now whether a client module was closure converted to optimise the treatment of data constructors.  Otherwise, we would have to apply `fr` to every scrutinee of a case expression (and `to` to all variables bound in a case branch).


