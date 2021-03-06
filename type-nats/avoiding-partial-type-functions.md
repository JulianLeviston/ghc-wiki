
It may be tempting to add type functions for subtraction, division, logarithms, and square roots.
These are the inverses of the functions that we already have.  The problem with having these operations is that they are not defined for all their inputs.  I can see the following options, if we were to introduce such functions:


1. Accept some ill defined "types" like (1 - 2), or (5 / 2).  They would belong to kind Nat, but are not really natural numbers.
1. Come up with a "completion" for the type function (e.g., by defining (1 - 2) \~ 0), and having division do some rounding)
1. Automatically generate additional constraints which would ensure that types are well-defined, somehow.


None of these options seem particularly attractive.



Furthermore, we can already express basically the same functionality by using (implicit or explicit) equalities.


```wiki
f :: Array n Byte -> Array (n / 8) Word64           -- using division.

f :: Array (8 * n) Byte -> Array n Word64           -- using multiplication and an implicit equality constraint.

f :: (m ~ 8 * n) => Array m Byte -> Array n Word64  -- using an explicit equality constraint.
```


Subjectively, the second form is the most readable.  The third form shows that we are essentially using a qualified type to restrict the valid instantiation of the function.


