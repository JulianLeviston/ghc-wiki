


---



**NOTE** This page was written in 2008 when we were considering switching from darcs to a different version control system.  Much of what it says may be incorrect or out of date.  The current status is that we have no immediate plans to switch from darcs, although we remain open to the possibility of switching in the future. In an informal show of hands at a meeting at ICFP'10 showed that there was roughly a 50/50 split for and against a switch.



Darcs has made important improvements over the last couple of years addressing some, but not all, of our concerns, so at least from my (Simon M.) point of view, switching is not as urgent as it was.



Switching is also difficult for GHC, because we use several repositories (GHC, packages, haddock, hsc2hs, etc.), some of which are upstream and hence we cannot unilaterally switch them from darcs to something else, but nevertheless we would like a GHC developer to only have to use and learn one VC tool.  We would probably have to mirror these respositories in git or whatever, which would complicate our workflow.


---


# Darcs retrospective, and the future



GHC has been using darcs for version control since the beginning of 2006.  It has not been all plain sailing, so in this page we will record our experiences with darcs, and attempt to objectively evaluate whether we would be better off with a different version control system.  In the event that we do switch, we need to track exactly what needs to change, so this page will also list those dependencies.


## Problems we currently experience with darcs


- Conflicts and merging.  This is the biggest problem we encounter, and is also the \#1 priority for
  Darcs development.  Any non-trivial branch is affected, and essentially the workaround is to discard
  the history from the branch when merging, and use ordinary diff/patch tools.  Keeping history is
  possible, but impractical for branches with more than a few patches.

