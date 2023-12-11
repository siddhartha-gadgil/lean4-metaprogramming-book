```lean
import Lean
open Lean Meta
```

## `MetaM`: Solutions

### 1.

```lean
#eval show MetaM Unit from do
  let hi ← Lean.Meta.mkFreshExprMVar (Expr.const `Nat []) (userName := `hi)
  IO.println s!"value in hi: {← instantiateMVars hi}" -- ?_uniq.1

  hi.mvarId!.assign (Expr.app (Expr.const `Nat.succ []) (Expr.const ``Nat.zero []))
  IO.println s!"value in hi: {← instantiateMVars hi}" -- Nat.succ Nat.zero
```

### 2.

```lean
-- It would output the same expression we gave it - there were no metavariables to instantiate.
#eval show MetaM Unit from do
  let instantiatedExpr ← instantiateMVars (Expr.lam `x (Expr.const `Nat []) (Expr.bvar 0) BinderInfo.default)
  IO.println instantiatedExpr -- fun (x : Nat) => x
```

### 3.

```lean
#eval show MetaM Unit from do
  let oneExpr := Expr.app (Expr.const `Nat.succ []) (Expr.const ``Nat.zero [])
  let twoExpr := Expr.app (Expr.const `Nat.succ []) oneExpr

  -- Create `mvar1` with type `Nat`
  let mvar1 ← Lean.Meta.mkFreshExprMVar (Expr.const `Nat []) (userName := `mvar1)
  -- Create `mvar2` with type `Nat`
  let mvar2 ← Lean.Meta.mkFreshExprMVar (Expr.const `Nat []) (userName := `mvar2)
  -- Create `mvar3` with type `Nat`
  let mvar3 ← Lean.Meta.mkFreshExprMVar (Expr.const `Nat []) (userName := `mvar3)

  -- Assign `mvar1` to `2 + ?mvar2 + ?mvar3`
  mvar1.mvarId!.assign (Lean.mkAppN (Expr.const `Nat.add []) #[(Lean.mkAppN (Expr.const `Nat.add []) #[twoExpr, mvar2]), mvar3])

  -- Assign `mvar3` to `1`
  mvar3.mvarId!.assign oneExpr

  -- Instantiate `mvar1`, which should result in expression `2 + ?mvar2 + 1`
  let instantiatedMvar1 ← instantiateMVars mvar1
  IO.println instantiatedMvar1 -- Nat.add (Nat.add 2 ?_uniq.2) 1
```

### 4.

```lean
elab "explore" : tactic => do
  let mvarId : MVarId ← Lean.Elab.Tactic.getMainGoal
  let metavarDecl : MetavarDecl ← mvarId.getDecl

  IO.println "Our metavariable"
  -- [anonymous] : 2 = 2
  IO.println s!"\n{metavarDecl.userName} : {metavarDecl.type}"

  IO.println "\nAll of its local declarations"
  let localContext : LocalContext := metavarDecl.lctx
  for (localDecl : LocalDecl) in localContext do
    if localDecl.isImplementationDetail then
      -- (implementation detail) red : 1 = 1 → 2 = 2 → 2 = 2
      IO.println s!"\n(implementation detail) {localDecl.userName} : {localDecl.type}"
    else
      -- hA : 1 = 1
      -- hB : 2 = 2
      IO.println s!"\n{localDecl.userName} : {localDecl.type}"

theorem red (hA : 1 = 1) (hB : 2 = 2) : 2 = 2 := by
  explore
  sorry
```

### 5.

```lean
-- The type of our metavariable `2 + 2`. We want to find a `localDecl` that has the same type, and `assign` our metavariable to that `localDecl`.
elab "solve" : tactic => do
  let mvarId : MVarId ← Lean.Elab.Tactic.getMainGoal
  let metavarDecl : MetavarDecl ← mvarId.getDecl

  let localContext : LocalContext := metavarDecl.lctx
  for (localDecl : LocalDecl) in localContext do
    if ← Lean.Meta.isDefEq localDecl.type metavarDecl.type then
      mvarId.assign localDecl.toExpr

theorem redSolved (hA : 1 = 1) (hB : 2 = 2) : 2 = 2 := by
  solve
```

### 6.

```lean
def sixA : Bool → Bool := fun x => x
-- .lam `x (.const `Bool []) (.bvar 0) (Lean.BinderInfo.default)
#eval Lean.Meta.reduce (Expr.const `sixA [])

