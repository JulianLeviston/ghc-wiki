
This page discusses the interaction of Trees That Grow with derived for `Data`.  (But it might apply to manual instances for, say, `Outputable` too.)



This page is part of [ImplementingTreesThatGrow](implementing-trees-that-grow).


## Example



Here's our example:


```
  type family XOverLit p
  data HsOverLit p = OverLit (XOverLit p) (HsExpr p)

  data GhcPass (c :: Pass)
  deriving instance Typeable c => Data (GhcPass c)

  data Pass = Parsed | Renamed | Typechecked

  type instance XOverLit (GhcPass 'Parsed     ) = PlaceHolder
  type instance XOverLit (GhcPass 'Renamed    ) = Name
  type instance XOverLit (GhcPass 'Typechecked) = Type

```


We want a `Data` instance for this type.


### PLAN A



I propose


```
 deriving instance (Typeable p, Data (XOverLit (GhcPass p))) =>
    Data (HsOverLit (GhcPass p))

```


But that gives rise to big constraint sets; for each data constructor
we get another `X` field, and another constraint in the `Data` instance.



This is what is currently implemented in GHC master. It works, but every time the constraint set is enlarged as the next step of Trees that Grow goes in, the time taken to compile GHC increases.


### PLAN B



Alan would like to see


```
deriving instance Data (HsOverLit (GhcPass p))
```


which should have all the information available to the derivation process, since `p` can only have one of three values and each of them has a type family instance for `XOverLit`.



This "does not compile".



Alan has discovered that the three instances:


```
  deriving instance Data (HsOverLit (GhcPass 'Parsed ))
  deriving instance Data (HsOverLit (GhcPass 'Renamed))
  deriving instance Data (HsOverLit (GhcPass 'Typechecked))
```


**will** compile, but only with GHC 8.2.1, not with 8.0.2, due to a flaw in the standalone deriving process.



That is: instead of one `Data` instance for the `HsSyn` traversals,
make three.



\[ The spurious constraint problem was resolved by including `deriving instance Typeable c => Data (GhcPass c)`, as recommended by \@RyanGlScott \]


### PLAN E



Proceed as per Plan A, but move all the standalone deriving code to a new file, which will produce orphan instances.



Inside this, use CPP to detect a modern enough compiler (GHC 8.2.1) to generate via Plan B, otherwise fall back to Plan A.



Eventually improve the standalone deriving sufficiently that it is able to generate a single traversal, instead of the three.



So for day-to-day work ghc devs can use GHC 8.2.1, and we confirm is still works with GHC 8.0.2 less frequently.


### PLAN F



(This is a bit like Plan C/D, but without the stupid error.)



Suppose we declared out extension fields like this


```wiki
data HsOverLit p = OverLit (GhcExt (XOverLit p)) (HsExpr p)

newtype GhcExt x = Ext x
```


Now we could execute on Plan C by saying


```wiki
instance Data (GhcExt x) where
  gmapM f x = x
  ...
```


The downside is that every pattern match and construction would need to wrap and unwrap with `Ext`.



But actually we are going to need to do that anyway!  We are planning to abolish the alternation of `Located t` and `t` by putting the SrcSpan for the construct in its extension field. Like this


```wiki
data GhcExt x = Ext SrcSpan x
```


So we have to do this wrapping business anyway!



This approach would mean that every client of `HsSyn` would have to have a `GhcExt` in every node, with a `SrcSpan`. If that's not OK (and it probably isn't ok) we could do this:


```wiki
data HsOverLit p = OverLit (XOverLit p) (HsExpr p)
type instance XOverLit (GhcPass p) = GhcExt (GhcXOverlit p)
```


The price is that there are two type-level functions for each constructor.  The benefit is that we can do Plan C.



Actually, in the common case where GHC does not use any extension fields for a constructor `K` we could dispense with the second function thus


```wiki
type instance XK (GhcPass p) = GhcExt PlaceHolder
```

## Outdated/Infeasible Plans


### PLAN C (Infeasible)



It is still painful generating three virtually identical chunks of traversal code.
So suppose we went back to


```
  deriving instance (...)
                 => Data (OverLit (GhcPass p)) where…
```


We'd get unresolved constraints like `Data (XOverLit (GhcPass p))`.  Perhaps we
could resolve them like this


```
  instance Data (XOverLIt (GhcPass p)) where
     gmapM f x = x
     ...etc...
```


That is: a no-op `Data` instances.  Traversals would not traverse extension fields.
That might not be so bad, for now!


### PLAN D (Infeasible)



If there were cases when we really did want to look at those extension fields,
we still could, by doing a runtime test, like this:


```
  instance IsGhcPass p => Data (XOverLIt (GhcPass p)) where
     gmapM = case ghcPass @p of
                IsParsed -> gmapM
                IsRenamed -> gmapM
                IsTypechecked -> gmapM
     ...etc...
```


Here I'm positing a new class and GADT:


```
class IsGhcPass p where
  ghcPass :: IsGhcPassT p

data IsGhcPassT p where
  IsParsed      :: IsGhcPass Parsed
  IsRenamed     :: IsGhcPass Renamed
  IsTypechecked :: IsGhcPass Typechecked

instance IsGhcPass Parsed where
  ghcPass = IsParsed
...etc...
```


Now, instead of passing lots of functions, we just pass a single GADT value
which we can dispatch on.



We can mix Plan C and D.



Alan:



Note:


-  We are solving a compilation time problem for GHC stage1/stage2. The produced compiler has the same performance regardless of which derivation mechanism.

- Ideally we would like to end up with `data` instances for an arbitrary `OverLit p`, which is an outcome for Plan A. i.e. Plan A will play nice with external index types,for GHC API users.
