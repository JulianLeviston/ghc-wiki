
This wiki discusses how bringing [
Nix](https://nixos.org/nix/)-style package management facilities to cabal can solve various cabal problems and help in effective mitigation of cabal hell. It also contains the goals and implementation plan for the GSoC project. It is based on a [
blog post by Duncan Coutts](http://www.well-typed.com/blog/2015/01/how-we-might-abolish-cabal-hell-part-2/).


# Problems


## Breaking re-installations



[](http://www.well-typed.com/blog/aux/images/cabal-hell/install-example1.png)



There are situations where Cabal's chosen solution would involve reinstalling an existing version of a package but built with different dependencies.
In this example, after installing app-1.1, app-1.0 and other-0.1 will be broken. The root of the problem is having to delete or mutate package instances when installing new packages. This is due to the limitation of only being able to have one instance of a package version installed at once.


## Type errors when using packages together



[](http://www.well-typed.com/blog/aux/images/cabal-hell/install-example2.png)



The second, orthogonal, problem is that it is possible to install two packages and then load them both in GHCi and find that you get type errors when composing things defined in the two different packages. Effectively you cannot use these two particular installed packages together.
The fundamental problem is that developers expect to be able to use combinations of their installed packages together, but the package tools do not enforce consistency of the developer's environment.


# Goals


- Fix breaking re-installs (Persistent package store)
- Implement garbage collection to free unreachable packages
- Enable sharing of packages between sandbox
- Enforce development environment consistency (Giving error earlier and better)
- Implement package manager tools in cabal-install(cabal upgrade and cabal remove)

# Implementation Plan


## Persistent package store



A [
patch](https://github.com/ghc/ghc/commit/dd3a7245d4d557b9e19bfa53b0fb2733c6fd4f88) has been pushed for ghc-7.11 that allows multiple instance of the same package to be installed. So the remaining work is in cabal tool for never modifying installed packages. I have written a [
patch to make cabal (almost)non-destructive](https://github.com/fugyk/cabal/commit/45ec5edbaada1fd063c67d6109e69efa0e732e6a). This patch makes all the changes to Package database non-destructive if Installed Package ID is different. To make it fully non-mutable, Thomas Tuegel suggested to


- change installed package IDs to be computed by hash of the sdist tarball.
- Enforce that a package is never overwritten by taking out a lock when updating the database.(before building the package)


It will have additional benefit that package will not be built again if same source has already been built earlier, thus saving time.


## Views



Views will the subset of packages of package store whose modules can be imported. Views will be present as various \*.view file in \<Package DB location\>/views like default.view. The view file contains list of installed package IDs. There will exist a default view which contains packages installed by cabal install outside sandbox. If a package name is installed two times, default view will contain the instance of package which is installed at last. Views' packages will also act like GC roots.



To facilitate views, ghc-pkg will need some new commands:


- Create a view / Symlink a view
- Delete a view
- List all views
- Modify a view
- Add a package
- Remove a package


Sandbox will be a view. Cabal needs to set view when using sandbox. It also needs the ability to make a view and also add a package to the view. Packages can be shared between views. View path will be passed to ghc using -package-env. The view file that sandbox creates lies in the same directory and is symlinked from the package database view file for allowing GC. It will have a benefit that when we just delete the sandbox directory without deleting the view, GC can free that sandbox package.



It looks similar to nix development environment but has some differences. nix environments are like everything that is visible. It is kind of like imported packages with dependency closure. nix needs to make directories visible, while here we already have one more layer. Here we only need exposed package and ghc can make complete environment already. Views are just exposed packages of the environment. So, dependency of the package need not be in the view that the package is in. The problem that we are trying to solve with views is consistent way of managing packages and sandboxes, allowing packages to be shared between sandboxes and its packages being used as GC root.



Summary of design details



When installing a package outside sandbox


- Package is added to default view / modified in default view


Making a sandbox


- A view is made in the directory
- The new view is symlinked from the database
- Packages that are installed are added to that view. Sandboxes cannot affect any other things outside sandbox.


Within sandbox


- All the cabal commands pass view name to ghc and ghc will use relevant package

## Consistent developer environment



It will require additional constraint to check that there is no other instance of the same package or its dependencies is in the environment(packages from which we have imported the modules with their dependency closure) when we are importing the module from a package. It also needs to be checked when cabal is configuring the package, that a package do not directly or indirectly depends on two version of same package. If it is violated it needs to give out an error.


## Garbage collection



This will firstly involve determining the root packages and package list. Root packages are the packages which are in some view. Then we find list of all packages in the database. As there will be single database after implementing views, we don't need to call it for every sandbox database. Then we need to do mark-sweep and find which package are not in the reachable package list and select it for garbage collection. Then the selected packages will be deleted from the package store and also unregistered from database with ghc-pkg.


## cabal remove



With everything implemented above, it is just removing a package from default view. If package is unreachable it can be freed from disk by GC. It is guaranteed to not break any package except the package that is removed.


## cabal upgrade



cabal upgrade is just installing every package that is present in default view that has update available.


