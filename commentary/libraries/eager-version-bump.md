# Eager Version Bumping Strategy



Versioning of GHC core/boot libraries adheres to Haskell's [
Package Versioning Policy](http://pvp.haskell.org) whose scope is considered to apply to **released artifacts** (and therefore doesn't prescribe when to *actually* perform version increments during development)



However, in the spirit of continuous integration, GHC releases snapshot artifacts, and therefore it becomes important for early testers/evaluators/package-authors to be presented with accurate PVP-adhering versioning, especially for those who want adapt to upcoming API changes in new major GHC releases early (rather than being hit suddenly by a disruptive version-bump-wave occurring at GHC release time). 



So while the usual scheme is to update a package version in the VCS right before a release (and reviewing at that point whether a patchlevel, minor or major version bump is mandated by the PVP), for GHC bundled core/boot packages, the **eager version bumping** scheme is preferred, which basically means:



The Cabal version gets patchlevel/minor/major version bumped as soon as it becomes evident that a patchlevel/minor/major version increment (relative to the previous released version) is mandated by the PVP



This becomes particularly easy when also maintaining a `changelog` file during development highlighting the changes for releases, as then one easily keeps track of the last released version, as well as becoming aware more easily of minor/major version increment-worthy API changes.


