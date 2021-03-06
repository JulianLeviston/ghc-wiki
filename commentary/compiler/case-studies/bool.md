# Case study: Implementation of wired-in Bool data type



This page gives a hopefully comprehensive view of how `Bool` type is wired-in into the compiler. For easier location of functions within the source code I list the line numbers in which they appear. This may however change very quickly. If you find that is the case please update this wiki page.


## Constants for Bool type and data constructors



All data constructors, type constructors and so on have their unique identifier which is needed during the compilation process. For the wired-in types these unique values are defined in the [compiler/prelude/PrelNames.hs](/trac/ghc/browser/ghc/compiler/prelude/PrelNames.hs). In case of `Bool` the relevant definitions look like this:


```wiki
boolTyConKey, falseDataConKey, trueDataConKey :: Unique
boolTyConKey    = mkPreludeTyConUnique    4 -- line 1256
falseDataConKey = mkPreludeDataConUnique  4 -- line 1445
trueDataConKey  = mkPreludeDataConUnique 15 -- line 1451
```

### A side note on generating Unique values



The `mkPreludeTyConUnique` and `mkPreludeDataConUnique` take care of generating a unique `Unique` value. They are defined in [compiler/basicTypes/Unique.hs](/trac/ghc/browser/ghc/compiler/basicTypes/Unique.hs):


```wiki
data Unique = MkUnique FastInt

mkPreludeTyConUnique :: Int -> Unique
mkPreludeTyConUnique i = mkUnique '3' (3*i)

mkPreludeDataConUnique :: Int -> Unique
mkPreludeDataConUnique i = mkUnique '6' (2*i)
```


You will find definition of `mkUnique :: Char -> Int -> Unique` at line 135 in [compiler/basicTypes/Unique.hs](/trac/ghc/browser/ghc/compiler/basicTypes/Unique.hs).


## Defining wired-in information about Bool



All the wired-in information that compiler needs to know about `Bool` is defined in [compiler/prelude/TysWiredIn.hs](/trac/ghc/browser/ghc/compiler/prelude/TysWiredIn.hs). This file exports following functions related to `Bool`:


```wiki
  boolTy, boolTyCon, boolTyCon_RDR, boolTyConName,
  trueDataCon,  trueDataConId,  true_RDR,
  falseDataCon, falseDataConId, false_RDR,
```


They define `Name`s, `RdrName`s, `Type`, `TyCon`, `DataCon`s and `Id`s for `Bool` type and its two data constructors `True` and `False`.


## Defining Names of type and data constructors



Having defined unique constants we can finally define all needed information about type and data constructors. These definitions might be tricky because they are mutually recursive.



Definitions of type and data constructor `Name` look like this (lines 185-188):


```wiki
boolTyConName, falseDataConName, trueDataConName :: Name
boolTyConName     = mkWiredInTyConName   UserSyntax gHC_TYPES (fsLit "Bool")  boolTyConKey    boolTyCon
falseDataConName  = mkWiredInDataConName UserSyntax gHC_TYPES (fsLit "False") falseDataConKey falseDataCon
trueDataConName   = mkWiredInDataConName UserSyntax gHC_TYPES (fsLit "True")  trueDataConKey  trueDataCon
```


`boolTyConKey`, `falseDataConKey` and `trueDataConKey` are `Unique` values defined earlier. `boolTyCon`, `falseDataCon` and `trueDataCon` are yet undefined. Type of syntax is defined in [compiler/basicTypes/Name.hs](/trac/ghc/browser/ghc/compiler/basicTypes/Name.hs), line 129:


```wiki
data BuiltInSyntax = BuiltInSyntax | UserSyntax
```


`BuiltInSyntax` is used for things like (:), \[\] and tuples. All other things are `UserSyntax`. `gHC_TYPES` is a module `GHC.Types` to which these type and data constructors get assigned. It is defined in [compiler/prelude/PrelNames.hs](/trac/ghc/browser/ghc/compiler/prelude/PrelNames.hs):


