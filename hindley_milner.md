~~~~ {literal="latex_macros"}
~~~~

![](img/titles/hindley_milner.png)

******

<!--
<blockquote>
There is nothing more practical than a good theory.
<cite>James C. Maxwell</cite>
</blockquote>
-->

<p class="halfbreak">
</p>

Hindley-Milner Inference
========================

The Hindley-Milner type system ( also referred to as Damas-Hindley-Milner or HM
) is a family of type systems that admit the serendipitous property of having a
tractable algorithm for determining types from untyped syntax. This is achieved
by a process known as *unification*, whereby the types for a well-structured
program give rise to a set of constraints that when solved always have a unique
*principal type*.

The simplest Hindley Milner type system is defined by a very short set of rules.
The first four rules describe the judgements by which we can each syntactic
construct (``Lam``, ``App``, ``Var``, ``Let``) to their expected types. We'll
elaborate on these rules shortly.

$$
\begin{array}{cl}
 \displaystyle\frac{x:\sigma \in \Gamma}{\Gamma \vdash x:\sigma} & \trule{T-Var} \\ \\
 \displaystyle\frac{\Gamma \vdash e_1:\tau_1 \rightarrow \tau_2 \quad\quad \Gamma \vdash e_2 : \tau_1 }{\Gamma \vdash e_1\ e_2 : \tau_2} & \trule{T-App} \\ \\
 \displaystyle\frac{\Gamma,\;x:\tau_1 \vdash e:\tau_2}{\Gamma \vdash \lambda\ x\ .\ e : \tau_1 \rightarrow \tau_2}& \trule{T-Lam} \\ \\
 \displaystyle\frac{\Gamma \vdash e_1:\sigma \quad\quad \Gamma,\,x:\sigma \vdash e_2:\tau}{\Gamma \vdash \mathtt{let}\ x = e_1\ \mathtt{in}\ e_2 : \tau} & \trule{T-Let} \\ \\
 \displaystyle\frac{\Gamma \vdash e: \sigma \quad \overline{\alpha} \notin \mathtt{ftv}(\Gamma)}{\Gamma \vdash e:\forall\ \overline{\alpha}\ .\ \sigma} & \trule{T-Gen}\\ \\
 \displaystyle\frac{\Gamma \vdash e: \sigma_1 \quad\quad \sigma_1 \sqsubseteq \sigma_2}{\Gamma \vdash e : \sigma_2 } & \trule{T-Inst} \\ \\
\end{array}
$$

Milner's observation was that since the typing rules map uniquely onto syntax,
we can in effect run the typing rules "backwards" and whenever we don't have a
known type for an subexpression, we "guess" by putting a fresh variable in its
place, collecting constraints about its usage induced by subsequent typing
judgements.  This is the essence of *type inference* in the ML family of
languages, that by the generation and solving of a class of unification problems
we can reconstruct the types uniquely from the syntax. The algorithm itself is
largely just the structured use of a unification solver,

However full type inference leaves us in a bit a bind, since while the problem
of inference is tractable within this simple language and trivial extensions
thereof, but nearly any major addition to the language destroys the ability to
infer types unaided by annotation or severely complicates the inference
algorithm. Nevertheless the Hindley-Milner family represents a very useful,
productive "sweet spot" in the design space.

Syntax
------

The syntax of our first type inferred language will effectively be an extension
of our untyped lambda calculus, with fixpoint operator, booleans, integers, let,
and a few basic arithmetic operations.

```haskell
type Name = String

data Expr
  = Var Name
  | App Expr Expr
  | Lam Name Expr
  | Let Name Expr Expr
  | Lit Lit
  | If Expr Expr Expr
  | Fix Expr
  | Op Binop Expr Expr
  deriving (Show, Eq, Ord)

data Lit
  = LInt Integer
  | LBool Bool
  deriving (Show, Eq, Ord)

data Binop = Add | Sub | Mul | Eql
  deriving (Eq, Ord, Show)

data Program = Program [Decl] Expr deriving Eq

type Decl = (String, Expr)
```

The parser is trivial, the only addition will be the toplevel let declarations
(``Decl``) which are joined into the global ``Program``. All toplevel
declarations must be terminated with a semicolon, although they can span
multiple lines and whitespace is ignored. So for instance:

```haskell
-- SKI combinators
let I x = x;
let K x y = x;
let S f g x = f x (g x);
```

As before ``let rec`` expressions will expand out in terms of the fixpoint
operator and are just syntactic sugar.

Types
-----

The type language we'll use starts with the simple type system we used for our
typed lambda calculus.

