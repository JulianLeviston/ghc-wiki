## Reuse of Field Names in Haskell Records



**The Problem:** The current design of Haskell's record system does not allow for field names
to be reused  across different record types.  For example, a programmer may not define
two record types in the same module that both have a field called `name`.  Even when the records
are defined in different modules, it is inconvenient to work with records that have the same field
name.



This page describes a point in the design space that lifts this restriction, and has the following additional
benefits:


- the change is **simple** and easy to describe to a Haskell programmer,
- it is **fully compatible with other record extensions**, such as `RecordWildCards` and `NamedFieldPuns`,
- it does not preclude further record system extensions for more sophisticated behavior,
- the compiler needs to do **less work** than in the current record implementation.

## The Design



The following paragraphs describe the various operations on records.


### Declaration



Records are declared in exactly the same way as they are in Haskell'98.
However, we don't need to check that multiple record declarations in
the same module have distinct field names.



For example:


```wiki
data Person   = Person   { name :: String, age :: Int }
data Variable = Variable { name :: Int, varType :: Type }
```

### Construction



Record values are constructed in exactly the same way as in Haskell'98.  For example:


```wiki
john = Person { name = "John", age = 10 }
var  = Variable { name = 0, varType = TInt }
```


There is no ambiguity when constructing values
because the constructors are still unique for each datatype.


### Update



The update notation of Haskell'98 is not compatible with reusing
fields across records, because it leads to ambiguity.  For example,
if we write `\a -> x { name = a }` we don't know if we are setting
the field `name` for a `Person`, or `Variable`.



To work around this, we propose an alternative notation for record update,
which is similar to the notation for record construction:


```wiki
C { e | f1 = e1, f2 = e2 }
```


Here, `C` is a constructor, `e` is the record expression that is being
updated, `f1`, `f2`, etc. are the fields that are being updates, and
`e1`, `e2`, etc. are the new values of the fields.



For example, using the `process` package, we could write things like this:


```wiki
main = do h <- openFile "stdout.txt" WriteMode
          createProcess CreateProcess { shell "ls" | std_out = h }

```


Unlike Haskell'98, this notation includes the constructor, so it is
clear (both to programmers and the compiler) which field is being updated.



Also, as in Haskell'98, record update may change the types of fields.
For example:


```wiki
data Node a = Node { left :: a, node :: Int, right :: a }

makeBlank :: Node a -> Node ()
makeBlank n = Node { n | left = (), right = () }
```


Note that in this example it is important that we can
update *both* fields at the same time---we could not
change the fields one at a time, as the intermediate value
would not be well typed.  This type of update is supported
by standard Haskell records, but it is not supported by most
other proposals, including ones that generate lenses.



**RAE:** But this doesn't allow a record update to affect potentially more than one constructor. Currently, if multiple constructors of one type include the same field name, you can update all constructors. Given that this change is not backward compatible, what's the migration strategy? **End RAE**


### Selectors



We propose that **no** selector functions are automatically generated by the compiler!



While at first sight this might look a little drastic, the benefit is that the namespace
is not polluted by automatically generated accessor functions.  This frees programmers
to access fields in whatever fashion is most convenient.



The simplest way to access a field, would be via pattern matching:


```wiki
someFun1 C { x = v } = ...    -- traditional patterns
someFun2 C { x }     = ...    -- using punning
someFun3 C { .. }    = ...    -- using record wild cards
```


In addition, if there are no name clashes for some field, a programmer might
choose to revert to the standard Haskell'98 behavior:


```wiki
garfinkle :: MyType -> Int
garfinkle MyType { garfinkle = x } = x
```


On the other hand, if some other field appears in many types, and we need to access it often, then---as usual---we could
define a class:


```wiki
class Size a where
  size :: a -> Int

instance Size T1 where size T1 { size = x } = x
instance Size T2 where size T2 { size = x } = x
instance Size T3 where size T3 { length = x } = x
```


Of course, one could also manipulate fields through
TH generated lenses, or whatever mechanism they find
most convenient:  we have a wide open space to experiment
with various record accessing mechanisms.


## Optional Extensions and Variations


### Field Update With A Function



It is common to update the value of a field based on the *previous* value of the same field.
In some cases, it might be cumbersome to pattern match just to get the old value for an update.
In such situations, the following piece of syntactic sugar for record updates might be handy:


```wiki
C { e | f1 = e1, f2 := e2 }
```


As before, `C` is a constructor, `e` is the record that is being updated, `f1` is a field
that is being set to the value of `e1`.  The new syntax is in field
`f2`: the `:=` notation signifies that `e2` is an *updating function*.  This means
that the new value for `f2` will be obtained by applying `e2` to the old value of `f2`
(i.e., the value that's in `e`).


```wiki
nodeToBool :: Node Int -> Node Bool
nodeToBool n = Node { n | left := (< 0), right := (>= 0) }

increment :: Node a -> Node a
increment n = Node { n | middle := (+1) }
```

### Variation of Update Notation



An alternative to the update notation would be to put the record that is being updated *last*:


```wiki
C { f1 = e1, f2 = e2 | e }
```


Again `C` is the constructor, `f1` and `f2` are the fields being updated, and `e` is
the record that is being updated.



Both notations seem compatible with parsing Haskell.
This notation emphasizes the fields that are being changed, while the other
emphasizes the old record.  It is not immediately clear, which of the
two variations is more readable in practice.

