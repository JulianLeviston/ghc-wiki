# GHC API Hooks



This document describes a proposal for a **hooks** interface to customise certain phases of GHC's source transformation facilities.  A hook is GHC API-user-supplied callback that overrides the built-in functionality when installed.


## The Problem



Much of GHC is exposed as a library, making it possible to use the compiler in various new ways, like extracting type information from Haskell code or generating code from one of the intermediate representations. Often, these use cases require some change from the default behaviour at certain points during compilation. For example someone writing a custom code generator would want reuse as much as possible of the GHC pipeline, to get dependency chasing and recompilation avoidance, but replace the part where normally the native code generator would be called.



Unfortunately the GHC API offers no configuration options for this, and it is hard to customize the compiler manually: For example once `load LoadAllTargets` is called, GHC runs its own pipeline, replacing specific functionality would require the user to copy large parts of `GhcMake` and `DriverPipeline`.


## Proposed Solution



One option is to split GHC's stages into lots of small function calls and allow the user to wire these stages together as needed.  Unfortunately, this is very difficult to do and wouldn't necessarily be very flexible since there are a number of expected invariants not encoded in the types.



Instead we propose hooks: Hooks are an extensible way to replace specific parts of the GHC pipeline by user-supplied callbacks. The goal is to add them to strategic points in the library, to cover most use cases without sprinkling hooks carelessly throughout the GHC code. The API should be considered in flux, since it could require some experimentation to find the best hook locations.



An example that uses all currently implemented hooks, along with who uses them can be found here:



[ Hooks demonstration program](https://gist.github.com/luite/6444273)


## Example


```wiki
hooksExample :: [Located String] -> [FilePath] -> IO ()
hooksExample args targetFiles =
    defaultErrorHandler defaultFatalMessager defaultFlushOut $
      runGhc (Just libdir) $ do
        dflags0 <- getSessionDynFlags
        (dflags1, _leftover, _warns) <- parseDynamicFlags dflags0 args
        let dflags2 = dflags1 { hooks = insertHook RunQuasiQuoteHook myRunQuasiQuote (hooks dflags1) }
        setSessionDynFlags dflags2
        setTargets =<< mapM (\file -> guessTarget file Nothing) targetFiles
        successFlag <- sourceErrorHandler (load LoadAllTargets)
        when (failed successFlag) (throw $ ExitFailure 1)

sourceErrorHandler m = handleSourceError (\e -> do
  GHC.printException e
  liftIO $ exitWith (ExitFailure 1)) m

myRunQuasiQuote :: HsQuasiQuote Name -> RnM (HsQuasiQuote Name)
myRunQuasiQuote q@(HsQuasiQuote name span quoted) = do
  liftIO $ putStrLn ("myRunQuasiQuote: running quasiquoter on\n" ++ show quoted)
  return q -- don't change the quote or quoter for the example
```


`myRunQuasiQuote` is called for every quasiquote



[
Demonstration program that uses all hooks](https://gist.github.com/luite/6444273)


## Design



Each hook has a potentially different type from all the other hooks. Additionally, we need to be able to communicate hooks to all the locations where they may be invoked. We can achieve this by storing the hooks in the `DynFlags`.



Unfortunately, it is impossible to store them directly, as a record, since that would lead to huge cyclic imports (the data types used by the hooks would depend on `DynFlags`, but `DynFlags` would depend on the modules defining the types in the hooks record)



Instead we implement `Hooks` as a heterogeneous map. The public API allows one to insert and lookup hooks safely, correctness of the types is guaranteed by the `Hook` type family. Internally, the `TypeRep` of `a` is used as the actual key for `Hook a`:


```wiki
newtype Hooks = Hooks (M.Map TypeRep Any)

type family Hook a :: *

insertHook :: Typeable a => a -> Hook a -> Hooks -> Hooks
insertHook tag hook (Hooks m) =
  Hooks (M.insert (typeOf tag) (unsafeCoerce hook) m)

lookupHook :: Typeable a => a -> Hooks -> Maybe (Hook a)
lookupHook tag (Hooks m) =
  fmap unsafeCoerce (M.lookup (typeOf tag) m)
```

## Installing a hook



To use a hook, just add it to the `DynFlags` for your session, using the Hook's key:


```wiki
dflags1 = dflags0 { hooks = insertHook SomeHook myImplementation (hooks dflags0) }
```


Keep in mind that Hooks is a low level API, it's easy to break things. Also in some cases, inserting one hook may require inserting another. For example if you use `TcForeignsHook` to accept extra types for your foreign imports, you'll need `DsForeignsHook` to desugar them, otherwise GHC will not know what to do with them.


## Making a new hook



Every hook requires a key, a data type that has to implement `Typeable`, since we use its `TypeRep` as a globally unique key for the `Hooks` map. Additionally, a type family instance of `Hook` is required, to map the key to the actual hook type. You might want to export the original unhooked function, or extra types and functions that users of the hook will need.



If you're in a monad with a `HasDynFlags` instance, you can use the `getHooked` function from `DynFlags`:


```wiki
data HscFrontendHook = HscFrontendHook deriving Typeable
type instance Hook HscFrontendHook = ModSummary -> Hsc TcGblEnv

genericHscFrontend :: ModSummary -> Hsc TcGblEnv
genericHscFrontend mod_summary =
  getHooked HscFrontendHook genericHscFrontend' >>= ($ mod_summary)

-- original function
genericHscFrontend' :: ModSummary -> Hsc TcGblEnv
genericHscFrontend' mod_summary
    | ExtCoreFile <- ms_hsc_src mod_summary =
        panic "GHC does not currently support reading External Core files"
    | otherwise = do
         hscFileFrontEnd mod_summary
```


Otherwise, use the hooks field from `DynFlags` directly:


```wiki

data HscCompileOneShotHook =
  HscCompileOneShotHook deriving Typeable
type instance Hook HscCompileOneShotHook =
  HscEnv -> FilePath -> ModSummary -> SourceModified -> IO HscStatus

 hscCompileOneShot :: HscEnv
                   -> FilePath
                   -> ModSummary
                   -> SourceModified
                   -> IO HscStatus
hscCompileOneShot env =
  fromMaybe hscCompileOneShot'
    (lookupHook HscCompileOneShotHook . hooks . hsc_dflags $ env) env

-- original function
hscCompileOneShot' :: HscEnv
                   -> FilePath
                   -> ModSummary
                   -> SourceModified
                   -> IO HscStatus
hscCompileOneShot' hsc_env extCore_filename mod_summary src_changed
   = ...
```

## List all currently available hooks



Todo, add list here, see [ demo program](https://gist.github.com/luite/6444273)