def sixB : Bool := (fun x => x) ((true && false) || true)
-- .const `Bool.true []
#eval Lean.Meta.reduce (Expr.const `sixB [])

def sixC : Nat := 800 + 2
-- .lit (Lean.Literal.natVal 802)
#eval Lean.Meta.reduce (Expr.const `sixC [])
```

### 7.

```lean
#eval show MetaM Unit from do
  let litExpr := Expr.lit (Lean.Literal.natVal 1)
  let standardExpr := Expr.app (Expr.const ``Nat.succ []) (Expr.const ``Nat.zero [])

  let isEqual ← Lean.Meta.isDefEq litExpr standardExpr
  IO.println isEqual -- true
```

### 8.

```lean
-- a) `5 =?= (fun x => 5) ((fun y : Nat → Nat => y) (fun z : Nat => z))`
-- Definitionally equal.
def expr2 := (fun x => 5) ((fun y : Nat → Nat => y) (fun z : Nat => z))
#eval show MetaM Unit from do
  let expr1 := Lean.mkNatLit 5
  let expr2 := Expr.const `expr2 []
  let isEqual ← Lean.Meta.isDefEq expr1 expr2
  IO.println isEqual -- true

-- b) `2 + 1 =?= 1 + 2`
-- Definitionally equal.
#eval show MetaM Unit from do
  let expr1 := Lean.mkAppN (Expr.const `Nat.add []) #[Lean.mkNatLit 2, Lean.mkNatLit 1]
  let expr2 := Lean.mkAppN (Expr.const `Nat.add []) #[Lean.mkNatLit 1, Lean.mkNatLit 2]
  let isEqual ← Lean.Meta.isDefEq expr1 expr2
  IO.println isEqual -- true

-- c) `?a =?= 2`, where `?a` has a type `String`
-- Not definitionally equal.
#eval show MetaM Unit from do
  let expr1 ← Lean.Meta.mkFreshExprMVar (Expr.const `String []) (userName := `expr1)
  let expr2 := Lean.mkNatLit 2
  let isEqual ← Lean.Meta.isDefEq expr1 expr2
  IO.println isEqual -- false

-- d) `?a + Int =?= "hi" + ?b`, where `?a` and `?b` don't have a type
-- Definitionally equal.
-- `?a` is assigned to `"hi"`, `?b` is assigned to `Int`.
#eval show MetaM Unit from do
  let a ← Lean.Meta.mkFreshExprMVar Option.none (userName := `a)
  let b ← Lean.Meta.mkFreshExprMVar Option.none (userName := `b)
  let expr1 := Lean.mkAppN (Expr.const `Nat.add []) #[a, Expr.const `Int []]
  let expr2 := Lean.mkAppN (Expr.const `Nat.add []) #[Lean.mkStrLit "hi", b]
  let isEqual ← Lean.Meta.isDefEq expr1 expr2
  IO.println isEqual -- true

  IO.println s!"a: {← instantiateMVars a}"
  IO.println s!"b: {← instantiateMVars b}"

-- e) `2 + ?a =?= 3`
-- Not definitionally equal.
#eval show MetaM Unit from do
  let a ← Lean.Meta.mkFreshExprMVar (Expr.const `Nat []) (userName := `a)
  let expr1 := Lean.mkAppN (Expr.const `Nat.add []) #[Lean.mkNatLit 2, a]
  let expr2 := Lean.mkNatLit 3
  let isEqual ← Lean.Meta.isDefEq expr1 expr2
  IO.println isEqual -- false

