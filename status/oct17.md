# GHC Status Report (October 2017)



GHC continues to focus on stability and infrastructure to improve
release reliability and predictability.


## Major changes in GHC 8.4



GHC 8.4 will continue the focus on stability and performance started in 8.2 and
will include a number of internal refactorings in addition to a few smattering
of user-facing changes.


### Libraries, source language, and type system


-  Phase 2 of the [
  Semigroup-Monoid Proposal](https://prime.haskell.org/wiki/Libraries/Proposals/SemigroupMonoid) (Herbert Valerio Riedel)

- Further refinement of `TypeInType` including improved error messages.

### Compiler


-   A new syntax tree representation based on [
  Trees That Grow](http://www.jucs.org/jucs_23_1/trees_that_grow/jucs_23_01_0042_0062_najd.pdf). This will make it easier for external users to add their own annotations to the HsSyn AST.  In future this should allow Shayan Najd to harmonise the GHC and Template Haskell ASTs, and for the ghc-exactprint annotations to move into the GHC parsed AST (Shayan Najd and Alan Zimmerman).

-   Further stabilization of the Backpack module system (Edward Yang)

-   Greatly improved support for cross-compilation (Moritz Angerman)

-   A new build system based on the `shake` library. This is the culmination

>
>
> of nearly two years of effort, replacing GHC's old Make-based build system
> (Andrey Mokhov and Zhen Zhang)
>
>

-   Improved leverage of join-points: By floating out the “exit path” as a join point of a recursive function, we expose more opportunities for inlining to the simplifier. In the case of nested loops, this can make inner loops allocation-free! (see [\#14152](http://gitlabghc.nibbler/ghc/ghc/issues/14152), Joachim Brietner)

-   Serialisation performance of `Typeable` has improved ([\#14254](http://gitlabghc.nibbler/ghc/ghc/issues/14254), David Feuer)

-   Improved SIMD support. While previous GHC versions supported a variety

>
>
> of SIMD primitives, they did not use native SIMD registers for passing
> arguments across function calls. This severely limited the usefulness of the
> primitives. Starting with 8.4, GHC will use native registers for passing
> vector arguments by default.
>
>

-   Support for the BMI and BMI2 instruction set extensions (John Ky)

-   Many, many bug fixes.

### Runtime system


-   Significantly improved Windows support, with improvements in exception handling, linking, crash diagnostics, GHCi responsiveness and memory allocation and protection (Tamar Christina).

## Development updates and acknowledgments



The past six months have seen a great deal of activity in GHC's infrastructure.
This began in the summer on the heels of the 8.2.1 release, with a small group
of developers reflecting on the various shortcomings of GHC's current
Phabricator-based continuous integration scheme. With the help of Davean this
effort grew from a hypothetical reimagining of GHC's continuous integration into
a functional Jenkins configuration. This effort revealed and addressed numerous
inadequacies in GHC's current test infrastructure. As a result of this work we
can now test GHC end-to-end: starting from repository clone, to source
distribution tarball, to binary distribution tarball, to completed testsuite
run. This allows us to find regressions not just in the compiler itself but also
in the sizeable mass of infrastructure dedicated to packaging and deploying it.



While Jenkins served as a good testing ground for ideas on improving GHC's
testing methodology, continued work with it revealed a number of issues:


- It lacked support for testing within `msys2` on Windows 
- It offered little support in ensuring build purity and reproducibly


configuring build environments


- It required a significant investment of effort to setup, followed by an
  on-going administrative overhead.


Manuel Chakravarty of the newly formed GHC Devops Committee noted these problems
and instead proposed that GHC follow the lead of Rust and consider moving to a combination of hosted
CI services, Appveyor and CircleCI. With Manuel's advocacy, the rest
of the committee ultimately agreed. As of the time of writing GHC is in the
final stages of moving towards this new scheme. Not only will this provide us
with more reliable and easier-to-administer CI system, but it will also
enable us to broaden the pool of contributors to our testing infrastructure.



GHC's performance testing infrastructure also saw some attention this summer
thanks to Haskell Summer of Code student Jared Weakly. Previously, GHC's
performance testsuite would build a variety of test programs, measuring a
variety of run-time and compile-time metrics of each. It would then compare each
metric against a supplied acceptance window to identify regressions. While this
simple approach served us well for years, it was far from perfect,


- inherent variability between runs and environment dependence meant that the testsuite would often incorrectly identify commits as regressing
- to reduce the frequency of false-positives, acceptance window sizes grew over the years meaning that only large regressions would be identified


In his Summer of Code project Jared refactored the testsuite to instead simply
record the performance metrics resulting from a testsuite run. This new
visibility into GHC's testsuite history will allow GHC developers to precisely
identify regressing changes, meanwhile freeing maintainers from the need to
periodically bump testsuite windows.



The recent work on CI goes hand-in-hand with recent changes in GHC's release
scheduling. As of GHC 8.4, GHC will be trying to hold to a six-month periodic
release schedule. We hope that this will allow us to get changes into users'
hands more quickly and more predictably.



In the compiler itself, Shayan Najd and Alan Zimmerman have been working hard on
porting the compiler's frontend AST to use the extension mechanism proposed
in Shayan's "Trees That Grow" paper. This is a significant refactoring that will
allow GHC API users to extend the AST for their own purposes, significantly
improving the reusability of the structure. Eventually this will allow us to
split the AST types out of the `ghc` package, allowing tooling authors, Template
Haskell users, and the compiler itself to use the same AST representation.



Joachim Brietner has been working on continuing the join points work started
in GHC 8.2 by Luke Maurer. Join points formalize a long-standing technique performed
by GHC for eliminating thunk allocation. This formalization has allowed GHC to
be better in identifying and preserving join points by directly representing
them in Core. Joachim is carrying on this work by teaching the Core simplifier
to float out the exit paths of a function, enabling more agressive inlining.



Thomas Jakway has also been looking at runtime performance, introducing loop
annotations in the native code generator. These annotations allow the backend to
identify "hot" variables in loops, which the register allocator can use to
inform its allocation decisions. Early indications suggest that this work may
produce significant speedups in some programs. This work will likely be present
in GHC 8.6.



Kavon Farvardin has been working on improving code-generation by the LLVM code
generator. For a long time, GHC has had to work around LLVM's lack
of externally visible labels in its intermediate language. This workaround meant
that many continuation blocks, which the NCG can optimize as
part of a single procedure, must be broken up into multiple LLVM functions, 
severely limiting the optimization opportunities that LLVM sees. Kavon has been
working with LLVM upstream to introduce pseudo-instructions allowing
GHC to directly represent proc-points in LLVM IR.



Kavon has also been looking at improving the default optimization pass
configuration used by GHC's LLVM backend. This should both improve compilation
time as well as runtime performance.



Peter Trommler, James Clarke, and Karel Gardas have been looking after the
PowerPC and SPARC native code generator backends. This work is valuable not only
as it improves GHC's portability story, but also because these architectures
have memory models which reveal latent bugs more readily than amd64.



Moritz Angerman has been hard at work on a number of areas of the compiler, with
a general focus on portability and cross-compilation. Not only has he
single-handedly rewritten much of GHC's ARM and AArch64 linker for ELF and Mach-O, but he is also
adding cross-compilation support to Template Haskell, improving
cross-compilation support in the build system, and written an alternative the LLVM backend.
Thanks Moritz!



As always, if you are interested in contributing to any facet of GHC,
be it the runtime system, type-checker, documentation, simplifier, or anything in
between, please come speak to us either on IRC (`#ghc` on
`irc.freeenode.net`) or `ghc-devs@haskell.org`. Happy Haskelling!


## Further reading


-   GHC website:

>
>
> \<[ https://haskell.org/ghc/](https://haskell.org/ghc/)\>
>
>

-   GHC users guide:

>
>
> \<[
> https://downloads.haskell.org/\~ghc/master/users-guide/](https://downloads.haskell.org/~ghc/master/users-guide/)\>
>
>

-   `ghc-devs` mailing list:

>
>
> \<[
> https://mail.haskell.org/mailman/listinfo/ghc-devs](https://mail.haskell.org/mailman/listinfo/ghc-devs)\>
>
>