- Speed.  many operations are impractical (annotate, `darcs changes <file>`), and many operations just take "too
  long" (i.e. long enough that you go and do something else rather than wait for it to finish,
  which incurs a context-switch cost).  We can't use Trac's darcs integration or darcsweb, for example,
  because both rely on invoking `darcs changes <file>` (for that matter, that's not completely true for the
  [
  trac darcs plugin](http://progetti.arstecnica.it/trac+darcs) as it does not execute that command
  on a per-file basis, but rather it loads and caches into its own database the result of `darcs changes -v` 
  on the "not-yet-loaded" changesets, visiting every patch in the repository just once. 
  It caches also the actual content of each file touched by any browsed changeset, to compute the unidiff.).

- bugs: we run into darcs bugs other than the conflict/merging bug on a regular basis.

- user interface issues: e.g. in a conflict there's no way to tell
  which two patches are conflicting with each other(!)

- Windows support: is quite flaky still.  (well, it's certainly better than it used to be, and
  at least some Windows users don't consider it to be bad).

## Current status



On the 23rd July 2008 an IRC meeting on the \#ghc channel decided to make a serious effort to replace Darcs, due to all the problems described above. The logs of that meeting are available in full at [
http://hackage.haskell.org/trac/ghc/attachment/wiki/IRC\_Meetings/ghc-metting-2008-07-23.log](http://hackage.haskell.org/trac/ghc/attachment/wiki/IRC_Meetings/ghc-metting-2008-07-23.log), but the main conclusions were:


- The GHC developers have sufficient problems with Darcs that a change would be beneficial


 


- We want to stick with distributed version control, and have a widely-used and well-supported system, so Mercurial and Git and bzr are the only real
  contenders

- Mercurial and Git and percived as being mostly feature-and-performance comparable, although git is more popular

- More investigation of the Mercurial option for GHC is needed, especially in light of reported poor support for Windows with Git. This
  work is ongoing


See also \[[DarcsConversion](darcs-conversion)\].


## Important workflows


### Cherry-picking patches



This is how we maintain the stable GHC branch. Particular fixes are pulled from the HEAD. When the desired patches don't depend on undesired patches, darcs takes care of this automatically, as demonstrated below. Otherwise, with darcs, the patch has to be merged by hand.


```wiki
# Make a repo with a single file with lines 1,3,5,7 in

mkdir repo1
cd repo1
darcs init
printf 'Line1\nLine3\nLine5\nLine7\n' > file
darcs record --all --look-for-adds -m patch1 -A igloo@earth.li
cd ..

# Check out 2 copies of the repo

darcs get repo1 repo2
darcs get repo1 repo3

# Add a patch that adds lines 2 and 6, then another that adds line 4

cd repo1
printf 'Line1\nLine2\nLine3\nLine5\nLine6\nLine7\n' > file
darcs record --all -m patch2
printf 'Line1\nLine2\nLine3\nLine4\nLine5\nLine6\nLine7\n' > file
darcs record --all -m patch3

# Pull the line 4 patch, but not the lines 2 and 6 patch, into the
# other repos non-interactively and interactively

cd ../repo2
darcs pull --all --patches patch3
cd ../repo3
darcs pull 
n
y

# repo2's and repo3's file now contains lines 1,3,4,5,7
```


Git:


```wiki
mkdir testrepo
cd testrepo
git init
printf 'Line1\nLine3\nLine5\nLine7\n' > file
git add *
git commit -a -m patch1

# create the branch
git checkout -b branch1  # also switches to the branch
git checkout master      # switch back to master

printf 'Line1\nLine2\nLine3\nLine5\nLine6\nLine7\n' > file
git commit -a -m patch2
printf 'Line1\nLine2\nLine3\nLine4\nLine5\nLine6\nLine7\n' > file
git commit -a -m patch3

git log --grep=patch3   # prints the commit id for that change
git checkout branch1
git cherry-pick <commit-id from above>

# we can also cherry-pick using the GUI
git checkout master
git checkout -b branch2 master^^  # start a new branch from tree after patch1 
# we're in branch2 now
gitk master  # start the gui for master branch (which containts patch2 & patch3)
# in the gui you can rightclick on the desired patch and choose 
# "cherry-pick" this commit.  et voila, it's in branch2
```


Mercurial:


```wiki
# Make a repo with a single file with lines 1,3,5,7 in

mkdir repo1
cd repo1
hg init
printf 'Line1\nLine3\nLine5\nLine7\n' > file
hg commit --addremove -m patch1 -u igloo@earth.li
cd ..

# Check out 2 copies of the repo

hg clone repo1 repo2
hg clone repo1 repo3

# Add a patch that adds lines 2 and 6, then another that adds line 4

cd repo1
printf 'Line1\nLine2\nLine3\nLine5\nLine6\nLine7\n' > file
hg commit -m patch2
printf 'Line1\nLine2\nLine3\nLine4\nLine5\nLine6\nLine7\n' > file
hg commit -m patch3

# Pull the line 4 patch, but not the lines 2 and 6 patch, into the
# other repos non-interactively and interactively

cd ../repo2
(cd ../repo1; hg log | grep patch3 -4) # note the changeset number of patch3, should be 2
hg transplant --source ../repo1 2
cd ../repo3
hg transplant --source ../repo1
# It doesn't appear to be possible to omit the source argument for transplant,
# though you can omit it from the non-cherrypicking hg pull
n <press enter>
y <press enter>

# repo2's and repo3's file now contains lines 1,3,4,5,7
```


Bzr:


```wiki
# Make a repo with a single file with lines 1,3,5,7 in

mkdir repo1
cd repo1
bzr init
printf 'Line1\nLine3\nLine5\nLine7\n' > file
bzr add file
bzr commit -m patch1 --author igloo@earth.li
cd ..

# Check out 2 copies of the repo

bzr clone repo1 repo2
bzr clone repo1 repo3

# Add a patch that adds lines 2 and 6, then another that adds line 4

cd repo1
printf 'Line1\nLine2\nLine3\nLine5\nLine6\nLine7\n' > file
bzr commit file -m patch2
printf 'Line1\nLine2\nLine3\nLine4\nLine5\nLine6\nLine7\n' > file
bzr commit file -m patch3

# Pull the line 4 patch, but not the lines 2 and 6 patch, into the
# other repos non-interactively and interactively

cd ../repo2
(cd ../repo1; bzr log | grep -5 patch2) # Note revision number 2
bzr merge -c 2 ../repo1
cd ../repo3
# We can't do this yet because bzr does not support interactive cherrypicking for merge:
#darcs pull 
#n
#y
# However, these is a plugin that aims to add it, though it's not very active:
# https://launchpad.net/bzr-interactive
# That plugin does however support interactive record

# repo2's and repo3's file now contains lines 1,3,4,5,7
```

### Cherry-picking during record



In this example, we want to record just the fixes we have made, and not the debugging prints. We want something similar for:


- reverting just the debugging prints, and not the fixes
- reverting the parts of a change you are working on that turned out not to be right
- in the middle of doing a large change, discovering a little bug and wanting to fix and record it

```wiki
# Make a repo with a single file with lines 1,3,5,7,9 in

mkdir repo1
cd repo1
darcs init
printf 'Line1\nLine3\nLine5\nLine7\nLine9\n' > file
darcs record --all --look-for-adds -m patch1 -A igloo@earth.li

# Now we fix a bug, and in the process add some debugging prints.

printf 'Line1\nFix2\nLine3\nDebug4\nLine5\nFix6\nLine7\nDebug8\nLine9\n' > file

# We want to record our fix, but not the debugging prints.

darcs rec -m the_fix
y
n
y
n

# Get rid of the debug prints

darcs revert -a

# Now file contains lines 1,2,3,5,6,7,9
```


Git:


```wiki
mkdir repo1
cd repo1
git init






printf 'Line1\nLine3\nLine5\nLine7\nLine9\n' > file
git add *
git commit -m patch1

printf 'Line1\nFix2\nLine3\nDebug4\nLine5\nFix6\nLine7\nDebug8\nLine9\n' > file

git add --patch
s  # split the diffs
y
n
y
n
git commit -m the_fix

git reset --hard   # delete all changes in working dir
```


Mercurial:


```wiki
# Make a repo with a single file with lines 1,3,5,7,9 in

mkdir repo1
cd repo1
hg init
printf 'Line1\nLine3\nLine5\nLine7\nLine9\n' > file
hg commit --addremove -m patch1 -u igloo@earth.li

# Now we fix a bug, and in the process add some debugging prints.

printf 'Line1\nFix2\nLine3\nDebug4\nLine5\nFix6\nLine7\nDebug8\nLine9\n' > file

# We want to record our fix, but not the debugging prints.

hg record -m the_fix
Y <press enter>
y <press enter>
n <press enter>
y <press enter>
n <press enter>

# Get rid of the debug prints

hg revert -a

# Now file contains lines 1,2,3,5,6,7,9
```


Bzr:


```wiki
# Make a repo with a single file with lines 1,3,5,7,9 in

mkdir repo1
cd repo1
bzr init
printf 'Line1\nLine3\nLine5\nLine7\nLine9\n' > file
bzr add file
bzr commit -m patch1 --author igloo@earth.li

# Now we fix a bug, and in the process add some debugging prints.

printf 'Line1\nFix2\nLine3\nDebug4\nLine5\nFix6\nLine7\nDebug8\nLine9\n' > file

# We want to record our fix, but not the debugging prints.

# Using the new interactive plugin: https://launchpad.net/bzr-interactive
#bzr commit -m the_fix -i
# .. and it won't let us cherry pick each line. It's all or nothing!
# bzr shelve is a better supported plugin that allows something similar to this
# (shelve the debug prints, commit what remains, unshelve the debug prints
# and revert them)
# but it also doesn't let us pick between the changes in this hunk. They are
# welded into one hunk because they are close together.

# Get rid of the debug prints

bzr revert

# Now file contains lines 1,2,3,5,6,7,9
```

### amend-record



So, you make your lovely patch, it all looks good, so you record it. Then you do a build to make sure it works, and during the build or testsuite run you find that the patch wasn't quite right after all. You could just add a little 2-line patch, but that isn't very pleasant: It's nice if, as far as possible, all intermediate compiler states are buildable. Also, people might pull the first patch but not the second when cherry-picking, leading to head-scratching down the line. It's much nicer to be able to just amend-record the fix into your original patch.



The same is available for Git.  The command is called `git commit --amend`.  You usually checkout the commit you want to edit into a branch, do the changes, then rebase the remaining patches on top of this.  Example coming soon...



I can't find a way to do this directly with Mercurial. You can of course do `hg rollback` and then add a new commit. The Mercurial Queues extension is also able to do this (`hg qrefresh`) but it is rather complicated to use.



It doesn't appear to be possible with Bzr either. You have to do `bzr uncommit` and then `bzr commit`, similarly to Mercurial.


### File renames



We often want to cherry-pick a change where the file has been renamed on one branch or the other.  This should work without any extra intervention from the user, and does under darcs.



Git doesn't handle file renames well.  Here's a script to demonstrate the problem:


```wiki
# Demonstrates problem with git's cherry picking not commuting around
# file renmaes.

rm -rf repo1 repo2

mkdir repo1
cd repo1
git init
printf "b\nd\n" >file
git-add file
git-status
git-commit -m "bd"

cd ..
git clone repo1 repo2
 
cd repo1
git mv file file1
git commit -m move
printf "a\nb\nd\ne\n" >file1
git commit -m "abde" file1 
printf "a\nb\nc\nd\ne\n" >file1
git commit -m "abcde" file1
 
cd ../repo2
git remote add -f repo1 ../repo1
git cherry-pick repo1/master
# cherry-picks the most recent change from repo1
# BANG!!!
```


Apparently git didn't realise that "file" had been renamed to "file1" in one branch, because its contents had also changed sufficiently.  In fact, if you add enough other stuff to the file so that both versions are similar, then the merge works, which is deeply worrying.



This goes wrong with git version 1.5.2.5.  I wouldn't be surprised if other versions work, but the underlying issue is that git doesn't store information about file and directory renames, and has to rely on heuristics to recover the information when necessary.  Converting a darcs repo into a git repo is a lossy conversion - it discards information about renames.



Hg doesn't seem to deal with it well either:


```wiki
rm -rf repo1 repo2

mkdir repo1
cd repo1
hg init
printf "b\nd\n" >file
hg add file
hg status
hg commit -m "bd"

cd ..
hg clone repo1 repo2
 
cd repo1
hg rename file file1
hg commit -m move
printf "a\nb\nd\ne\n" >file1
hg commit -m "abde" file1 
printf "a\nb\nc\nd\ne\n" >file1
hg commit -m "abcde" file1
 
cd ../repo2
hg transplant --source ../repo1 tip
# transplant the most recent change from repo1
# BANG!!!
#
# searching for changes
# applying b613e5e3dc1a
# unable to find 'file1' for patching
# 1 out of 1 hunk FAILED -- saving rejects to file file1.rej
# file1: No such file or directory
# patch failed to apply
# abort: Fix up the merge and run hg transplant --continue
```


bzr manages this example without any difficulty:


```wiki
#!/bin/sh

rm -rf repo1 repo2

mkdir repo1
cd repo1
bzr init
printf "b\nd\n" >file
bzr add file
bzr status
bzr commit -m "bd"

cd ..
bzr checkout repo1 repo2
  
cd repo1
bzr mv file file1
bzr commit -m move
printf "a\nb\nd\ne\n" >file1
bzr commit -m "abde" file1 
printf "a\nb\nc\nd\ne\n" >file1
bzr commit -m "abcde" file1
  
cd ../repo2
bzr merge -c 4 ../repo1
# cherry-picks revision 4 from repo1
bzr diff
```

### Popularity



The Debian popularity contest has some interesting numbers: [
http://people.debian.org/\~igloo/popcon-graphs/index.php?packages=bzr%2Cgit-core%2Cmercurial&show\_vote=on&want\_legend=on&from\_date=&to\_date=&hlght\_date=&date\_fmt=%25Y-%25m&beenhere=1](http://people.debian.org/~igloo/popcon-graphs/index.php?packages=bzr%2Cgit-core%2Cmercurial&show_vote=on&want_legend=on&from_date=&to_date=&hlght_date=&date_fmt=%25Y-%25m&beenhere=1). Git is the clear winner in popularity terms.


## Darcs alternatives still in the running


### Mercurial



\#mercurial: 118 members



Sample repo available at [
http://darcs.haskell.org/ghc.hg](http://darcs.haskell.org/ghc.hg)



Advantages:


- Speed comparable to Git
- Some operations become feasible (bisect, annotate)
- Many helper tools
- Good Windows support
- HTTP and SSH sync possible, but unknown how this compares to Git native protocol sync speed


Disadvantages:


- Similar problems with bisect support as Git
- (Unknown: suitability of command set?)
- No rebase, though this is being added as part of the Summer of Code
- Need to run a special server to be able to clone over HTTP (free hosting service is available, will provide free plans for open source projects, supports HTTPS and SSH, [
  http://www.bitbucket.org/](http://www.bitbucket.org/))
- Some suggestion that you may get performance problems on OS X due to the pervasive use of hardlinks, which HFS+ supports poorly (see [
  http://weblogs.mozillazine.org/jst/archives/2007/02/more\_on\_distributed\_vcs\_perfor.html](http://weblogs.mozillazine.org/jst/archives/2007/02/more_on_distributed_vcs_perfor.html))
- Debian stable (which e.g. darcs.haskell.org runs) only has v0.9.1, which does not include the new repository format that makes case-sensitivity fixes

#### Darcs vs Mercurial Overview



Commands somehow different in behaviour between Hg and Darcs:


```wiki
darcs whatsnew -> hg diff / hg status
darcs record -> hg commit / hg record (record extension needed to allow cherrypicking)
darcs pull -> hg pull -u / hg pull && hg update (hg pull does not modify the working copy by default)
(automatic merging) -> hg merge && hg commit / hg fetch (fetch extension does pull/update/merge in one step, like Darcs)
darcs unrecord -> hg rollback (works for just the most recent record/push, confusingly different from Darcs equivalent command)
```


Commands that differ essentialy only in name:


```wiki
darcs rollback -> hg backout --merge (records an inverse changeset, go back as far as you like)
darcs changes -> hg log
darcs move <FILE> -> hg rename <FILE>
darcs send -o <FILE> -> hg bundle <FILE>
darcs apply <FILE> -> hg unbundle <FILE>
```


Commands the same between Hg and Darcs:


```wiki
darcs push -> hg push
darcs add/remove <FILE> -> hg add/remove <FILE>

darcs revert -> hg revert
darcs tag -> hg tag
darcs annotate -> hg annotate
```


`hg addremove` adds untracked files and marks missing files as removed. hg commit -A does a similar thing at commit time



Misc. differences from Darcs:


- Don't have summary/message split: the first line of the message is the summary

- Files are not automatically considered removed if you delete them. You need to run hg remove --after \<FILE\> to remove them from the repo as well


To be able to use all the commands in the example above, you should create a .hgrc file in your home directory, looking something like this:


```wiki
[extensions]
hgext.record=
transplant=

[ui]
username = My Name <foo@bar.com>
```


Other issues:


- (Note: not nearly as bad as I first thought, this only applies to the use of an extension called win32text: There appears to be poor support for Windows with the transplant command [
  http://www.selenic.com/mercurial/bts/issue1077](http://www.selenic.com/mercurial/bts/issue1077))


 


- The transplant command wiki page [
  http://www.selenic.com/mercurial/wiki/index.cgi/TransplantExtension](http://www.selenic.com/mercurial/wiki/index.cgi/TransplantExtension) contains the text "Three-way merge doesn't cope particularly well with transplanted patches - it will tend to generate false conflicts",
  which doesn't fill me with confidence. However, we only want to use transplant to maintain a branch (e.g. 6.8) which we won't merge back into the one we are pulling from (e.g. HEAD), so this may be a non-issue


Setting up a Mercurial HTTP interface: [
http://hgbook.red-bean.com/hgbookch6.html\#x10-1310006.6](http://hgbook.red-bean.com/hgbookch6.html#x10-1310006.6)


#### Notes On Conversion



Currently using Tailor. Problems encountered:
 


- Darcs outputs XML without an encoding header. Patched Tailor to append Latin-1 encoding to the XML output. This will be sent to the tailor author
- In hg.py, replace the line `self._hgCommand('tag', tag)` with `self._hgCommand('tag', tag, force=True)` because we seem to be trying to apply duplicate tags at some points. Don't know quite how this is possible!
- MUST USE the seperate-subdir mode of Tailor because we have some tricky Darcs patches. I've added my scripts to [
  http://www.selenic.com/mercurial/wiki/index.cgi/Tailor\#preview](http://www.selenic.com/mercurial/wiki/index.cgi/Tailor#preview)
- No support for author remapping in Tailor yet. I've added it and I'm going to submit the patch to the Tailor author

### Git



\#git: 388 members



Sample repo available at [
http://darcs.haskell.org/ghc.git](http://darcs.haskell.org/ghc.git)



Advantages:
  


- Speed
- Very similar workflow possible: `git add --patch`, `git cherry-pick`, and others
- Some operations become feasible (bisect, annotate)
- Many helper tools


Disadvantages:


- Complex command set?  (Though, it should be possible to find replacements for the darcs commands and be happy.) 
- Lack of good Windows support?
- file and directory renames are not tracked accurately.  Merging uses heuristics to discover file/directory renames, which sometimes goes wrong.
- bisect support would require git modules to also pick the correct version of libraries.  Keeping this in sync is not easy, atm.
- uses its own protocol for network transmission (http works but is slower, however, other hosting services are available, e.g., github)

#### Darcs / Git Command Comparison


<table><tr><th>`darcs whatsnew -s`</th>
<th>`git status`
</th></tr>
<tr><th>`darcs whatsnew`   </th>
<th>`git diff`
</th></tr>
<tr><th>`darcs record`     </th>
<th>`git add --patch` (goes through all changes)/`git add -i` (starts with a file-based view) Git add only marks changes for commit.  This can be nicer if you want to check some things first before you commit them.
</th></tr>
<tr><th> </th>
<th>`git commit` (do the actual commit)
</th></tr>
<tr><th>`darcs record -a -m foo`</th>
<th>`git commit -a -m foo`
</th></tr>
<tr><th>`darcs pull`</th>
<th>`git pull` then `git cherry-pick`/`gitk` + select patches using mouse.  It's probably best to have one local branch correspond to the remote branch and then cherry-pick from that.  You can also create local names for several remote repositories.
</th></tr></table>


### Bzr



\#bzr: 143 members



Sample repo available at [
http://darcs.haskell.org/ghc.bzr](http://darcs.haskell.org/ghc.bzr)



Advantages:


- Fairly fast (but not as fast as Git or Mercurial - see Benchmark below)
- Allows checkout without any history
- Portable (as portable as python, anyhow), works on Windows
- Merging works correctly based on closest-common-ancestor
- Tracking of renamed files / directories merges correctly (better even than Mercurial)
- Revisions form a DAG (more like a tree with merge-points) rather than patchsets
- Supports convenient "centralised-style" commit-remote-by-default as well as "distributed-style" commit-local-by-default. Just 'bind' or 'unbind' your branch whenever you want.
- Simple clear UI
- Has rebase


Disadvantages


- Revisions form a DAG (more like a tree with merge-points) rather than patchsets (this is a subjective point, which is why it's in both lists. Which model do you believe in?)
- Cherry-picking isn't very "native" to the data model. Support for this is very poor.
- UI is rather different from darcs (which current contributors are used to).

### Benchmarks



These benchmark figures were obtained with a warm disk cache on a clean tree, using OS X 10.5:


<table><tr><th>     </th>
<th>Annotate</th>
<th>Log   </th>
<th>Status</th>
<th>Clone   
</th></tr>
<tr><th>Git  </th>
<th>0.936s  </th>
<th>0.523s</th>
<th>0.030s</th>
<th>0.580s  
</th></tr>
<tr><th>Hg   </th>
<th>0.230s  </th>
<th>3.772s</th>
<th>0.178s</th>
<th>9.455s  
</th></tr>
<tr><th>Bzr  </th>
<th>2.131s  </th>
<th>7.278s</th>
<th>0.312s</th>
<th>49.788s 
</th></tr>
<tr><th>Darcs</th>
<th>47.080s\*</th>
<th>2.115s</th>
<th>0.053s</th>
<th>28.276**
**</th></tr></table>



Footnotes:


- `darcs annotate` fails with `Stack space overflow: current size 8388608 bytes.`, so you don't get an answer


** `darcs get` fails with `Unapplicable patch` due to the case-insensitivity of HFS+, so you don't get a clone
**



These figures were obtained with a warm disk cache on a clean tree, using a Windows XP Parallels virtual machine running under OS X 10.5:


<table><tr><th>     </th>
<th>Annotate</th>
<th>Log   </th>
<th>Status</th>
<th>Clone   
</th></tr>
<tr><th>Git  </th>
<th>1.191s  </th>
<th>0.681s</th>
<th>0.180s</th>
<th>3.035s  
</th></tr>
<tr><th>Hg   </th>
<th>0.420s  </th>
<th>2.844s</th>
<th>0.221s</th>
<th>16.383s 
</th></tr>
<tr><th>Bzr  </th>
<th>2.954s  </th>
<th>7.731s</th>
<th>0.350s</th>
<th>62.190s 
</th></tr></table>



The Bzr clone time is high because it does an actual copy rather than just using hard links, by design (you can use the shared repository feature instead). However, even on the other commands it seems to be about twice as slow as Hg, which is on average somewhat slower than Git.



Note that this is a very limited benchmark: it doesn't even test merging / pulling or the cost of cloning over a network.


<table><tr><th>    </th>
<th>HTTP Clone (darcs.haskell.org)
</th></tr>
<tr><th>Git </th>
<th>21m 23.612s                   
</th></tr>
<tr><th>Hg  </th>
<th>231m 11.580s                  
</th></tr>
<tr><th>Bzr </th>
<th>77m 11.526s                   
</th></tr></table>



All of these clone times can be improved by deploying server-side software of some sort or another. But it's nice to just be able to set up a repo by exposing a directory, and these are the sort of pull times you can expect from such a deployment.


## Eliminated alternatives


### Darcs



\#darcs: 39 members



Advantages to staying with darcs:
  


- Community consistency: essentially the Haskell community has standardised on darcs, so it would be 
  an extra barrier for contributors if they had to learn another VC system.

- Merging, when it works, is done right in darcs.


Disadvantages to staying with darcs:


- Uncertain future: no critical mass of hackers/maintainers.  The technical basis is not well enough
  understood by enough people.


Reason for elimination: persistent performance and algorithmic problems, see above.


## External references



Posts/blogs:


- [
  Hans Fugal: Mercurial and Darcs](http://hans.fugal.net/blog/articles/2007/11/16/mercurial-and-darcs)
- [
  Hans Fugal: Darcs and Mercurial Redux](http://hans.fugal.net/blog/articles/2007/11/20/darcs-and-mercurial-redux)
- [
  iBanjo: The Risks of Distributed Version Control](http://blog.red-bean.com/sussman/?p=20)
- [
  https://lists.ubuntu.com/archives/bazaar/2007q4/033256.html](https://lists.ubuntu.com/archives/bazaar/2007q4/033256.html)
- [ http://bazaar-vcs.org/BzrVsGit](http://bazaar-vcs.org/BzrVsGit)
- [ cgit = super-fast](http://community.livejournal.com/evan_tech/236528.html)
- [
  How I stopped missing Darcs and started loving Git](http://blog.moertel.com/articles/2007/12/10/how-i-stopped-missing-darcs-and-started-loving-git)
- [
  Thomas Schilling converts the GHC tree to Git](http://nominolo.blogspot.com/2008/05/thing-that-should-not-be-or-how-to.html)
