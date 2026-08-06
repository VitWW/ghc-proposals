====================
Local Quantifiers
====================

.. author:: Viktor WW
.. date-accepted::
.. ticket-url:: 
.. implemented::
.. highlight:: haskell
.. header:: This proposal is `discussed at this pull request <https://github.com/ghc-proposals/ghc-proposals/pull/710>`_.
.. sectnum::
.. contents::

.. _`#448`: https://github.com/ghc-proposals/ghc-proposals/blob/master/proposals/0448-type-variable-scoping.rst


This proposal introduces local quantifiers into GHC, which grabs local type variables.

Motivation
----------

With GHC's powerful type-level programming features, we need powerful abilities 
to explicitly bring local type variables into the scope. 

To write some trivial functions ``ScopedTypeVariables`` extension is needed (or ``TypeAbstractions``), 
which adds implicit rules for how to read type signatures!

And this is a bit unhandy to use implicit-only rules in a language that has a huge system of types.

Right now there are three kinds of scoped:

* The scope local to each type signature
::

  -- 'a' is introduced and used only within this type signature
  res :: a -> a
  --res :: forall a. a -> a
  res x = x

* The lexical scope around a type signature (modified by ``TypeAbstractions``)
::

  {-# LANGUAGE TypeAbstractions #-}
  const :: forall b. b -> c -> b
  const @a x = res where
    -- uses 'a' from the lexical scope '@a'
    res :: a
  --res :: forterm a. a
    res = x

* The magical / borrowed scope introduced by ``ScopedTypeVariables``
::

  {-# LANGUAGE ScopedTypeVariables #-}
  const :: forall a. a -> b -> a
  const x = res where
    -- uses 'a' from parent signature 'const :: forall a.'
    res :: a
  --res :: fortype a. a
    res = x

Notice how the signature ``res :: a`` in (2) and (3) 
does not itself say where ``a`` comes from.

This is confusing because traditionally Haskell has made it optional 
to write "forall" in a signature, so it is unclear if ``res :: a`` means ``res :: forall a. a`` 
or the other thing (and it's not even possible to express what 
that the other meaning is currently!).

This proposal says such uses of a should explicitly say they use '``a``' 
from somewhere else (exactly which-where) in the program.

This Proposal suggests to add the **ForTerm Quantifier**, **ForType Quantifier** 
and **ForArgm Quantifier** which allow to write explicitly type signatures, 
which depends from internal or external type variables.

Explicitness is preferential in Haskell over implicitness. 
And this Proposal propose how to write local quantifiers explicitly! 
It does not aim to allow writing more programs, just to allow being more explicit 
about where type variables come from. 
Non-quantified type variable means that this variable is somehow-quantified.

Just like ``ExplicitForall`` extension allow explicitly say exactly what 
this specific type variable is ``forall`` quantified, 
this Proposal allow to switch on ``LocalQuantifiers`` extension explicitly say 
exactly what this specific type variable is local quantified!

An additional advantage is that adding such quantifiers makes signatures 
that have one-to-one correspondences with pure mathematical descriptions in Predicate Logic!
 
Main alternative is "Modern Scoped Type Variables" `#448`_ which was added into 
``ScopedTypeVariables`` extension and ``TypeAbstractions`` extension.

``ScopedTypeVariables`` and partly ``TypeAbstractions`` are *de facto* **Implicit Local Quantifiers** : 
Implicit rules to add a local scope (or universal) quantifier to type variables 
if they are not explicitly quantified.

Also Alternative is ``PartialTypeSignatures`` extension, 
with opposite philosophy: compiler infer type not for holes.


Rule (aka math-like proof)
~~~~~~~~~~~~~~~~~~~~~~~~~~

De facto Local Quantifiers are a special case of Existential Quantifier 
(Existential Unique Quantifier), which is known during compile time.

Some people have doubt that this Proposal use correct theoretical term names.

See more details in "Unresolved Questions" section.

Author of this proposal use "Duck typing logic":

- if it looks like "quantifier" then it is a "quantifier"

- if it looks like "local quantifier" then it is a "local quantifier"

- if it looks like "existential quantifier" then it is an "existential quantifier"

- if it looks like "unique quantifier" then it is an "unique quantifier"

**Math-like Proof:**

All local scoped and parametric non-quantified type variables in Haskell 
are unique quantified (if not ``forall``-quantified) type variables.

::

  -- pseudo-haskell
  
  f1 :: ∀ a. [a] -> [a]
  f1 (x:xs) = xs ++ [ x :: ∃! b. b ]

  f :: ∀ a. [a] -> [a]
  f xs = ys ++ ys
     where
       ys :: ∃! b. [b]
       ys = reverse xs

If we use mathematical induction we could show that all "similar" cases could use unique quantifier.

Main benefit is that all local quantifiers are utilized by Haskell-renamer,
so nothing is required to change in Core-language.

Local Quantifiers are just explanation to GHC which external type variable they means: 
they indicates the binding site of the type variable (e.g. whether it was bound by a type abstraction, 
a scoped type variable bound in a type signature, or somewhere else).


Proposed Change Specification
-----------------------------

Local Quantifiers "grab"(use) already existed type variables external to this signature
::

  f :: forall a b. [a] -> [b] -> [(a, b)]
  f @aa @bb xs ys  = zip (xs :: forterm aa. [aa]) yys
     where
       yys :: fortype b. [b]
       yys = reverse ys


By using ``for<local> a`` quantifier we ask do not create a new type variable ``forall a``, 
but use already existed external type variable ``a``.

1. ForTerm ``forterm`` quantifier pick type variable **by name** 
   lifted from nearest explicit **type-term** argument 
   (full or partial either ``@tyterm`` or ``(type a)`` or ``a`` in place which 
   is responsible from ``forall ->`` quantifier), not from **type**.

2. ForType ``fortype`` quantifier pick type variable **by name** 
   from explicit only signature declaration from nearest sibling ones, 
   then from parent one ans so on, except siblings of top declaration signatures.

3. ForArgm ``forargm`` quantifier pick type variable **by name** 
   from ``class``, ``instance``, ``data``, ``type`` and ``newtype`` head type variable.
   
   In ``data``, ``type`` and ``newtype`` signatures is also allowed to write "old way" - 
   with ``forall`` quantifier without warning. Order of type variables has meaning.

   In ``class`` and ``instance`` signatures it is the only allowed quantifier - 
   Order of type variables hasn't any meaning. 
   But for future Backward Compatibility it is better to write first with head declaration order.


Since ``forterm`` , ``fortype`` , ``forargm`` are quantifier by picking by name, 
they must use same **name** for type variable as external ones.

Extension
~~~~~~~~~~~~

Introduce a new extension ``LocalQuantifiers`` .

With ``LocalQuantifiers`` words ``forterm``, ``fortype``, ``forargm``  becomes keywords in types.

Syntax
~~~~~~

Syntax for local quantifiers has a simple form.

.. code:: abnf

  quantifiers ::= { quantifier }

  quantifier  ::=
    | 'forall'  { tyvar } tyvar ( '.' | '->' )
    | 'forargm' { tyvar } tyvar   '.'
    | 'forterm' { tyvar } tyvar   '.'
    | 'fortype' { tyvar } tyvar   '.'


With ``-XModifiers``, introduce modifier syntax on forall type variables if we don't want to mix quantifiers

.. code:: abnf

  quantifier ::= ......
       | 'forall' { modifiers tyvar } modifiers tyvar ( '.' | '->' )


where ``%forargm``, ``%forterm`` and ``%fortype`` are modifiers.


Every local quantifier is utilized by the Haskell renamer, so no changes are required for the Core Language.

Examples
--------

ForTerm Quantifier
~~~~~~~~~~~~~~~~~~

Examples uses ForTerm Quantifier
::

  -- Example 1
  data T = forall a. MkT [a] (a -> Int)
			
  f :: T -> [Int]
  f (MkT @a xs f) = let mf :: forterm a. [a] -> [Int]
                        mf = map f
                    in mf xs

  -- Example 2
  foo :: forall b. Maybe b -> ()
  foo @a (_ :: forterm a. Maybe a) = ()

  -- Example 3
  bar :: forall b. Maybe b -> ()
  bar (Just @a (_ :: forterm a. a)) = ()

  -- Example 4
  baz :: forall c. c ~ () -> ()
  baz @b () = ()
    where
      () :: forterm b. b = ()
	  
  -- Example 5
  data T a where
    MkT1 :: forall a.              T a
    MkT2 :: forall a.              T (a,a)
    MkT3 :: forall a b.            T a
    MkT4 :: forall a b. b ~ Int => T a
    MkT5 :: forall a b c. b ~ c => T a

  foo :: T (Int, Int) -> ()
  foo (MkT1 @(Int,Int))  = ()
  foo (MkT2 @x)          = (() :: forterm x. x ~ Int => ())
  foo (MkT3 @_ @x)       = (() :: forterm x. x ~ x => ())
  foo (MkT4 @_ @x)       = (() :: forterm x. x ~ Int => ())

  -- Example 6
  f :: Maybe Int -> Int
  f (Nothing @a) = (4 :: forterm a. a)
  f (Just @a _)  = (5 :: forterm a. a)
  
  -- Example 6
  g :: forall a. a -> a
  g @a x = (x :: forterm a. a)

  -- Example 7  
  
  -- accepted
  f8 @a (x :: forterm a. a) = x 

  -- accepted
  f2 @a True  x (y :: forterm a. a) = x
  f2 @_ False x y                   = y

  -- rejected: too confusing to have different type variable bindings
  f3 @a True  x (y :: forterm a. a) = x
  f3    False x y                   = y

  -- accepted: the type signature allows us to do this
  f4 :: Bool -> a -> a -> a
  f4 @a True  x (y :: forterm a. a) = x
  f4    False x y                   = y

  -- accepted
  f5 :: Bool -> forall a. a -> a -> a
  f5 True @a x (y :: forterm a. a) = x
  f5 False   x y                   = y
  
  -- Example 8
  id :: forall a. a -> a
  id @t x = x :: forterm t. t

ForType Quantifier
~~~~~~~~~~~~~~~~~~~~~~~~

Examples uses ForType Quantifier
::

  -- Example 1
  f1 :: forall a. [a] -> [a]
  f1 (x:xs) = xs ++ [ x :: fortype  a. a ]
  
  -- Example 2
  f2 :: forall a. [a] -> [a]
  f2 (x:xs) = xs ++ [ x :: fortype a. a ]

  -- Example 3
  f :: [a] -> [b] -> [(a, b)]  
  f xs ys = zip (xs :: fortype a. [a]) yys 
     where
       yys :: fortype b. [b]
       yys = reverse ys

  -- Example 4
  f :: forall a b c. [a] -> [b] -> c -> ....
  f xs ys z = .....
    where
      zzs :: fortype c. [c]
      zzs = [z, z, z] 
      yys :: fortype b. [b]
      yys = reverse ys
      x2 :: forall d. d -> ....
      x2 t = ...
        where
          x3 :: fortype a. a
          x3 = head xs
          xt :: fortype a d. (d, a)
          xt = (t, x3)

ForArgm Quantifier
~~~~~~~~~~~~~~~~~~

Examples uses ForArgm Quantifier
::

  -- Example 1
  class C a where
    foo :: forargm a. forall b. b -> a -> (a, [b])

  -- Example 2
  class Trans t where
    lift :: forargm t. forall m. Monad m => m a -> (t m) a
	
  -- Example 3
  class C a where
    op :: forargm a. [a] -> a
  
    op xs = let ys:: forargm a. [a]
                ys = reverse xs
            in
            head ys
			
  -- Example 4
  instance C b => C [b] where
    op xs = reverse (head (xs :: forargm b. [[b]]))

  -- Example 5	
  class D a where
    m :: forargm a. a -> a

  instance Num a => D [a] where
    m :: forargm a. [a] -> [a]
    m x = map (*2) x
	
  -- Example 6
  class Collects e ce | ce -> e where
    empty  :: forargm ce. ce
    insert :: forargm e ce. e -> ce -> ce
    member :: forargm e ce. e -> ce -> Bool


Example uses both ForArg and ForTerm Quantifiers:
::

  type C :: forall i. (i -> i -> i) -> Constraint
  class C @i a where
    p :: forargm a. forterm i. P a i
  
New alternative way to write data declarations:
::

  -- Example 1
  data T a where
    MkT1 :: forargm a.                        T a
    MkT2 :: forargm a.                        T (a,a)
    MkT3 :: forargm a. forall b.              T a
    MkT4 :: forargm a. forall b. b ~ Int =>   T a
    -- with Modifiers extension
    MkT5 :: forall c %forargm a b. b ~ c =>   T a


Effect and Interactions
-----------------------

This proposals affect a lot of extensions, but mostly with "natural" way.

UnicodeSyntax
~~~~~~~~~~~~~~

We wish to preserve ``∃`` (There Exists, U+2203) symbol for universal existential quantifier, 
so it is proposed to add 3 symbols (``∃!`` + ``<something>``) to represent local quantifiers.

1. ``∃!@`` could represent ``forterm`` quantifier (There Exists, U+2203) + (Exclamation Mark, U+0021) + (Commercial At, U+0040).
   
   Maybe for not confusing with "at"-symbol it is better to allow (Fullwidth Commercial At, U+FF20) - ``∃!＠``

2. ``∃!≡`` could represent ``fortype`` quantifier (There Exists, U+2203) + (Exclamation Mark, U+0021) + (Identical To, U+2261).

3. ``∃!≝`` could represent ``forargm`` quantifier (There Exists, U+2203) + (Exclamation Mark, U+0021) + (Equal to By Definition, U+2254).


Maybe also it affects modifiers - ``%∃!＠``, ``%∃!≡`` and ``%∃!≝``.

Examples
::

  id :: ∀ a. a -> a
  id @t x = x :: ∃!＠ t. t

  f1 :: ∀ a b. [a] -> [b] -> [(a, b)]
  f1 @aa @bb xs ys  = zip (xs :: ∃!＠ aa. [aa]) yys
     where
       yys :: ∃!≡ b. [b]
       yys = reverse ys

  class D a where
    m :: ∃!≝ a. a -> a

  instance Num a => D [a] where
    m :: ∃!≝ a. [a] -> [a]
    m x = map (*2) x

Modifiers
~~~~~~~~~

We allow to write ``%forargm``, ``%forterm`` and ``%fortype`` (and ``%foreach``) as modifiers 
for ``forall`` type variable declarations near (before) type variable.

ScopedTypeVariables
~~~~~~~~~~~~~~~~~~~

``ScopedTypeVariables`` extension ignores local quantified variables.

But we could reuse part of searching algorithms from ``ScopedTypeVariables`` algorithms.

ScopedTypeAbstractions
~~~~~~~~~~~~~~~~~~~~~~

``TypeAbstractions`` extension ignores local quantified variables.

But it has build lexical scoping searching rules for unquantified type variables, 
which it is better to segregate later into new ``JustTypeAbstractions`` and ``ScopeForTypeAbstractions`` extension.

.. code:: none

    TypeAbstraction = JustTypeAbstractions + ScopeForTypeAbstractions

Visible ForAll and ForEach
~~~~~~~~~~~~~~~~~~~~~~~~~~

Since local quantifiers just use already existing type variables, 
there is no need to be used as visible or as unerased quantifiers.

NoImplicitForAll
~~~~~~~~~~~~~~~~

This Proposal do not contradicts ``NoImplicitForAll`` extension.

CurriedQuantifiers
~~~~~~~~~~~~~~~~~~

This proposal is better to use with curried / nested quantifiers (foralls) features.


Costs and Drawbacks
-------------------

We expect the implementation and maintenance costs of ``LocalQuantifiers`` has medium difficulty.


Alternatives
------------

Main alternative is "Modern Scoped Type Variables" `#448`_ (``ScopedTypeVariables`` extension), 
but also ``TypeAbstractions`` and ``PartialTypeSignatures``.

Alternative keywords
~~~~~~~~~~~~~~~~~~~~

We could choose different keywords instead of proposed latin and unicode keywords.

However, the template ``for<local>`` and ``∃<something>`` are welcomed.


Backward Compatibility
----------------------

This proposal is fully backward compatible.


Unresolved Questions
--------------------

Most unresolved question are theoretical: how to to call right local quantifiers.

Is it Quantifier? Is it Existential? Is it Unique?
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Some people think, that it it wrong naming: maybe it is more carefully 
to call "local quantifier" as either "pseudo-quantifier" or "quasi-quantifier".

Also some people think, that is wrong naming: 
either it call "existential quantifier" or "unique quantifier".

1) `Adam Gundry <https://github.com/adamgundry>`__ said:
     
    I don't think you are using the term "quantifier" as it is normally 
    understood in logic or type theory, and it is difficult 
    to unpack what you mean.

    I think the question of the relationship of this proposal to 
    (unique) existential quantification is a red herring. 
    The core idea of the proposal seems to be that a "local quantifier" 
    is an annotation on a type signature that does not itself bind a type variable 
    or quantify over types, but rather indicates the binding site of the type variable 
    (e.g. whether it was bound by a type abstraction, a scoped type variable 
    bound in a type signature, or somewhere else). 
    This does not increase expressivity, but allows the programmer to be more explicit.

2) `Jaro <https://github.com/noughtmare>`__ said:

    Here's a proof in Agda that ``∃! b. Bool → b`` is false:

    ::
	
      open import Relation.Nullary.Negation
      open import Data.Product
      open import Data.Bool
      open import Relation.Binary.PropositionalEquality
      open import Data.Nat

      postulate Bool≠ℕ : ¬ (Bool ≡ ℕ)

      foo : Bool → Bool
      foo x = x

      bar : Bool → ℕ
      bar false = zero
      bar true = suc zero

      qux : ¬ (∃! _≡_ λ b → Bool → b)
      qux (A , this , that≡this) with that≡this foo | that≡this bar
      ... | refl | Bool≡ℕ = Bool≠ℕ Bool≡ℕ
	  

    I needed to postulate that Bool is not ℕ because I think that is a bit hard to prove.
	 
    ...
    The "truth" of a type in a functional language is, by 
    the Curry-Howard correspondence, whether it is inhabited or not.
	 
    ...
    The most confusing part seems to be the lambda. 
    You should really read it more like ``∃![ b ] (Bool → b)``. 
    It's just that the lambda is the only way to bind new variables 
    and ``∃!`` needs to bind the ``b`` variable in this case.
	 
    (The ``_≡_`` argument is just for saying up to which kind 
    of equality we want it to be unique. 
    In this case it is propositional equality which is the built-in equality in Agda. 
    We could instead use isomorphisms on types as our notion of equality, 
    which would also allow us to prove that ``Bool`` is not ``ℕ`` more easily.)
	 
    If I wanted to say that there is a unique function 
    then I'd write that something like this:
	 
    ::
	
      ¬ (∃! {A = Bool → ?} _≡_ λ f → ⊤)

3) `Tom Ellis <https://github.com/tomjaguarpaw>`__ said:

    I think that this is correct: ``∃!`` doesn't correspond to 
    uniqueness quantification in a Curry Howard interpretation. 
    In fact I'm not sure that ``∃! b. Bool -> b`` is a type at all. 
    Rather it seems to be a property of a type ``t``, 
    specifically ``∃! b. t`` is satisfied by types ``t'`` for which 
    there exists a type ``s`` such that ``t[b -> s] = t'``.

ForWhich
~~~~~~~~~

It is unclear which local quantifier should be used in next example
::

    data Proxy a = P

    g2 :: forall a. Proxy (Nothing @(a, a)) -> ()
    g2 (P @(Nothing :: for??? t. Maybe (t, t))) = ()


Is it any? Or it is more correct to describe it as "as-pattern" in signatures at ``KindSignatures`` for that?

::

    g2 (P @(Nothing :: Maybe ( t@Type , t))) = ()

This proposal do not cover this example.


Implementation Plan
-------------------

Unclear. The author cannot implement this proposal.


Acknowledgments
---------------


Endorsements
------------