```haskell
newtype TVar = TV String
  deriving (Show, Eq, Ord)

data Type
  = TVar TVar
  | TCon String
  | TArr Type Type
  deriving (Show, Eq, Ord)

typeInt, typeBool :: Type
typeInt  = TCon "Int"
typeBool = TCon "Bool"
```

However we will add an additional construct that will admit a new form of
*polymorphism* for our language. *Type schemes* model polymorphic types, they
indicate that the type variables bound in quantifier are polymorphic across the
enclosed type and can be instantiated with any type consistent with the
signature.

```haskell
data Scheme = Forall [TVar] Type
```

Type schemes will be written as $\sigma$ in our typing rules.

$$
\begin{align*}
\sigma ::=\ & \tau \\
            & \forall \overline \alpha . \tau  \\
\end{align*}
$$

For example the ``id`` and the ``const`` functions would have the following
types:

$$
\begin{align*}
\text{id}    & : \forall a. a \rightarrow a \\
\text{const} & : \forall a b. a \rightarrow b \rightarrow a
\end{align*}
$$

We've now divided our types into two syntactic categories, the *monotypes* and
*polytypes*. In our simple initial languages type schemes will always be the
representation of top level signature, even if there are no polymorphic type
variables. In implementation terms this means when a monotype is yielded from
our Infer monad after inference, we will immediately generalize it at the
toplevel *closing over* all free type variables in a type scheme.

Context
-------

The typing context or environment is the central container around which all
information during the inference process is stored and queried.

~~~~ {.haskell slice="chapter7/poly/Infer.hs" lower=19 upper=19}
~~~~

The two primary operations are *extension*  and *restriction* which introduce or
remove named quantities from the context.

$$
\Gamma \backslash x = \{ y : \sigma | y : \sigma ∈ \Gamma, x \ne y \}
$$

$$
\Gamma, x:\tau = (\Gamma \backslash x) \cup \{ x :\tau \}
$$

