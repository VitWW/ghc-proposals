Extra Data Commas
=====================

.. author:: Viktor WW
.. date-accepted::
.. ticket-url::
.. implemented::
.. highlight:: haskell
.. header:: This proposal is `discussed at this pull request <https://github.com/ghc-proposals/ghc-proposals/pull/749>`_.
.. sectnum::
.. contents::

This proposal suggests extending the Haskell syntax by adding alternative syntax 
for lists, records, tuples and constraint tuples with build support of trailing and leading commas.

This change aims to improve code readability and maintainability by allowing more flexibility 
in formatting lists, records, tuples and constraint tuples. 

First of all, particularly important in scenarios involving version control, code reviews, and automated code generation.


Motivation
----------

In many programming languages, including JavaScript, Python, and Rust, trailing commas in lists
(and other collection types) are a common feature.

This feature provides several benefits:

1. **Improved Diff Quality**: When adding or removing items in a list, having a trailing comma means only the relevant lines are changed in version control systems, 
   reducing the noise in diffs and making reviews easier.

.. code-block:: diff

  --- example.hs	2024-06-10 12:00:00 +0000
  +++ example.hs	2024-06-10 12:01:00 +0000
  @@ -2,5 +2,5 @@
      "apple",
      "banana",
  -   "cherry"
  +   "cherry",
  +   "peach"
      ]
      
.. code-block:: diff

  --- example.hs	2024-06-10 12:00:00 +0000
  +++ example.hs	2024-06-10 12:01:00 +0000
  @@ -2,5 +2,5 @@
      "apple",
      "banana",
      "cherry",
  +   "peach",
      ]
      
2. **Ease of Code Modification**: Developers can add new elements to the end of a list without having to modify the previous last line, which reduces the likelihood of syntax errors.

3. **Consistency in Formatting**: When generating code automatically or formatting lists in a specific way, leading and trailing commas can simplify the process.


History
~~~~~~~~~~~~

Adding trailing commas (and more rarely leading commas) is a frequently asked feature to add in Haskell.

But this task is highly divisive in the Haskell community.

Original Proposal #87 `ExtraCommas (was: Trailing and leading commas in sub-export lists) <https://github.com/ghc-proposals/ghc-proposals/pull/87>`__ 
was discussed for several years, and that discussion was so controversial 
that the author withdrew their own proposal just before the final Acceptation was received (with minor changes).

Unfortunately, redundant commas contradict with ``TupleSection`` (and presumably/conceivable ``ListSections``) extension's notation. This is inconstancy.

This proposal is an attempt to allow redundant commas more universally and consistently.


Proposed Change Specification
-----------------------------

Main idea is: 

1. To forbid redundant commas for existing data structures (records, lists and tuples, list comprehensions, constraint tuples)

2. To create an alternating syntax for data structures (records, lists, tuples, and constraint tuples) that has built support
   for trailing and leading commas.

This proposal introduces the following syntactical changes to Haskell:

1. Add language extension ``ExtraDataCommas``