```wiki
gHC_TYPES = mkPrimModule (fsLit "GHC.Types") -- line 359

mkPrimModule :: FastString -> Module               -- line 435
mkPrimModule m = mkModule primPackageId (mkModuleNameFS m)
```


`FastString` is a string type based on `ByteStrings` and the `fsLit` function converts a standard Haskell `Strings` to `FastString`. See [compiler/utils/FastString.hs](/trac/ghc/browser/ghc/compiler/utils/FastString.hs) for more details.


### A side note on creating wired-in Names



`Name` is a data type used across the compiler to give a unique name to something and identify where that thing originated from (see [NameType](commentary/compiler/name-type) for more details):


```wiki
data Name = Name {
                n_sort :: NameSort,     -- What sort of name it is
                n_occ  :: !OccName,     -- Its occurrence name
                n_uniq :: FastInt,      
                n_loc  :: !SrcSpan      -- Definition site
            }
    deriving Typeable

data NameSort
  = External Module
  | WiredIn Module TyThing BuiltInSyntax
  | Internal
  | System
```


The `mkWiredInTyConName` and `mkWiredInDataConName` are functions that create `Name`s for wired in types and data constructors. They are defined in [compiler/prelude/TysWiredIn.hs](/trac/ghc/browser/ghc/compiler/prelude/TysWiredIn.hs), lines 163-173:


```wiki
mkWiredInTyConName :: BuiltInSyntax -> Module -> FastString -> Unique -> TyCon -> Name
mkWiredInTyConName built_in modu fs unique tycon
  = mkWiredInName modu (mkTcOccFS fs) unique
      (ATyCon tycon)  -- Relevant TyCon
      built_in

mkWiredInDataConName :: BuiltInSyntax -> Module -> FastString -> Unique -> DataCon -> Name
mkWiredInDataConName built_in modu fs unique datacon
  = mkWiredInName modu (mkDataOccFS fs) unique
      (ADataCon datacon)  -- Relevant DataCon
      built_in
```


The `mkWiredInName` is defined in [compiler/basicTypes/Name.hs](/trac/ghc/browser/ghc/compiler/basicTypes/Name.hs) (lines 279-283), and it just assigns values to fields of `Name`:


```wiki
mkWiredInName :: Module -> OccName -> Unique -> TyThing -> BuiltInSyntax -> Name
mkWiredInName mod occ uniq thing built_in
  = Name { n_uniq = getKeyFastInt uniq,
           n_sort = WiredIn mod thing built_in,
           n_occ = occ, n_loc = wiredInSrcSpan}
```

## RdrNames for Bool



Having defined `Name`s for `Bool`, the [RdrName](commentary/compiler/rdr-name-type)s can be defined ([compiler/prelude/TysWiredIn.hs](/trac/ghc/browser/ghc/compiler/prelude/TysWiredIn.hs), lines 221-225):


```wiki
boolTyCon_RDR, false_RDR, true_RDR :: RdrName
boolTyCon_RDR   = nameRdrName boolTyConName
false_RDR = nameRdrName falseDataConName
true_RDR  = nameRdrName trueDataConName
```


`nameRdrName` is defined in `basicTypes.hs` (line 203) and it simply wraps the `Name` into one of `RdrName`'s value constructors:


```wiki
nameRdrName :: Name -> RdrName
nameRdrName name = Exact name
```

## Type and Data constructors for Bool



Having defined the `Name`s we can define type and data constructors for `Bool`. Lines 578--588 contain these definitions:


```wiki
boolTy :: Type
boolTy = mkTyConTy boolTyCon

boolTyCon :: TyCon
boolTyCon = pcTyCon True NonRecursive boolTyConName
                    (Just (CType Nothing (fsLit "HsBool")))
                    [] [falseDataCon, trueDataCon]

falseDataCon, trueDataCon :: DataCon
falseDataCon = pcDataCon falseDataConName [] [] boolTyCon
trueDataCon  = pcDataCon trueDataConName  [] [] boolTyCon
```