In Haskell our implementation will simply be a newtype wrapper around a Map`
of ``Var`` to ``Scheme`` types.

~~~~ {.haskell slice="chapter7/poly/Infer.hs" lower=43 upper=44}
~~~~

Infer Monad
-----------

All our logic for type inference will live inside of the ``Infer`` monad. Which
is a monad transformer stack of ``ExcpetT`` + ``State``, allowing various error
reporting and statefully holding the fresh name supply.

~~~~ {.haskell slice="chapter7/poly/Infer.hs" lower=22 upper=22}
~~~~

Running the logic in the monad results in either a type error or a resulting
type scheme.

~~~~ {.haskell slice="chapter7/poly/Infer.hs" lower=31 upper=34}
~~~~

Substitution
------------

Two operations that will perform quite a bit are querying the free variables of
an expression and applying substitutions over expressions.

$$
\begin{align*}
\FV{x}            &= x \\
\FV{\lambda x. e} &= \FV{e} - \{ x \} \\
\FV{e_1 e_2}      &= \FV{e_1} \cup \FV{e_2} \\
\end{align*}
$$

Likewise the same pattern applies for type variables at the type level.

$$
\begin{align*}
\FTV{\alpha}                    &= \{ \alpha \} \\
\FTV{\tau_1 \rightarrow \tau_2} &= \FTV{\tau_1} \cup \FTV{\tau_2} \\
\FTV{\t{Int}}                   &= \emptyset \\
\FTV{\t{Bool}}                  &= \emptyset \\
\FTV{\forall x. t}              &= \FTV{t} - \{ x \} \\
\end{align*}
$$

Substitutions over expressions apply the substitution to local variables,
replacing the named subexpression if matched. In the case of name capture a
fresh variable is introduced.


$$
\begin{align*}
[x / e'] x &= e' \\
[x / e'] y &= y \quad (y \ne x) \\
[x / e'] (e_1 e_2) &= [x / e'] \ e_1 [x / e']  e_2 \\
[x / e'] (\lambda y. e_1) &= \lambda y. [x / e']e \quad y \ne x, y \notin \FV{e'} \\
\end{align*}
$$

And likewise, substitutions can be applied element wise over the typing
environment.

$$
[t / s] \Gamma = \{ y : [t /s] \sigma \ |\  y : \sigma \in \Gamma \}
$$

Our implementation of a substitution in Haskell is simply a Map from type
variables to types.

~~~~ {.haskell slice="chapter7/poly/Infer.hs" lower=23 upper=23}
~~~~

Composition of substitutions ( $s_1 \circ s_2$, ``s1 `compose` s2`` ) can be encoded simply
as operations over the underlying map. Importantly note that in our
implementation we have chosen the substitution to be left-biased, it is up to
the implementation of the inference algorithm to ensure that clashes do not
occur between substitutions.

~~~~ {.haskell slice="chapter7/poly/Infer.hs" lower=79 upper=83}
~~~~

The implementation in Haskell is via a series of implementations of a
``Substitutable`` typeclass which exposes the ``apply`` which applies the
substitution given over the structure of the type replacing type variables as
specified.

~~~~ {.haskell slice="chapter7/poly/Infer.hs" lower=52 upper=76}
~~~~

Throughout both the typing rules and substitutions we will require a fresh
supply of names. In this naive version we will simply use an infinite list of
strings and slice into n'th element of list per a index that we hold in a State
monad. This is a simplest implementation possible, and later we will adapt this
name generation technique to be more robust.

~~~~ {.haskell slice="chapter7/poly/Infer.hs" lower=104 upper=111}
~~~~

The creation of fresh variables will be essential for implementing the inference
rules. Whenever we encounter the first use a variable within some expression we
will create a fresh type variable.

Unification
-----------

Central to the idea of inference is the notion *unification*. A unifier is a
function $s$ for two expressions $e_1$ and $e_2$ is a relation such that:

$$
s := [n_0 / m_0, n_1 / m_1, ..., n_k / m_k] \\
[s] e_1 = [s] e_2
$$

Two terms are said to be *unifiable* if there exists a unifying substitution set
between them. A substitution set is said to be **confluent** if the application
of substitutions is independent of the order applied, i.e. if we always arrive
at the same normal form regardless of the order of substitution chosen.

The notation we'll adopt for unification is, read as two types $\tau, \tau'$ are
unifiable by a substitution $s$.

$$
\tau \sim \tau' : s
$$

Such that:

$$
[s] \tau = [s] \tau'
$$

Two identical terms are trivially unifiable by the empty unifier.

$$
c \sim c : [\ ]
$$

The unification rules for our little HM language are as follows:

$$
\begin{array}{cl}
c \sim c : []           & \quad \trule{Uni-Const} \\
\alpha \sim \alpha : [] & \quad \trule{Uni-Var} \\
\infrule{\alpha \notin \FTV{\tau}}{\alpha \sim \tau : [\alpha / \tau]}
& \quad \trule{Uni-VarLeft} \\ 
\infrule{\alpha \notin \FTV{\tau}}{\tau \sim \alpha : [\alpha / \tau]}
& \quad \trule{Uni-VarRight} \\ 
\infrule{
  \tau_1 \sim \tau_1' : \theta_1 \andalso [\theta_1] \tau_2  \sim [\theta_1] \tau_2' : \theta_2
}{
  \tau_1 \tau_2 \sim \tau_1' \tau_2' : \theta_2 \circ \theta_1
} 
& \quad \trule{Uni-Con} \\ 
\infrule{
  \tau_1 \sim \tau_1' : \theta_1 \andalso [\theta_1] \tau_2  \sim [\theta_1] \tau_2' : \theta_2
}{
  \tau_1 \rightarrow \tau_2 \sim \tau_1' \rightarrow \tau_2' : \theta_2 \circ \theta_1
} 
& \quad \trule{Uni-Arrow} \\ 
\end{array}
$$

There is a precondition for unifying two variables known as the *occurs check*
which asserts that if we are applying a substitution of variable $x$ to an
expression $e$, the variable $x$ cannot be free in $e$. Otherwise the rewrite
would diverge, constantly rewriting itself. For example the following
substitution would not pass the occurs check and would generate an infinitely
long function type.

$$
[x/x \rightarrow x]
$$

The occurs check will forbid the existence of many of the pathological livering
terms we discussed when covering the untyped lambda calculus, including the
omega combinator.

~~~~ {.haskell slice="chapter7/poly/Infer.hs" lower=101 upper=102}
~~~~

The unify function lives in the Infer monad and yields a subsitution:

~~~~ {.haskell slice="chapter7/poly/Infer.hs" lower=85 upper=99}
~~~~

Generalization and Instantiation
--------------------------------

At the heart of Hindley-Milner is two fundamental operations:

* **Generalization**: Converting a $\tau$ type into a $\sigma$ type by closing
  over all free type variables in a type scheme.
* **Instantiation**: Converting a $\sigma$ type into a $\tau$ type by creating
  fresh names for each type variable that does not appear in the current typing
  environment.

$$
\begin{array}{cl}
 \displaystyle\frac{\Gamma \vdash e: \sigma \quad \overline{\alpha} \notin \mathtt{ftv}(\Gamma)}{\Gamma \vdash e:\forall\ \overline{\alpha}\ .\ \sigma} & \trule{T-Gen}\\ \\
 \displaystyle\frac{\Gamma \vdash e: \sigma_1 \quad\quad \sigma_1 \sqsubseteq \sigma_2}{\Gamma \vdash e : \sigma_2 } & \trule{T-Inst} \\ \\
\end{array}
$$

The $\sqsubseteq$ operator in the $\trule{T-Inst}$ rule indicates that a type is
an *instantiation* of a type scheme.

$$
\forall \overline \alpha. \tau_2 \sqsubseteq  \tau_1
$$

A type $\tau_1$ is a instantiation of a type scheme $\sigma = \forall
\overline \alpha. \tau_2$ if there exists a substitution $[s] \beta = \beta$ for
all $\beta \in \mathtt{ftv}(\sigma)$ so that $\tau_1 = [s] \tau_2$. Some
examples:

$$
\begin{align*}
  \forall a. a \rightarrow a & \sqsubseteq \t{Int} \rightarrow \t{Int} \\
  \forall a. a \rightarrow a & \sqsubseteq b \rightarrow b \\
  \forall a b. a \rightarrow b \rightarrow a & \sqsubseteq \t{Int} \rightarrow \t{Bool} \rightarrow \t{Int}
\end{align*}
$$

These map very intuitively into code that simply manipulates the Haskell ``Set``
objects of variables and the fresh name supply:

~~~~ {.haskell slice="chapter7/poly/Infer.hs" lower=113 upper=121}
~~~~

By convention let-bindings are generalized as much as possible. So for
instance in the following definition ``f`` is generalized across the body of the
binding so that at each invocation of ``f`` it is instantiated with fresh type
variables.

```haskell
Poly> let f = (\x -> x) in let g = (f True) in f 3
3 : Int
```

In this expression, the type of ``f`` is generated at the let definition and
will be instantiated with two different signatures. At call site of ``f`` it
will unify with ``Int`` and the other unify with ``Bool``.

By contrast, binding ``f`` in a lambda will result in a type error.

```haskell
Poly> (\f -> let g = (f True) in (f 3)) (\x -> x)
Cannot unify types: 
	Bool
