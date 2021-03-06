# Ordering the Core-to-Core optimisation passes



This page has notes about the ordering of optimisation phases. An overview of the whole Core-to-Core optimisation pipeline can be found [here](commentary/compiler/core2-core-pipeline).



**NOTE:** This is old documentation and may not be very relevant any more!


## This ordering obeys all the constraints except (5)


- full laziness
- simplify with foldr/build
- float-in
- simplify
- strictness
- float-in


\[check FFT2 still gets benefits with this ordering\]


## Constraints


### 1. float-in before strictness



Reason: floating inwards moves definitions inwards to a site at which
the binding might well be strict.


```wiki
Example		let x = ... in
		    y = x+1
		in 
		...
===>
		let y = let x = ... in x+1
		in ...
```


The strictness analyser will do a better job of the latter
than the former.


### 2. Don't simplify between float-in and strictness



...unless you disable float-let-out-of-let, otherwise
the simiplifier's local floating might undo some 
useful floating-in.


```wiki
Example		let f = let y = .. in \x-> x+y
		in ...
===>
		let y = ... 
		    f = \x -> x+y
		in ...
```


This is a bad move, because now y isn't strict.
In the pre-float case, the binding for y is strict.
Mind you, this isn't a very common case, and
it's easy to disable float-let-from-let.


### 3. Want full-laziness before foldr/build



Reason: Give priority to sharing rather than deforestation.


```wiki
Example		\z -> let xs = build g 
		      in foldr k z xs
===>
		let xs = build g
		in \x -> foldr k z xs
```


In the post-full-laziness case, xs is shared between all
applications of the function.   If we did foldr/build
first, we'd have got 


```wiki
		\z -> g k z
```


and now we can't share xs.


### 4.  Want strictness after foldr/build



Reason: foldr/build makes new function definitions which
can benefit from strictness analysis.


```wiki
Example:	sum [1..10]
===> (f/b)
		let g x a | x > 10    = a
		          | otherwise = g (x+1) (a+x)
```


Here we clearly want to get strictness analysis on g.


### 5.  Want full laziness after strictness



Reason: absence may allow something to be floated out
which would not otherwise be.


```wiki
Example		\z -> let x = f (a,z) in ...
===> (absence anal + inline wrapper of f)
		\z -> let x = f.wrk a in ...
===> (full laziness)
		let x= f.wrk a in  \z -> ...
```


TOO BAD.  This doesn't look a common case to me.


### 6. Want float-in after foldr/build



Reason: Desugaring list comprehensions + foldr/build
gives rise to new float-in opportunities.


```wiki
Example		...some list comp...
==> (foldr/build)
		let v = h xs in
		case ... of
		  []     -> v
		  (y:ys) -> ...(t v)...
==> (simplifier)
		let v = h xs in
		case ... of
		  [] -> h xs
		  (y:ys) -> ...(t v)...
```


Now v could usefully be floated into the second branch.


### 7. Want simplify after float-inwards



(Occurred in the prelude, compiling `ITup2.hs`, function `dfun.Ord.(*,*)`)
This is due to the following (that happens with dictionaries):


```wiki
let a1 = case v of (a,b) -> a
in let m1 = \ c -> case c of I# c# -> case c# of 1 -> a1 5
                                                 2 -> 6
in let m2 = \ c -> case c of I# c# -> 
                     case c# +# 1# of cc# -> let cc = I# cc#
                                             in m1 cc 
   in (m1,m2)
```


floating inwards will push the definition of a1 into m1 (supposing
it is only used there):


```wiki
in let m1 = let a1 = case v of (a,b) -> a
            in \ c -> case c of I# c# -> case c# of 1 -> a1 5
                                		    2 -> 6
in let m2 = \ c -> case c of I# c# -> 
                     case c# +# 1# of cc# -> let cc = I# cc#
                                             in m1 cc 
   in (m1,m2)
```