2. **Comma Qualified tuples**: Allow to write ``data`` keyword after tuple close bracket `)` in tuples, 
   unboxed-tuples and constraint tuples at terms and types.

   The only difference between qualified and non-qualified lists is:

   - non-qualified tuples don't allow extra commas, but allow ``TupletSections`` (or presumably allow for constraint tuples)
   - qualified tuples allow extra commas, but ignore ``TupletSections``
     ::
  
       myTuple1 :: (Int, String, Char)
       myTuple1 = (1, "2abc", 'd') data

       myTuple2 :: (Int, Int, String, Char) data
       myTuple2 = (42, 43, "xyz", 'w') data

       unlftTuple1 = (# 1#, 'x'#, 3.2## #) data

3. **Comma Qualified solo-tuples**: Allow to write solo-tuples with ``data`` keyword ::

       mySoloTuple :: (Int) data
       mySoloTuple  = (5) data

4. **Comma Qualified lists**: Allow to write ``data`` keyword after list close bracket `]` in lists at terms and types.
   The only difference between qualified and non-qualified lists is:
   
   - non-qualified lists don't allow extra commas, but allow presumably/conceivable ``ListSections`` extension
   - qualified lists allow exra commas, but ignore presumably/conceivable ``ListSections`` extension

   ::
     
      myList1 = [1, 2, 3, 4, 5, 6, 7, 8] data

      myList2 :: [Int]
      myList2 = [
                    1, 
                    2, 
                    3, 
                    4, 
                    5, 
                    6, 
                    7, 
                    8,
                ] data

      instance (GSerialize a, GSerialize b,) data => GSerialize (a :+: b) where

      myfun1 :: forall a s. (
                    C1 a,
                    C2 a s,
                    C3 s,
                ) data. =>
                     (# 
                       SuperLongType a,
                       SuperPuperLongType a s,
                       MegaPuperLongType (Maybe a),
                     #) data
                     -> Int#
                     -> Int#
                     -> Int#
      myfun1 = ....

      myTuple3 :: (,Int, String, Char,) data
      myTuple3 = (,1, "2abc", 'd') data

      myList3 :: [Int]
      myList3 = [
                    , 1
                    , 2 
                    , 3 
                    , 4 
                    , 5 
                    , 6 
                    , 7 
                    , 8
                ] data
      
5. **Qualified trailing & leading commas**: 

   - Allow a comma before the first element in qualified 
     records, lists (expanded, non-enumerations) and tuples/constraint tuples/s-context.

   - Allow a comma after the last element in qualified 
     records, lists (expanded, non-enumerations) and tuples/constraint tuples/s-context.

   - Allow both trailing & leading commas in  same qualified structure


Syntax
~~~~~~~~~~~~
	  
The formal grammar changes for ``ExtraDataCommas``:

.. code:: abnf

    atype := gtycon
        | tyvar
        | ( [,] type1 , … , typek [,] ) [data]       (tuple type, k>=1)    --; upd
        | (# [,] type1 , … , typek [,] #) [data]     (tuple type, k>=1)    --; upd
        | '[' [,] type1 , … , typek [,] ']' [data]         (type, k>=1)    --; upd
        | '[' type ']'	                                    (list type)
        | qcon { [,] fbind1 , … , fbindn [,] } [data]         (labeled construction, n ≥ 0)   ;-- upd
        | aexp_(qcon) { [,] fbind1 , … , fbindn [,] } [data]  (labeled update, n  ≥  1)       ;-- upd
        | ……

    gtycon := qtycon
        | () [data]        (unit type)            --; upd
        | (##) [data]      (unlifted unit type)   --; upd
        | '[]' [data]      (list constructor)     --; upd
        | ……

    constr ::= con [!] atype1 … [!] atypek                  (arity con = k, k>=0)
        | con { [,] fielddecl1 , … , fielddecln [,] } [data]       (records n>=0)   ;-- upd
        | ……

    apat ::= var [ @ apat]	                                         (as pattern)
        | ……
        | '[' [,] pat1 , … , patk [,] ']' [data]            (list pattern, k ≥ 1)   ;-- upd
        | ……
        | qcon { [,] fpat1 , … , fpatk [,] } [data]      (labeled pattern, k ≥ 0)   ;-- upd


Proposed Library Change Specification
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

This is a fully backwards-compatible syntax extension, so no changes to user libraries are required.

Libraries from the GHC ecosystem may need to be updated to handle the new syntax.

Examples
--------

1. **Trailing Commas**

   List with trailing comma::

      numbers = [
        1,
        2,
        3,
      ] data

   This would be equivalent to:

   ::
   
      numbers = [1, 2, 3]
      
2. **Leading Commas**

   List with leading comma::

      fruits = [
        , "apple"
        , "banana"
        , "cherry"
      ] data
      
   This would be equivalent to: ::

      fruits = ["apple", "banana", "cherry"]
      
3. **Combined Leading and Trailing Commas**

   List with mix of leading and trailing commas::

      mixed = [
        , 1, 2
        , 3, 4, -- 5
      ] data
      
   This would be equivalent to:  ::

      mixed = [1, 2, 3, 4]


Effect and Interactions
-----------------------

We choose the **postfix variant** ``[x,y,z] qualified`` over **prefix variant** ``qualified [x,y,z]`` to avoid injections 
for ``BangPatterns``, ``AsPattern``, ``StrictPattern``,  ``Irrefutable Patterns``, ``Specified(Qualified)Literals`` ::

    -- Bang Patterns
    let !(,p,q,) data = e in body
    
    let (!x, ![y,] data) = e in body

    -- As Pattern
    foo1 :: (a, b) -> a
    foo1 t@(,p,q,) data = t

    -- Specified(Qualified) Literals
    bar2 a b  = M.[a, b,] data
    
    -- StrictPattern
    data T = MkT ~(Int, Int, Int,) data
    
    -- Irrefutable Patterns
    let ~[a,b,] data = expr in e0 a b

Tuple Section
~~~~~~~~~~~~~~~~~~

Both ``TupleSection`` extension and presumable/conceivable ``ListSection`` extension
are fully consistent and compatible with ``ExtraDataCommas``.


Costs and Drawbacks
-------------------

We expect the implementation and maintenance costs of ``ExtraDataCommas`` has medium difficulty.


Backward Compatibility
---------------------------------

This change is backward compatible with existing Haskell code,
as it introduces new syntactical permissiveness without altering the existing valid syntax.

All current Haskell programs will remain valid and unchanged in their behavior.

Alternatives
------------

The primary alternative is "status quo".

Alternative Syntax
~~~~~~~~~~~~~~~~~~

Alternative to ``(x, y, z) data`` and ``[x, y, z] data`` syntax we could choose alternative syntax,
like ``q(x, y, z)`` and ``q[x, y, z]`` or ``%(x, y, z)`` and ``%[x, y, z]``. Or alternative keywords could be chosen, like ``qualified``.


Unresolved Questions
--------------------

Since the length of the ``qualified`` keyword is long it is better to have some Unicode ``Char`` equivalent in ``UnicodeSyntax``.

But is unclear what to choose.


Implementation Plan
-------------------

It is unclear.


Acknowledgments
----------------

Thanks to all contributors of `ExtraCommas (was: Trailing and leading commas in sub-export lists) 
<https://github.com/ghc-proposals/ghc-proposals/pull/87>`__.

Thanks to all contributors of `Add Support for Trailing and Leading Commas in Lists 
<https://github.com/ghc-proposals/ghc-proposals/pull/658>`__.

Thanks to all contributors of `Allow Trailing Comma in List Constructor Syntaxs 
<https://github.com/ghc-proposals/ghc-proposals/issues/653>`__.


Endorsements
-------------