with 
	Int
```

This is the essence of *let generalization*.

Typing Rules
------------

And finally with all the typing machinery in place, we can write down the typing
rules for our simple little polymorphic lambda calculus. 

```haskell
infer :: TypeEnv -> Expr -> Infer (Subst, Type)
```

The ``infer`` maps the local typing environment and the active expression to a
2-tuple of the partial unifier solution and the intermediate type. The AST is
traversed bottom-up and constraints are solved at each level of recursion by
applying partial substitutions from unification across each partially inferred
subexpression and the local environment. If an error is encountered the
``throwError`` is called in the ``Infer`` monad and an error is reported.

~~~~ {.haskell slice="chapter7/poly/Infer.hs" lower=138 upper=185}
~~~~

Let's walk through each of the rule derivations and look how it translates into
code:

**T-Var**

The ``T-Var`` rule, simply pull the type of the variable out of the typing
context.

~~~~ {.haskell slice="chapter7/poly/Infer.hs" lower=141 upper=141}
~~~~

The function ``lookupVar`` looks up the local variable reference in typing
environment and if found it instantiates a fresh copy.

~~~~ {.haskell slice="chapter7/poly/Infer.hs" lower=131 upper=136}
~~~~

$$
\begin{array}{cl}
 \displaystyle\frac{x:\sigma \in \Gamma}{\Gamma \vdash x:\sigma} & \trule{T-Var} \\ \\
\end{array}
$$

**T-Lam**

For lambdas the variable bound by the lambda is locally scoped to the typing
environment and then the body of the expression is inferred with this scope. The
output type is a fresh type variable and is unified with the resulting inferred
type.

~~~~ {.haskell slice="chapter7/poly/Infer.hs" lower=143 upper=147}
~~~~

$$
\begin{array}{cl}
 \displaystyle\frac{\Gamma,\;x:\tau_1 \vdash e:\tau_2}{\Gamma \vdash \lambda\ x\ .\ e : \tau_1 \rightarrow \tau_2}& \trule{T-Lam} \\ \\
\end{array}
$$


**T-App**

For applications, the first argument must be a lambda expression or return a
lambda expression, so know it must be of form ``t1 -> t2`` but the output type
is not determined except by the confluence of the two values. We infer both
types, apply the constraints from the first argument over the result second
inferred type and then unify the two types with the excepted form of the entire
application expression.

~~~~ {.haskell slice="chapter7/poly/Infer.hs" lower=149 upper=154}
~~~~

$$
\begin{array}{cl}
 \displaystyle\frac{\Gamma \vdash e_1:\tau_1 \rightarrow \tau_2 \quad\quad \Gamma \vdash e_2 : \tau_1 }{\Gamma \vdash e_1\ e_2 : \tau_2} & \trule{T-App} \\ \\
\end{array}
$$

**T-Let**

As mentioned previously, let will be generalized so we will create a local
typing environment for the body of the let expression and add the generalized
inferred type let bound value to the typing environment of the body.

~~~~ {.haskell slice="chapter7/poly/Infer.hs" lower=156 upper=161}
~~~~

$$
\begin{array}{cl}
 \displaystyle\frac{\Gamma \vdash e_1:\sigma \quad\quad \Gamma,\,x:\sigma \vdash e_2:\tau}{\Gamma \vdash \mathtt{let}\ x = e_1\ \mathtt{in}\ e_2 : \tau} & \trule{T-Let} \\ \\
\end{array}
$$

**T-BinOp**

There are several builtin operations, we haven't mentioned up to now because the
typing rules are trivial. We simply unify with the preset type signature of the
operation.

~~~~ {.haskell slice="chapter7/poly/Infer.hs" lower=177 upper=182}
~~~~

~~~~ {.haskell slice="chapter7/poly/Infer.hs" lower=123 upper=129}
~~~~

$$
\begin{array}{cl}
(+) &: \t{Int} \rightarrow \t{Int} \rightarrow \t{Int} \\ 
(\times) &: \t{Int} \rightarrow \t{Int} \rightarrow \t{Int} \\ 
(-) &: \t{Int} \rightarrow \t{Int} \rightarrow \t{Int} \\ 
(=) &: \t{Int} \rightarrow \t{Int} \rightarrow \t{Bool} \\
\end{array}
$$

**Literals**

The type of literal integer and boolean types is trivially their respective
types.

$$
\begin{array}{cl}
\frac{}{\Gamma \vdash n : \mathtt{Int}}           & \trule{T-Int}  \\
\frac{}{\Gamma \vdash \t{True} : \mathtt{Bool}}   & \trule{T-True} \\
\frac{}{\Gamma \vdash \t{False} : \mathtt{Bool}}  & \trule{T-False}
\end{array}
$$

Constraint Generation
=====================

The previous implementation of Hindley Milner is simple, but has this odd
property of intermingling two separate processes: constraint solving and
traversal. Let's discuss another implementation of the inference algorithm that
does not do this. 

In the *constraint generation* approach, constraints are generated by bottom-up
traversal, added to a ordered container, canonicalized, solved, and then
possibly back-substituted over a typed AST. This will be the approach we will
use from here out, and while there is an equivalence between the "on-line
solver", using the separate constraint solver becomes easier to manage as our
type system gets more complex and we start building out the language.

Our inference monad now becomes a ``RWST`` ( Reader-Writer-State Transformer ) +
``Except`` for typing errors. The inference state remains the same, just the
fresh name supply.

```haskell
-- | Inference monad
type Infer a = (RWST
                  Env             -- Typing environment
                  [Constraint]    -- Generated constraints
                  InferState      -- Inference state
                  (Except         -- Inference errors
                    TypeError)
                  a)              -- Result

