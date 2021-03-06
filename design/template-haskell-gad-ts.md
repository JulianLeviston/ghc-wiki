# A Proposal for using GADTs within the implementation of Template Haskell


## The problem



Haskell declarations and pragmas can appear in a variety of contexts, but not all declarations/pragmas can appear in all contexts. For examples, data declarations can appear only at top level; type family instances can appear at top-level, in classes (as defaults), and in instances, but never in `let`/`where` blocks; fixity declarations can be used anywhere other than in an instance declaration; etc. Yet, the Template Haskell `Dec` type monolithically includes *all* possible declarations. Currently, consumers of `Dec`s need to eliminate the possibility of `Dec`s in various contexts by emitting errors. Many users of Template Haskell might not know the intricate details of where which declarations can appear, making these "impossible" scenarios error-prone. And, with the solutions to [\#8100](http://gitlabghc.nibbler/ghc/ghc/issues/8100) (adding standalone-deriving) and [\#9064](http://gitlabghc.nibbler/ghc/ghc/issues/9064) (adding `default` type signatures to class declarations), the problem gets worse.


## A proposed solution



Use the type system to track which declaration form can be used where. First, declare


```wiki
data InTopLevel = YesInTopLevel | NoInTopLevel  deriving Typeable
data InClass    = YesInClass    | NoInClass     deriving Typeable
data InInstance = YesInInstance | NoInInstance  deriving Typeable
data InLet      = YesInLet      | NoInLet       deriving Typeable
```


These are all Boolean-like flags with more perspicuous names.



Then, pack these flags into a type:


```wiki
data DecContext = DC InTopLevel InClass InInstance InLet deriving Typeable
```


Now, define `Dec` as a GADT indexed by `DecContext`:


```wiki
data Dec :: DecContext -> * where
  FunD ::  Name -> [Clause] -> Dec (DC top NoInClass inst lt)
    -- ^ @{ f p1 p2 = b where decs }@
  ValD ::  Pat -> Body -> [LetDec] -> Dec (DC top NoInClass inst lt)
    -- ^ @{ p = b where decs }@
  DataD :: Cxt -> Name -> [TyVarBndr] -> [Con] -> [Name]
        -> Dec (DC top NoInClass NoInInstance NoInLet)
    -- ^ @{ data Cxt x => T x = A x | B (T x)
    --       deriving (Z,W)}@
  NewtypeD :: Cxt -> Name -> [TyVarBndr] -> Con -> [Name]
           -> Dec (DC top NoInClass NoInInstance NoInLet)
    -- ^ @{ newtype Cxt x => T x = A (B x)
    --       deriving (Z,W)}@
  TySynD :: Name -> [TyVarBndr] -> Type -> Dec (DC top NoInClass NoInInstance NoInLet)
    -- ^ @{ type T x = (x,x) }@
  ClassD :: Cxt -> Name -> [TyVarBndr] -> [FunDep] -> [ClassDec]
         -> Dec (DC top NoInClass NoInInstance NoInLet)
    -- ^ @{ class Eq a => Ord a where ds }@
  InstanceD :: Cxt -> Type -> [InstanceDec] -> Dec (DC top NoInClass NoInInstance NoInLet)
    -- ^ @{ instance Show w => Show [w]
                                  --       where ds }@
  SigD :: Name -> Type -> Dec (DC top cls inst lt)
    -- ^ @{ length :: [a] -> Int }@
  ForeignD :: Foreign -> Dec (DC top NoInClass NoInInstance NoInLet)
    -- ^ @{ foreign import ... }
    -- { foreign export ... }@

  InfixD :: Fixity -> Name -> Dec (DC top cls NoInInstance lt)
    -- ^ @{ infix 3 foo }@

  -- | pragmas
  PragmaD :: Pragma loc -> Dec loc
    -- ^ @{ {\-# INLINE [1] foo #-\} }@

  -- | type families (may also appear in [Dec] of 'ClassD' and 'InstanceD')
  FamilyD :: FamFlavour -> Name -> [TyVarBndr] -> Maybe Kind
          -> Dec (DC top cls NoInInstance NoInLet)
    -- ^ @{ type family T a b c :: * }@

  DataInstD :: Cxt -> Name -> [Type] -> [Con] -> [Name]
            -> Dec (DC top NoInClass inst NoInLet)
    -- ^ @{ data instance Cxt x => T [x] = A x
    --                                | B (T x)
    --       deriving (Z,W)}@
  NewtypeInstD :: Cxt -> Name -> [Type] -> Con -> [Name]
               -> Dec (DC top NoInClass inst NoInLet)
    -- ^ @{ newtype instance Cxt x => T [x] = A (B x)
    --       deriving (Z,W)}@
  TySynInstD :: Name -> TySynEqn -> Dec (DC top cls inst NoInLet)
    -- ^ @{ type instance ... }@

  ClosedTypeFamilyD :: Name -> [TyVarBndr] -> Maybe Kind -> [TySynEqn]
                    -> Dec (DC top NoInClass NoInInstance NoInLet)
    -- ^ @{ type family F a b :: * where ... }@

  RoleAnnotD :: Name -> [Role] -> Dec (DC top NoInClass NoInInstance NoInLet)
    -- ^ @{ type role T nominal representational }@
```


Each constructor's return type tells you where the declaration is prohibited.



Then, specialize `Dec` to the four cases that exist:


```wiki
type TopLevelDec = Dec (DC YesInTopLevel NoInClass  NoInInstance  NoInLet)
type ClassDec    = Dec (DC NoInTopLevel  YesInClass NoInInstance  NoInLet)
type InstanceDec = Dec (DC NoInTopLevel  NoInClass  YesInInstance NoInLet)
type LetDec      = Dec (DC NoInTopLevel  NoInClass  NoInInstance  YesInLet)
```


And now, we can use these last 4 types as simple datatypes that have fewer constructors than the full `Dec` type. Types to the rescue!



For example:


```wiki
data Clause = Clause [Pat] Body [LetDec]
                                  -- ^ @f { p1 p2 = body where decs }@
    deriving( Show, Eq, Data, Typeable, Generic )
```


The `[LetDec]` is the `where` stanza of a declaration. Or:


```wiki
data Exp
  =  ...
  | LetE [LetDec] Exp                  -- ^ @{ let x=e1;   y=e2 in e3 }@
  | ...
  deriving( Show, Eq, Data, Typeable, Generic )
```


Here, we include only `LetDec`s in a `let`. You can see above that `ClassDec` and `InstanceDec` are used as appropriate in `Dec`.



Under this scenario, `[d| ... |]` would have the type `Q [TopLevelDec]`, and if you have a top-level splice, it must also be of type `Q [TopLevelDec]`.



`Pragma`, which are similarly sometimes available and sometimes unavailable, depending on context, are similarly indexed:


```wiki
data Pragma :: DecContext -> * where
  InlineP :: Name -> Inline -> RuleMatch -> Phases
          -> Pragma (DC top cls inst lt)
  SpecialiseP :: Name -> Type -> Maybe Inline -> Phases
              -> Pragma (DC top NoInClass inst lt)
  SpecialiseInstP :: Type -> Pragma (DC NoInTopLevel NoInClass inst NoInLet)
  RuleP :: String -> [RuleBndr] -> Exp -> Exp -> Phases
        -> Pragma (DC top NoInClass NoInInstance NoInLet)
  AnnP :: AnnTarget -> Exp -> Pragma (DC top NoInClass NoInInstance NoInLet)
  LineP :: Int -> String -> Pragma (DC top NoInClass NoInInstance NoInLet)
```

## Discussion


### Pros



This seems to work, in preliminary tests. Matching on this GADT does not trigger [\#3927](http://gitlabghc.nibbler/ghc/ghc/issues/3927) (that is, it does not warn about vacuous incomplete patterns), and you can't squeeze, say, a `DataD` into a `LetDec`.


### Cons


- `Data` and `Generic` (see [\#9527](http://gitlabghc.nibbler/ghc/ghc/issues/9527), requesting `Generic` instances) don't play well with GADTs. I can hand-write `Data` instances for `Dec` and `Pragma`, but these implement only `gfoldl` and error on `gunfold`. I have not yet tried to write hand-written `Generic` instances, but I'm not hopeful.

- I frequently use `:info Dec` in GHCi when writing TH code. The output after this transformation is much noisier than it was before.

## Questions


- Does anyone use `gunfold` for TH types?

- Is there a need for `Generic` instances for `Dec` and `Pragma`? Are these possible for GADTs?
