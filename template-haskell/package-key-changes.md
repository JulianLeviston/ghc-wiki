
In GHC 7.10, we changed the internal representation of names to be based on package keys (`base_XXXXXX`) rather than package IDs (`base-4.7.0.1`).  However, we forgot to update the Template Haskell API to track these changes.  This lead to some bugs in TH code which was synthesizing names by using package name and version directly, of which we've seen two cases (and there are probably more):


>
>
> [
> https://github.com/ekmett/lens/issues/496](https://github.com/ekmett/lens/issues/496)
> [
> https://ghc.haskell.org/trac/ghc/ticket/10279](https://ghc.haskell.org/trac/ghc/ticket/10279)
>
>


The primary use-case for generating a `Name` from scratch is in order to provide some functions intended for use in Template Haskell which need to refer to a renamed identifier, e.g. `GHC.Num.(+)`, but WITHOUT using a TH splice to get the name `[| (+) |]`



We now propose the following changes to the TH API in order to track these changes.  See these tickets for background and motivation


- [\#9265](http://gitlabghc.nibbler/ghc/ghc/issues/9265): why move to package keys instead of package names
- [\#10279](http://gitlabghc.nibbler/ghc/ghc/issues/10279): breakage caused by faking package keys
- [ Similar lens bug](https://github.com/ekmett/lens/issues/496)

## Rename PkgName to UnitId



Currently, a TH `NameG` contains a `PkgName`, defined as:


```wiki
newtype PkgName = PkgName String
```


This is badly misleading, even in the old world order, since these needed version numbers as well (except for wired-in packages, which are always just a bare package name). We propose that this be renamed to `UnitId`:


```wiki
newtype UnitId = UnitId String
mkUnitId :: String -> UnitId
mkUnitId = UnitId
```


**Variation 1.** Keep the old `PkgName` type and constructor around for backwards compatibility.  One reason **not** to do this is that the change from package name to package key was a semantic change, meaning that people who were faking keys by concatenating a name with a version had their code break. (On the other hand, by the end of the 7.10 cycle most people probably will be using the old name correctly.)



Here is the GitHub search for uses of `NameG`, which usually indicates a package key is being manipulated directly: [
https://github.com/search?l=haskell&q=NameG&type=Code&utf8=%E2%9C%93](https://github.com/search?l=haskell&q=NameG&type=Code&utf8=%E2%9C%93)


## Querying about units



Unit IDs are somewhat hard to synthesize, so we also offer an API for querying the package database of the GHC which is compiling your code for information about packages.  So, we introduce a new abstract data type:


```wiki
data Unit
unitId :: Unit -> UnitId
unitVersionString :: Unit -> String
unitPackageName   :: Unit -> String

unitDependencies :: Unit -> Q [Unit]
```


and some functions for getting packages:


```wiki
searchUnit :: String -- Package name
           -> String -- Version
           -> Q [Unit]

reifyUnit :: UnitId -> Q Unit
```


We could add other functions (e.g., return all packages with a package name).



**Flex point 1.** Should hidden packages be hidden from this query, or reported but with their `exposed` field set to `False`?


## Retrieving the current package key



Commonly, a user wants to get the package key of the current package.  Following Simon's suggestion, this will be done by augmenting `ModuleInfo`:


```wiki
 data ModuleInfo =
     ModuleInfo { mi_this_mod :: Module -- new
                , mi_imports :: [Module] }
```


We'll also add a function for accessing the module package key:


```wiki
moduleUnitId :: Module -> UnitId
```


And a convenience function for accessing the current module:


```wiki
 thisUnitId :: Q UnitId
 thisUnitId = fmap (moduleUnitId . mi_this_mod) qReifyModule

 thisPackage :: Q Unit
 thisPackage = reifyUnit =<< thisUnitId
```


**Variation 1.** The current package and module is accessible from `qLocation`. Should the types in the record that `qLocation` returns change?