-- | Inference state
data InferState = InferState { count :: Int }
```

Instead of unifying type variables at each level of traversal, we will instead
just collect the unifiers inside the Writer and emit them with the ``uni``
function.

```haskell
-- | Unify two types
uni :: Type -> Type -> Infer ()
uni t1 t2 = tell [(t1, t2)]
```

Since the typing environment is stored in the Reader monad, we can use the
``local`` to create a locally scoped additions to the typing environment. This
is convenient for typing binders.

```haskell
-- | Extend type environment
inEnv :: (Name, Scheme) -> Infer a -> Infer a
inEnv (x, sc) m = do
  let scope e = (remove e x) `extend` (x, sc)
  local scope m
```

Typing
------

The typing rules are identical, except they now can be written down in a much
less noisy way that isn't threading so much state. All of the details are taken
care of under the hood and encoded in specific combinators manipulating the
state of our Infer monad in a way that lets focus on the domain logic.

```haskell
infer :: Expr -> Infer Type
infer expr = case expr of
  Lit (LInt _)  -> return $ typeInt
  Lit (LBool _) -> return $ typeBool

  Var x -> lookupEnv x

  Lam x e -> do
    tv <- fresh
    t <- inEnv (x, Forall [] tv) (infer e)
    return (tv `TArr` t)

  App e1 e2 -> do
    t1 <- infer e1
    t2 <- infer e2
    tv <- fresh
    uni t1 (t2 `TArr` tv)
    return tv

  Let x e1 e2 -> do
    env <- ask
    t1 <- infer e1
    let sc = generalize env t1
    t2 <- inEnv (x, sc) (infer e2)
    return t2

  Fix e1 -> do
    t1 <- infer e1
    tv <- fresh
    uni (tv `TArr` tv) t1
    return tv

  Op op e1 e2 -> do
    t1 <- infer e1
    t2 <- infer e2
    tv <- fresh
    let u1 = t1 `TArr` (t2 `TArr` tv)
        u2 = ops Map.! op
    uni u1 u2
    return tv

  If cond tr fl -> do
    t1 <- infer cond
    t2 <- infer tr
    t3 <- infer fl
    uni t1 typeBool
    uni t2 t3
    return t2