-- f) `2 + ?a =?= 2 + 1`
-- Definitionally equal.
-- `?a` is assigned to `1`.
#eval show MetaM Unit from do
  let a ← Lean.Meta.mkFreshExprMVar (Expr.const `Nat []) (userName := `a)
  let expr1 := Lean.mkAppN (Expr.const `Nat.add []) #[Lean.mkNatLit 2, a]
  let expr2 := Lean.mkAppN (Expr.const `Nat.add []) #[Lean.mkNatLit 2, Lean.mkNatLit 1]
  let isEqual ← Lean.Meta.isDefEq expr1 expr2
  IO.println isEqual -- true

  IO.println s!"a: {← instantiateMVars a}"
```

### 9.

```lean
@[reducible] def reducibleDef     : Nat := 1 -- same as `abbrev`
@[instance] def instanceDef       : Nat := 2 -- same as `instance`
def defaultDef                    : Nat := 3
@[irreducible] def irreducibleDef : Nat := 4

@[reducible] def sum := [reducibleDef, instanceDef, defaultDef, irreducibleDef]

#eval show MetaM Unit from do
  let constantExpr := Expr.const `sum []

  Meta.withTransparency Meta.TransparencyMode.reducible do
    let reducedExpr ← Meta.reduce constantExpr
    dbg_trace (← ppExpr reducedExpr) -- [1, instanceDef, defaultDef, irreducibleDef]

  Meta.withTransparency Meta.TransparencyMode.instances do
    let reducedExpr ← Meta.reduce constantExpr
    dbg_trace (← ppExpr reducedExpr) -- [1, 2, defaultDef, irreducibleDef]

  Meta.withTransparency Meta.TransparencyMode.default do
    let reducedExpr ← Meta.reduce constantExpr
    dbg_trace (← ppExpr reducedExpr) -- [1, 2, 3, irreducibleDef]

  Meta.withTransparency Meta.TransparencyMode.all do
    let reducedExpr ← Meta.reduce constantExpr
    dbg_trace (← ppExpr reducedExpr) -- [1, 2, 3, 4]

  -- Note: if we don't set the transparency mode, we get a pretty strong `TransparencyMode.default`.
  let reducedExpr ← Meta.reduce constantExpr
  dbg_trace (← ppExpr reducedExpr) -- [1, 2, 3, irreducibleDef]
```

### 10.

```lean
-- Non-idiomatic: we can only use `Lean.mkAppN`.
def tenA : MetaM Expr := do
  let body := Lean.mkAppN (Expr.const `Nat.add []) #[Lean.mkNatLit 1, Expr.bvar 0]
  return Expr.lam `x (Expr.const `Nat []) body BinderInfo.default

-- Idiomatic: we can use both `Lean.mkAppN` and `Lean.Meta.mkAppM`.
def tenB : MetaM Expr := do
  Lean.Meta.withLocalDecl `x .default (Expr.const `Nat []) (fun x => do
    -- let body := Lean.mkAppN (Expr.const `Nat.add []) #[Lean.mkNatLit 1, x]
    let body ← Lean.Meta.mkAppM `Nat.add #[Lean.mkNatLit 1, x]
    Lean.Meta.mkLambdaFVars #[x] body
  )

#eval show MetaM _ from do
  ppExpr (← tenA) -- fun x => Nat.add 1 x
#eval show MetaM _ from do
  ppExpr (← tenB) -- fun x => Nat.add 1 x
```

### 11.

```lean
def eleven : MetaM Expr :=
  return Expr.forallE `yellow (Expr.const `Nat []) (Expr.bvar 0) BinderInfo.default

#eval show MetaM _ from do
  dbg_trace (← eleven) -- forall (yellow : Nat), yellow
```

### 12.

```lean
-- Non-idiomatic: we can only use `Lean.mkApp3`.
def twelveA : MetaM Expr := do
  let nPlusOne := Expr.app (Expr.app (Expr.const `Nat.add []) (Expr.bvar 0)) (Lean.mkNatLit 1)
  let forAllBody := Lean.mkApp3 (Expr.const ``Eq []) (Expr.const `Nat []) (Expr.bvar 0) nPlusOne
  let forAll := Expr.forallE `n (Expr.const `Nat []) forAllBody BinderInfo.default
  return forAll