if we do strictness analysis now we will not get a worker-wrapper
for m1, because of the "let a1 ..." (notice that a1 is not strict in
its body).



Not having this worker wrapper might be very bad, because it might
mean that we will have to rebox arguments to m1 if they are
already unboxed, generating extra allocations, as occurs with m2 (cc)
above. 



To solve this problem we have decided to run the simplifier after
float-inwards, so that lets whose body is a HNF are floated out, 
undoing the float-inwards transformation in these cases.
We are then back to the original code, which would have a worker-wrapper
for m1 after strictness analysis and would avoid the extra let in m2.



What we lose in this case are the opportunities for case-floating 
that could be presented if, for example, a1 would indeed be demanded (strict) 
after the floating inwards.



The only way of having the best of both is if we have the worker/wrapper
pass explicitly called, and then we could do with 


- float-in
- strictness analysis
- simplify
- strictness analysis
- worker-wrapper generation


as we would 


- be able to detect the strictness of m1 after the first call to the strictness analyser, and exploit it with the simplifier (in case it was strict).
- after the call to the simplifier (if m1 was not demanded) it would be floated out just like we currently do, before stricness analysis II and worker/wrapperisation.


The reason to not do worker/wrapperisation twice is to avoid 
generating wrappers for wrappers which could happen.


### 8.  If full laziness is ever done after strictness



...remember to switch off
demandedness flags on floated bindings!  This isn't done at the moment.


### 9. Ignore-inline-pragmas flag for final simplification



\[Occurred in the prelude, compiling ITup2.hs, function dfun.Ord.(\*,\*)\]
Sometimes (e.g. in dictionary methods) we generate
worker/wrappers for functions but the wrappers are never
inlined. In dictionaries we often have 


```wiki
dict = let f1 = ...
           f2 = ...
           ...
       in (f1,f2,...)
```


and if we create worker/wrappers for f1,...,fn the wrappers will not
be inlined anywhere, and we will have ended up with extra
closures (one for the worker and one for the wrapper) and extra
function calls, as when we access the dictionary we will be acessing 
the wrapper, which will call the worker.
The simplifier never inlines workers into wrappers, as the wrappers
themselves have INLINE pragmas attached to them (so that they are always
inlined, and we do not know in advance how many times they will be inlined).



To solve this problem, in the last call to the simplifier we will
ignore these inline pragmas and handle the workers and the wrappers
as normal definitions. This will allow a worker to be inlined into
the wrapper if it satisfies all the criteria for inlining (e.g. it is
the only occurrence of the worker etc.).


### 10. Run Float Inwards once more after strictness-simplify



\[Occurred in the prelude, compiling `IInt.hs`, function `const.Int.index.wrk`\]
When workers are generated after strictness analysis (worker/wrapper),
we generate them with "reboxing" lets, that simply reboxes the unboxed
arguments, as it may be the case that the worker will need the 
original boxed value: 


```wiki
f x y = case x of 
          (a,b) -> case y of 
                     (c,d) -> case a == c of
                                True -> (x,x)
                                False -> ((1,1),(2,2))

==> (worker/wrapper)

f_wrapper x y = case x of
                  (a,b) -> case y of 
                             (c,d) -> f_worker a b c d 

f_worker a b c d = let x = (a,b)
                       y = (c,d)
                   in case a == c of
                        True -> (x,x)
                        False -> ((1,1),(2,2))
```


in this case the simplifier will remove the binding for y as it is not
used (we expected this to happen very often, but we do not know how
many "reboxers" are eventually removed and how many are kept), and
will keep the binding for x.  But notice that x is only used in \*one\*
of the branches in the case, but is always being allocated! The
floating inwards pass would push its definition into the True branch.
A similar benefit occurs if it is only used inside a let definition.
These are basically the advantages of floating inwards, but they are
only exposed after the S.A./worker-wrapperisation of the code!  As we
also have reasons to float inwards before S.A. we have to run it
twice.