```

Constraint Solver
-----------------

The Writer layer for the Infer monad contains the generated set of constraints
emitted from inference pass. Once inference has completed we are left with a
resulting type signature full of meaningless unique fresh variables and a set of
constraints that we must solve to refine the type down to its principal type.

The constraints are pulled out solved by a separate ``Solve`` monad which holds
the Unifier ( most general unifier ) solution that when applied to generated
signature will yield the solution.

```haskell
type Constraint = (Type, Type)

type Unifier = (Subst, [Constraint])

-- | Constraint solver monad
type Solve a = StateT Unifier (ExceptT TypeError Identity) a
```

The unification logic is also identical to before, except it is now written
independent of inference and stores its partial state inside of the Solve
monad's state layer.

```haskell
unifies :: Type -> Type -> Solve Unifier
unifies t1 t2 | t1 == t2 = return emptyUnifer
unifies (TVar v) t = v `bind` t
unifies t (TVar v) = v `bind` t
unifies (TArr t1 t2) (TArr t3 t4) = unifyMany [t1, t2] [t3, t4]
unifies t1 t2 = throwError $ UnificationFail t1 t2

unifyMany :: [Type] -> [Type] -> Solve Unifier
unifyMany [] [] = return emptyUnifer
unifyMany (t1 : ts1) (t2 : ts2) =
  do (su1,cs1) <- unifies t1 t2
     (su2,cs2) <- unifyMany (apply su1 ts1) (apply su1 ts2)
     return (su2 `compose` su1, cs1 ++ cs2)
unifyMany t1 t2 = throwError $ UnificationMismatch t1 t2

```

The solver function simply iterates over the set of constraints, composing them
and applying the resulting constraint solution over the intermediate solution
eventually converting on the *most general unifier* which yields the final
subsitution which when applied over the inferred type signature, yields the
principal type solution for the expression.

```haskell
-- Unification solver
solver :: Solve Subst
solver = do
  (su, cs) <- get
  case cs of
    [] -> return su
    ((t1, t2): cs0) -> do
      (su1, cs1)  <- unifies t1 t2
      put (su1 `compose` su, cs1 ++ (apply su1 cs0))
      solver
```

This a much more elegant solution than having to intermingle inference and
solving in the same pass, and adapts itself well to the generation of a typed
Core form which we will discuss in later chapters.

Worked Example
--------------

**Example 1**

Let's walk through a few examples of how inference works for a few simple
functions. Consider:

```haskell
\x y z -> x + y + z
```

The generated type from the ``infer`` function is simply a fresh variable for
each of the arguments and return type. This is completely unconstrained.

```haskell
a -> b -> c -> e
```

The constraints induced from **T-BinOp** are emitted as we traverse both of
addition operations.

1. ``a -> b -> d  ~  Int -> Int -> Int``
2. ``d -> c -> e  ~  Int -> Int -> Int``

By applying **Uni-Arrow** we can then deduce the following set of substitutions.

1. ``a ~ Int``
2. ``b ~ Int``
3. ``c ~ Int``
4. ``d ~ Int`` 
5. ``e ~ Int``

Substituting this solution back over the type yields the inferred type:

```haskell
Int -> Int -> Int -> Int
```

**Example 2**

```haskell
compose f g x = f (g x)
```

The generated type from the ``infer`` function is again simply a set of unique
fresh variables.

```haskell
a -> b -> c -> e
```

Induced two cases of the **T-App** we get the following constraints.

1. ``b  ~  c -> d``
2. ``a  ~  d -> e``

By applying **Uni-VarLeft** twice we get the following set of substitutions.

1. ``a ~ d -> e``
2. ``b ~ c -> d``

```haskell
compose :: forall c d e. (d -> e) -> (c -> d) -> c -> e
```

If desired, you can rearrange the variables in alphabetical order to get: 

```haskell
compose :: forall a b c. (a -> b) -> (c -> a) -> c -> b
```

Evaluation
==========

Interpreter
-----------

Our evaluator will operate directly on the syntax and evaluate the results in
into a ``Value`` type.

```haskell
data Value
  = VInt Integer
  | VBool Bool
  | VClosure String Expr TermEnv