Note that `boolTyCon` is on the list of wired in type constructors created by `wiredInTyCons :: [TyCon]` (line 138).


### A side note on functions generating type and data constructors



[compiler/types/TypeRep.hs](/trac/ghc/browser/ghc/compiler/types/TypeRep.hs), lines 281-282:


```wiki
mkTyConTy :: TyCon -> Type
mkTyConTy tycon = TyConApp tycon []
```


[compiler/prelude/TysWiredIn.hs](/trac/ghc/browser/ghc/compiler/prelude/TysWiredIn.hs), 247-257:


```wiki
pcTyCon :: Bool -> RecFlag -> Name -> Maybe CType -> [TyVar] -> [DataCon] -> TyCon
pcTyCon is_enum is_rec name cType tyvars cons
  = tycon
  where
    tycon = mkAlgTyCon name
    (mkArrowKinds (map tyVarKind tyvars) liftedTypeKind)
                tyvars
                cType
                []    -- No stupid theta
    (DataTyCon cons is_enum)
    NoParentTyCon
                is_rec
    False   -- Not in GADT syntax
```


`compiler/prelude/TysWiredIn.hs`, 261-297:


```wiki
pcDataCon :: Name -> [TyVar] -> [Type] -> TyCon -> DataCon
pcDataCon = pcDataConWithFixity False

pcDataConWithFixity :: Bool -> Name -> [TyVar] -> [Type] -> TyCon -> DataCon
pcDataConWithFixity infx n = pcDataConWithFixity' infx n (incrUnique (nameUnique n))

pcDataConWithFixity' :: Bool -> Name -> Unique -> [TyVar] -> [Type] -> TyCon -> DataCon
pcDataConWithFixity' declared_infix dc_name wrk_key tyvars arg_tys tycon
  = data_con
  where
    data_con = mkDataCon dc_name declared_infix
                (map (const HsNoBang) arg_tys)
                []  -- No labelled fields
                tyvars
    []  -- No existential type variables
    []  -- No equality spec
    []  -- No theta
    arg_tys (mkTyConApp tycon (mkTyVarTys tyvars))
    tycon
    []  -- No stupid theta
                (mkDataConWorkId wrk_name data_con)
    NoDataConRep  -- Wired-in types are too simple to need wrappers

    modu     = ASSERT( isExternalName dc_name )
         nameModule dc_name
    wrk_occ  = mkDataConWorkerOcc (nameOccName dc_name)
    wrk_name = mkWiredInName modu wrk_occ wrk_key
           (AnId (dataConWorkId data_con)) UserSyntax
```

## Generating Id for True and False data constructors



Finally, lines 590-592 contain definitions of `Id` for `True` and `False` data constructors:


```wiki
falseDataConId, trueDataConId :: Id
falseDataConId = dataConWorkId falseDataCon
trueDataConId  = dataConWorkId trueDataCon
```


`falseDataConId` and `trueDataConId` just extract `Id` from previously defined data constructors. These definitions are from [compiler/basicTypes/DataCon.hs](/trac/ghc/browser/ghc/compiler/basicTypes/DataCon.hs):


```wiki
data DataCon -- line 253
  = MkData {
  ...
  dcWorkId :: Id -- line 360
  ...
 }

dataConWorkId :: DataCon -> Id -- line 736
dataConWorkId dc = dcWorkId dc
```

## Final remarks



Remember that all the non-primitive wired-in things are also defined in GHC's libraries. `Bool` is defined in ghc-prim library, `GHC.Types` module: 


```wiki
data {-# CTYPE "HsBool" #-} Bool = False | True
```


See [wired-in and known-key things](commentary/compiler/wired-in) for more details


# TODO


- add information about code generation for Bool values