-- Idiomatic: we can use both `Lean.mkApp3` and `Lean.Meta.mkEq`.
def twelveB : MetaM Expr := do
  withLocalDecl `n BinderInfo.default (Expr.const `Nat []) (fun x => do
    let nPlusOne := Expr.app (Expr.app (Expr.const `Nat.add []) x) (Lean.mkNatLit 1)
    -- let forAllBody := Lean.mkApp3 (Expr.const ``Eq []) (Expr.const `Nat []) x nPlusOne
    let forAllBody ← Lean.Meta.mkEq x nPlusOne
    let forAll := mkForallFVars #[x] forAllBody
    forAll
  )

#eval show MetaM _ from do
  ppExpr (← twelveA) -- (n : Nat) → Eq Nat n (Nat.add n 1)

#eval show MetaM _ from do
  ppExpr (← twelveB) -- ∀ (n : Nat), n = Nat.add n 1
```

### 13.

```lean
def thirteen : MetaM Expr := do
  withLocalDecl `f BinderInfo.default (Expr.forallE `a (Expr.const `Nat []) (Expr.const `Nat []) .default) (fun y => do
    let lamBody ← withLocalDecl `n BinderInfo.default (Expr.const `Nat []) (fun x => do
      let fn := Expr.app y x
      let fnPlusOne := Expr.app y (Expr.app (Expr.app (Expr.const `Nat.add []) (x)) (Lean.mkNatLit 1))
      let forAllBody := mkApp3 (mkConst ``Eq []) (Expr.const `Nat []) fn fnPlusOne
      let forAll := mkForallFVars #[x] forAllBody
      forAll
    )
    let lam := mkLambdaFVars #[y] lamBody
    lam
  )

#eval show MetaM _ from do
  ppExpr (← thirteen) -- fun f => (n : Nat) → Eq Nat (f n) (f (Nat.add n 1))
```

### 14.

```lean
#eval show Lean.Elab.Term.TermElabM _ from do
  let stx : Syntax ← `(∀ (a : Prop) (b : Prop), a ∨ b → b → a ∧ a)
  let expr ← Elab.Term.elabTermAndSynthesize stx none

  let (_, _, conclusion) ← forallMetaTelescope expr
  dbg_trace conclusion -- And ?_uniq.10 ?_uniq.10

  let (_, _, conclusion) ← forallMetaBoundedTelescope expr 2
  dbg_trace conclusion -- (Or ?_uniq.14 ?_uniq.15) -> ?_uniq.15 -> (And ?_uniq.14 ?_uniq.14)

  let (_, _, conclusion) ← lambdaMetaTelescope expr
  dbg_trace conclusion -- forall (a.1 : Prop) (b.1 : Prop), (Or a.1 b.1) -> b.1 -> (And a.1 a.1)
```

### 15.

```lean
#eval show MetaM Unit from do
  let a ← Lean.Meta.mkFreshExprMVar (Expr.const `String []) (userName := `a)
  let b ← Lean.Meta.mkFreshExprMVar (Expr.sort (Nat.toLevel 1)) (userName := `b)
  -- ?a + Int
  let c := Lean.mkAppN (Expr.const `Nat.add []) #[a, Expr.const `Int []]
  -- "hi" + ?b
  let d := Lean.mkAppN (Expr.const `Nat.add []) #[Lean.mkStrLit "hi", b]

  IO.println s!"value in c: {← instantiateMVars c}" -- Nat.add ?_uniq.1 Int
  IO.println s!"value in d: {← instantiateMVars d}" -- Nat.add String ?_uniq.2

  let state : SavedState ← saveState
  IO.println "\nSaved state\n"

  if ← Lean.Meta.isDefEq c d then
    IO.println true
    IO.println s!"value in c: {← instantiateMVars c}"
    IO.println s!"value in d: {← instantiateMVars d}"

  restoreState state
  IO.println "\nRestored state\n"

  IO.println s!"value in c: {← instantiateMVars c}"
  IO.println s!"value in d: {← instantiateMVars d}"
```