```

The interpreter is set up an Identity monad. Later it will become a more
complicated monad, but for now its quite simple. The value environment will
explicitly threaded around, and whenever a closure is created we simply store a
copy of the local environment in the closure.

```haskell
type TermEnv = Map.Map String Value
type Interpreter t = Identity t
```

Our logic for evaluation is an extension of the lambda calculus evaluator
implemented in previous chapter. However you might notice quite a few incomplete
patterns used throughout evaluation. Fear not though, the evaluation of our
program cannot "go wrong". Each of these patterns represents a state that our
type system guarantees will never happen. For example, if our program did have
not every variable referenced in scope then it would never reach evaluation to
begin with and would be rejected in our type checker. We are morally correct in
using incomplete patterns here!

```haskell
eval :: TermEnv -> Expr -> Interpreter Value
eval env expr = case expr of
  Lit (LInt k)  -> return $ VInt k
  Lit (LBool k) -> return $ VBool k

  Var x -> do
    let Just v = Map.lookup x env
    return v

  Op op a b -> do
    VInt a' <- eval env a
    VInt b' <- eval env b
    return $ (binop op) a' b'

  Lam x body -> 
    return (VClosure x body env)

  App fun arg -> do
    VClosure x body clo <- eval env fun
    argv <- eval env arg
    let nenv = Map.insert x argv clo
    eval nenv body

  Let x e body -> do
    e' <- eval env e
    let nenv = Map.insert x e' env
    eval nenv body

  If cond tr fl -> do
    VBool br <- eval env cond
    if br == True
    then eval env tr
    else eval env fl

  Fix e -> do
    eval env (App e (Fix e))

binop :: Binop -> Integer -> Integer -> Value
binop Add a b = VInt $ a + b
binop Mul a b = VInt $ a * b
binop Sub a b = VInt $ a - b
binop Eql a b = VBool $ a == b
```

Interactive Shell
-----------------

Our language has now grown out the small little shells we were using before, and
now we need something much more robust to hold the logic for our interactive
interpreter.

We will structure our REPL as a monad wrapped around IState (the interpreter
state) datatype. We will start to use the repline library from here out which
gives us platform independent readline, history, and tab completion support.

```haskell
data IState = IState
  { tyctx :: TypeEnv  -- Type environment
  , tmctx :: TermEnv  -- Value environment
  }

initState :: IState
initState = IState emptyTyenv emptyTmenv

type Repl a = HaskelineT (StateT IState IO) a

hoistErr :: Show e => Either e a -> Repl a
hoistErr (Right val) = return val
hoistErr (Left err) = do
  liftIO $ print err
  abort
```

Our language can be compiled into a standalone binary by GHC:

```bash
$ ghc --make Main.hs -o poly
$ ./poly
Poly>
```

At the top of our program we will look at the command options and allow three
variations of commands.

```bash
$ poly                # launch shell
$ poly input.ml       # launch shell with 'input.ml' loaded
$ poly test input.ml  # dump test for 'input.ml' to stdout
```

```haskell
main :: IO ()
main = do
  args <- getArgs
  case args of
    []      -> shell (return ())
    [fname] -> shell (load [fname])
    ["test", fname] -> shell (load [fname] >> browse [] >> quit ())
    _ -> putStrLn "invalid arguments"
```

The shell command takes a ``pre`` action which is run before the shell starts
up. The logic simply evaluates our Repl monad into an IO and runs that from the
main function.

```haskell
shell :: Repl a -> IO ()
shell pre
  = flip evalStateT initState
  $ evalRepl "Poly> " cmd options completer pre
