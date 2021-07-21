class: center, middle

# (Linear) Haskell

## Practical Linearity in a Higher‑Order Polymorphic Language

Presented by [Artem Pelenitsyn](http://staff.mmcs.sfedu.ru/~ulysses/), Northeastern U (artem@ccs.neu.edu)

---

## Linear Types: Why Care?

* Well-Typed Resource-Aware Protocols

  * Think a file handle that needs to be closed

--

* Efficiency

  * Linear Types Can Change the World! by _P. Wadler_ (1990)  
    <img src="fig/wadler-cit1.png" width="930px">

--

* Rust

---


# Linear Haskell: Challenges

* Backward Compatibility

* Reusing Existing Codebase

* Retrofitting In a “Realistic” Language

---

# Outline


1. Haskell Prelude

2. Linear Haskell Compiler Extension

3. Linear Constraints Idea (ICFP '21) 

4. General Q&A 

---

class: center, middle

# Haskell Prelude

---

# What is Haskell


---



---

class: center, middle

# Linear Haskell Extension

---

# Linear Extension Guarantees

* Does my sorting function return sensible result?
    ```haskell
        sort :: [Int] → SortedList Int
        -- vs.
        sort :: [Int] ⊸ SortedList Int
    ```
* Can I do in-place updates?

* Do I close my file handles?

---

# Meet Linear Arrows

`f :: s ⊸ t` means that: 

* **if** `(f u)` is _consumed_ exactly once,

* **then** the argument `u` is _consumed_ exactly once.

--

_Consume_ ::=

* To consume a **value of atomic base type** (like `Int` or `Ptr`) exactly once, just _evaluate_ it.

* To consume a **function** value exactly once, apply it to one argument, and consume its result exactly once.

* To consume a **value of an algebraic datatype** exactly once, pattern-match on it, and consume all its linear components exactly once.

---

# Example: List Concatenation

```haskell
-- List datatype
data List a where
  Nil  :: List a
  Cons :: a  ⊸ List a ⊸ List a

-- Linear concatenation function
concat :: List a ⊸ List a ⊸ List a
concat Nil         ys = ys
concat (Cons x xs) ys = Cons x (concat xs ys)
```

--

Data constructors are linear by default: above list is equivalent to

```haskell
data List' a = Nil' | Cons' a (List' a)
```

---

# Interplay Between `⊸` And `→` (1)

.left-half[
```haskell
f :: s ⊸ t
g :: s → t
g x = f x -- apply f to anything!
```
]
--
.right-half[
```haskell
sum :: [Int] ⊸ Int
f   :: [Int] ⊸ [Int] → Int
f xs ys = sum (xs ++ ys) + sum ys
```
]

--

.left-half[
```haskell
f1 :: (Int, Int) → (Int, Int)
f1 x = case x of (a, b) → (a, a)

f2 :: (Int, Int) ⊸ (Int, Int)
f2 x = case x of (a, b) → (b, a)

f3 :: (Int, U Int) ⊸ (Int, Int)
f3 x = case x of (a, _) → (42, a)
```
]
---

# Interplay Between `⊸` And `→` (2)

```haskell
f :: Int ⊸ Int
g :: (Int → Int) → Bool
h = g f -- Legal???
```
In general, do we want `⊸ <: →`?

--

Compiler will take care of it:

```haskell
h = g f ↝  g (λx → f x)    -- s.c. η-expansion
```

---

# Multiplicity Polymorphism

Is `map` of `(a ⊸ b) → [a] ⊸ [b]` or of `(a → b) → [a] → [b]`?

--

Neither. It is:

```haskell
map :: ∀{p} a b. (a %p → b) → [a] %p → [b]
```

--

Abbreviate as follows:


* `a ⊸ b` ::= `a %1 → b`

* `a → b` ::= `a %Many → b`

Thus, there are two concrete multiplicities: `1` and `Many`.

---

# Quiz: Typing Function Composition

```haskell
(◦) :: ∀{p} {q}.
  (b %p → c) %1 →
  (a %q → b) %p →
  a           %? →      -- what's `?`?
  c
  
(f ◦ g) x = f (g x)
```
---

count: false

# Quiz: Typing Function Composition

```haskell
(◦) :: ∀{p} {q}.
  (b %p → c) %1 →
  (a %q → b) %p →
  a           %{p·q} →
  c
  
(f ◦ g) x = f (g x)
```

---

# Can we have linear return types?

Use the double negation trick:
```haskell
f :: A → (B ⊸ r) ⊸ r -- f effectively accepts A and returns linear B
```
--

This plays nicely with monads!

```haskell
type IO p a

return :: a →_p IO p a
bind   :: IO p a ⊸ (a %p → IO q b) ⊸ IO q b
```

---

# Larger Example: Linear Input/Output

```haskell
printHandle :: File ⊸ IO ω ()
printHandle f = do
                 { (f, U b) ← atEOF f
                 ; if b then closeFile f
                   else do { (f, U c) ← read f 
                           ; putChar c
                           ; printHandle f } }
```

```haskell
atEOF     :: File ⊸ IO 1 (File, U Bool)
closeFile :: File ⊸ IO ω ()
read      :: File ⊸ IO 1 (File, U Char)
putChar   :: Char → IO ω ()
```
```haskell
do { x ← a; f x }      ~       bind a (\x → f x)
```

---

# References

* Linear Haskell: Practical Linearity In a Higher-Order Polymorphic Language / _J.-P. Bernardy et al._, POPL'18,
  [doi:10.1145/3158093](https://dl.acm.org/doi/10.1145/3158093)

* Bounded Linear Types in a Resource Semiring / _D.R. Ghica and A.I.&nbsp;Smith_, ESOP '14
    * (_or, if like dep-types:_) I Got Plenty o’ Nuttin’ / _C. McBride_, Wadler's&nbsp;Fest&nbsp;'16

* Linear Types Can Change the World! / _P. Wadler_, PCM '90,
  [[PDF]](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.31.5002&rep=rep1&type=pdf)


---

class: center, middle

# Backup

---

# Datatype Linearity a la Carte

_Task_ Function `f` needs a pair to use its components non-symmetrically  
(say, first one linearly, second — unrestrictedly)

--
```haskell
data PLU a b where 
    PLU :: a ⊸ b → PLU a b

f :: PLU (MArray Int) Int ⊸ MArray Int

```
--
Even better:
```haskell
data U a where                  -- Unrestricted resource
    U :: a → U a
    
f :: (MArray Int, U Int) ⊸ MArray Int
```

---

# Threats To Validity

* Huge overhaul: change the whole `base`, also modification of GHC Core

* Retrofitting still not completely thought of

    * GHC Proposals process is staling
    
    * Multiplicity 0 is already dropped

* Type inference is not formally developed

* Generally, elaboration of LH to $\lambda_{\to}^q$ (or GHC Core) is not 100% clear

---

# On the bright side

* A lot of work has been done.

--

* Efficiency is on its way:

    * some reported in the paper (ad-hoc);
    
    * some of «cardinality&nbsp;analysis» subsumed by multiplicity annotations
    (good for inling: e.g. don't inline `(λx → x ++ x) expensive`).

--

* Many applications in mind:

    * network-communication with 0 allocations,
    * GC-less, manual allocations,
    * resource-safe I/O,
    * safe API for the `streaming` library,
    * DSL for “printable” 3D-models.

---

# Linear Arrows vs Linear Kinds 

.right[(or: Girard was right in the first place)]

--

- Strictness & Divergence

.left-half[
```haskell
f :: a ⊸ (a, Bool)
f x = (x, True)
```
]
.right-half[
``` haskell
g :: [Int] ⊸ [Int]
g xs = repeat 1 ++ xs
```
]

- Exceptions

- Single type universe

- Can play nicely with dependent function types too as shown by Idris 2.


