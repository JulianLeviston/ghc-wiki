# List of tools needed to build GHC



Here are the gory details about which programs and tools you need in order to build GHC.  For instructions tailored to your particular operating system, see [Building/Preparation](building/preparation).



In most cases the `configure` script will tell you if you are missing something.


<table><tr><th>GHC</th>
<td>
GHC is required to build GHC, because GHC itself is
written in Haskell, and uses GHC extensions.  It is possible
to build GHC using just a C compiler, and indeed some
distributions of GHC do just that, but it isn't the best
supported method, and you may encounter difficulties.  Full
instructions are in [Porting GHC](building/porting).

GHC can be built using either an earlier released
version of GHC, or
bootstrapped using a GHC built from exactly the same
sources.  Note that this means you cannot in general build
GHC using an arbitrary development snapshot, or a build from
say last week.  It might work, it might not - we don't
guarantee anything.  To be on the safe side, start your
build using the most recently released stable version of
GHC.

In general, we support building with the previous 2
major releases, e.g.:

- To build 6.8.\* you need GHC \>= 6.4
- To build 6.10.\* you need GHC \>= 6.6
- To build 7.4.\* you need GHC \>= 7.0
- To build 7.6.\* you need GHC \>= 7.2
- To build 7.8.\* you need GHC \>= 7.4
- To build 7.10.\* you need GHC \>= 7.6
- To build 8.0.\* you need GHC \>= 7.8
- To build 8.2.\* you need GHC \>= 7.10

</td></tr></table>


<table><tr><th>Perl</th>
<td>
Perl version 5 at least is required.  GHC has been known to
tickle bugs in Perl, so if you find that Perl crashes when
running GHC try updating (or downgrading) your Perl
installation.  Versions of Perl before 5.6 have been known to have
various bugs tickled by GHC, so the configure script
will look for version 5.6 or later.
  
Perl should be put somewhere so that it can be invoked
by the `#!` script-invoking mechanism.
</td></tr></table>


<table><tr><th>GNU C (`gcc`)</th>
<td>
Most GCC versions should work with the most recent GHC
sources.  Expect trouble if you use a recent GCC with
an older GHC, though (trouble in the form of mis-compiled code,
link errors, and errors from the `ghc-asm`
script).

If your GCC dies with "internal error"" on
some GHC source file, please let us know, so we can report
it and get things improved.  (Exception: on x86
boxes, you may need to fiddle with GHC's
`-monly-N-regs` option; see the User's
Guide).
</td></tr></table>


<table><tr><th>GNU Make</th>
<td>
The GHC build system makes heavy use of features
specific to recent versions of GNU `make`, so
you must have at least GNU make 3.80 installed in
order to build GHC.
</td></tr></table>


<table><tr><th>[ Happy](http://www.haskell.org/happy)</th>
<td>
Happy is a parser generator tool for Haskell, and is
used to generate GHC's parsers.

If you start from a source tarball of GHC, then you don't need Happy, because we supply the
pre-processed versions of the Happy parsers.  If you intend to
modify the compiler and/or build from a git repository, then you
need Happy.

Happy version 1.19.4 or higher is currently required to build GHC.
Grab a copy from
[ Happy's Web Page](http://www.haskell.org/happy/).
</td></tr></table>


<table><tr><th>[ Alex](http://www.haskell.org/alex/)</th>
<td>
Alex is a lexical-analyser generator for Haskell,
which GHC uses to generate its lexer.

Like Happy, you don't need Alex if you're building GHC from a
source tarball, but you do need it if you're modifying GHC and/or
building from a git repository.

Alex version 3.1 or higher is currently required to build GHC.
Alex is
written in Haskell.
Alex distributions are available from 
[ Alex's Web Page](http://www.haskell.org/alex/).
</td></tr></table>


<table><tr><th>[ Haddock](http://www.haskell.org/haddock/)</th>
<td>
Haddock is a documentation generator for Haskell,
used for making the docs for the libraries. If you don't want to build the docs then you don't need haddock.

Haddock is only needed for GHC 6.8.3 and older; GHC 6.10 comes with Haddock.  
For GHC 6.8 and older you need a 0.\* version of haddock; 2.\* versions won't work.
</td></tr></table>


<table><tr><th>`autoconf` and `automake`</th>
<td>
These are needed if you intend to build from the
darcs/git sources, they are *not* needed if you
just intend to build a standard source distribution.

Version 2.52 or later of the autoconf package is required.
NB. version 2.13 will no longer work, as of GHC version
6.1.  Version 1.9 of automake is known to work, use others at
your own risk.

`autoreconf` (from the autoconf package)
recursively builds `configure` scripts from
the corresponding `configure.ac` and
`aclocal.m4` files.  If you modify one of
the latter files, you'll need `autoreconf` to
rebuild the corresponding `configure`.
</td></tr></table>


<table><tr><th>`diff`</th>
<td>
Most installations should have this by default.
</td></tr></table>


<table><tr><th>`lndir`</th>
<td>
Needed if you want to prepare a source distribution with 
`make sdist`, and useful for maintaining a separate
[build tree](building/using).
</td></tr></table>


<table><tr><th>`sed`</th>
<td>
Most Unix installations and MSYS2 on
Windows already come with `sed`, so you're probably OK.
GNU sed version 2.0.4 is no good!  It has a bug
in it that is tickled by the build-configuration.  2.0.5 is
OK. Others are probably OK too (assuming we don't create too
elaborate configure scripts.)
</td></tr></table>


<table><tr><th>Python</th>
<td>
Required for [running the testsuite](building/running-tests).
Version 2.6 or later is needed. Python3 should also work.
Python3 is needed for GHC 8.2.1 or later.
</td></tr></table>


<table><tr><th>Sphinx</th>
<td>
Required for building the [documentation](building/docs).
</td></tr></table>


<table><tr><th>LLVM</th>
<td>
Required for compiling code with the `-fllvm` flag enabled (e.g., if you want to build GHC with the `perf-llvm` build flavour). Version 3.8 or later is required. See also [the LLVM installation page](commentary/compiler/backends/llvm/installing).
</td></tr></table>


<table><tr><th>[ libedit](http://www.thrysoee.dk/editline/)</th>
<td>
If libedit is installed, ghci will be built with a nice interactive line-editing interface.  
Version 2.6.9 and later are known to work; note that the libeditline package listed on some distros is too old.
If a suitable version of libedit cannot be found, ghc/ghci will still build fine, just without the nice line-editing capabilities.
 If your installation does not have libedit by default, you may either download and build it yourself from the link above, or else install your distro's relevant package (sometimes called "libedit-devel" or "libedit-dev").
GHC does not use libedit on Windows; instead, it uses the console's default line editor.
</td></tr></table>


<table><tr><th>[ libffi](http://sourceware.org/libffi/)</th>
<td>
TODO Document this.  I stumbled on this as a new dependency as a user, but I cannot comment further.
</td></tr></table>


## Nofib



See the [running NoFib](building/running-no-fib) page.