```

The ``cmd`` driver is the main entry point for our program, it is executed every
time the user enters a line of input. The first argument is the line of user
input.

```haskell
cmd :: String -> Repl ()
cmd source = exec True (L.pack source)
```

The heart of our language is then the ``exec`` function which imports all the
compiler passes, runs them sequentially threading the inputs and outputs and
eventually yielding a resulting typing environment and the evaluated result of
the program. These are monoidal joined into the state of the interpreter and
then the loop yields to the next set of inputs.

```haskell
exec :: Bool -> L.Text -> Repl ()
exec update source = do
  -- Get the current interpreter state
  st <- get

  -- Parser ( returns AST )
  mod <- hoistErr $ parseModule "<stdin>" source

  -- Type Inference ( returns Typing Environment )
  tyctx' <- hoistErr $ inferTop (tyctx st) mod

  -- Create the new environment
  let st' = st { tmctx = foldl' evalDef (tmctx st) mod
               , tyctx = tyctx' <> (tyctx st)
               }

  -- Update the interpreter state
  when update (put st')
```

Repline also supports adding special casing certain sets of inputs so that they
map to builtin commands in the compiler. We will implement three of these.

Command           Action   
-----------       ------------
``:browse``       Browse the type signatures for a program
``:load <file>``  Load a program from file
``:type``         Show the type of an expression
``:quit``         Exit interpreter

Their implementations are mostly straightforward.

```haskell
options :: [(String, [String] -> Repl ())]
options = [
    ("load"   , load)
  , ("browse" , browse)
  , ("quit"   , quit)
  , ("type"   , Main.typeof)
  ]

-- :browse command
browse :: [String] -> Repl ()
browse _ = do
  st <- get
  liftIO $ mapM_ putStrLn $ ppenv (tyctx st)

-- :load command
load :: [String] -> Repl ()
load args = do
  contents <- liftIO $ L.readFile (unwords args)
  exec True contents

-- :type command
typeof :: [String] -> Repl ()
typeof args = do
  st <- get
  let arg = unwords args
  case Infer.typeof (tyctx st) arg of
    Just val -> liftIO $ putStrLn $ ppsignature (arg, val)
    Nothing -> exec False (L.pack arg)

-- :quit command
quit :: a -> Repl ()
quit _ = liftIO $ exitSuccess
```

Finally tab completion for our shell will use the interpreter's typing
environment keys to complete on the set of locally defined variables. Repline
supports prefix based tab completion where the prefix of the current command
will be used to determine what to tab complete. In the case where we start with
the command ":load" we will instead tab complete on filenames in the current
working directly instead.

```haskell
completer :: CompleterStyle (StateT IState IO)
completer = Prefix (wordCompleter comp) defaultMatcher

-- Prefix tab completer
defaultMatcher :: MonadIO m => [(String, CompletionFunc m)]
defaultMatcher = [
    (":load"  , fileCompleter)
  ]

-- Default tab completer
comp :: (Monad m, MonadState IState m) => WordCompleter m
comp n = do
  let cmds = [":load", ":browse", ":quit", ":type"]
  TypeEnv ctx <- gets tyctx
  let defs = Map.keys ctx
  return $ filter (isPrefixOf n) (cmds ++ defs)
```

Observations
============

There we have it, our first little type inferred language! Load the ``poly``
interpreter by running ``ghci Main.hs`` and the call the ``main`` function.

```haskell
$ ghci Main.hs
λ: main
Poly> :load test.ml
Poly> :browse
```

Try out some simple examples by declaring some functions at the toplevel of the
program. We can query the types of expressions interactively using the ``:type``
command which effectively just runs the expression halfway through the pipeline
and halts after typechecking.

```haskell
Poly> let id x = x
Poly> let const x y = x
Poly> let twice x = x + x

Poly> :type id
id : forall a. a -> a

Poly> :type const
const : forall a b. a -> b -> a

Poly> :type twice
twice : Int -> Int
```

Notice several important facts. Our type checker will now flat our reject
programs with scoping errors before interpretation.

```haskell
Poly> \x -> y
Not in scope: "y"
```

Also programs that are also not well-typed are now rejected outright as well.

```haskell
Poly> 1 + True
Cannot unify types: 
	Bool
with 
	Int
```

The omega combinator will not pass the occurs check.

```haskell
Poly> \x -> x x
Cannot construct the the infinite type: a = a -> b
```

The file ``test.ml`` provides a variety of tests of the little interpreter. For
instance both ``fact`` and ``fib`` functions uses the fixpoint to compute
Fibonacci numbers or factorials.

```haskell
let fact = fix (\fact -> \n -> 
  if (n == 0) 
    then 1 
    else (n * (fact (n-1))));

let rec fib n = 
  if (n == 0) 
    then 0
    else if (n==1) 
      then 1
      else ((fib (n-1)) + (fib (n-2)));
```

```haskell
Poly> :type fact
fact : Int -> Int

Poly> fact 5
120

Poly> fib 16
610
```

Full Source
===========

* [PolyML](https://github.com/sdiehl/write-you-a-haskell/tree/master/chapter7/poly)
* [PolyML - Constraint Generation](https://github.com/sdiehl/write-you-a-haskell/tree/master/chapter7/poly_constraints)
