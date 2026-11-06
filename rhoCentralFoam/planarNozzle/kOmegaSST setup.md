# Planar Supersonic Nozzle in OpenFOAM: k-ω SST Turbulence Setup

> **Status:** The first run of this case was **laminar** and is **incomplete**. After about 2 days of computing the flow had not fully settled, so the saved results show transient behaviour (startup waves, jet front reaching the outlet). This document describes how to switch the case to the **k-ω SST** RAS model. Treat the results as a learning milestone, not a validated result.

Patch names below (`inlet`, `outlet`, `walls`, `farfield`) are placeholders. Replace them with the patch names in your own `constant/polyMesh/boundary`. Tested syntax is for OpenFOAM v2606 (ESI); check keyword names against your version.

---

## 1. Files to add in `0/`

### `0/k`

```
FoamFile { version 2.0; format ascii; class volScalarField; object k; }

dimensions      [0 2 -2 0 0 0 0];
internalField   uniform 1;          // k = 1.5*(I*U)^2

boundaryField
{
    inlet       { type fixedValue; value uniform 1; }
    outlet      { type inletOutlet; inletValue uniform 1; value uniform 1; }
    farfield    { type inletOutlet; inletValue uniform 1; value uniform 1; }
    walls       { type kqRWallFunction; value uniform 1; }
}
```

### `0/omega`

```
FoamFile { version 2.0; format ascii; class volScalarField; object omega; }

dimensions      [0 0 -1 0 0 0 0];
internalField   uniform 1000;       // omega = sqrt(k)/(Cmu^0.25 * L)

boundaryField
{
    inlet       { type fixedValue; value uniform 1000; }
    outlet      { type inletOutlet; inletValue uniform 1000; value uniform 1000; }
    farfield    { type inletOutlet; inletValue uniform 1000; value uniform 1000; }
    walls       { type omegaWallFunction; value uniform 1000; }
}
```

### `0/nut`

```
FoamFile { version 2.0; format ascii; class volScalarField; object nut; }

dimensions      [0 2 -1 0 0 0 0];
internalField   uniform 0;

boundaryField
{
    inlet       { type calculated; value uniform 0; }
    outlet      { type calculated; value uniform 0; }
    farfield    { type calculated; value uniform 0; }
    walls       { type nutkWallFunction; value uniform 0; }
}
```

### `0/alphat`

Compressible solvers (for example `rhoCentralFoam`) also read this field.

```
FoamFile { version 2.0; format ascii; class volScalarField; object alphat; }

dimensions      [1 -1 -1 0 0 0 0];
internalField   uniform 0;

boundaryField
{
    inlet       { type calculated; value uniform 0; }
    outlet      { type calculated; value uniform 0; }
    farfield    { type calculated; value uniform 0; }
    walls       { type compressible::alphatWallFunction; Prt 0.85; value uniform 0; }
}
```

---

## 2. Switch the model on: `constant/momentumTransport`

```
simulationType  RAS;

RAS
{
    model           kOmegaSST;
    turbulence      on;
    printCoeffs     on;
}
```

---

## 3. `system/` entries

### `system/fvSchemes` (in `divSchemes`)

```
div(phi,k)      Gauss upwind;
div(phi,omega)  Gauss upwind;
```

If the case already has a `div(tauMC)` or `div(((rho*nuEff)*dev2(T(grad(U)))))` entry, keep it. The exact names depend on the solver, so match any "keyword not found" error against your own `fvSchemes`.
### outside divSchemes 
add this wall distance method 
```
wallDist
{
    method     meshWave
}
```

### `system/fvSolution` (in `solvers`)

```
"(k|omega)"
{
    solver          smoothSolver;
    smoother        symGaussSeidel;
    tolerance       1e-10;
    relTol          0;
}
```

---

## 4. Choosing the values

| Quantity | Estimate |
|---|---|
| Turbulence intensity `I` | about 1–5 % at a nozzle inlet; about 0.1–1 % in the ambient region |
| `k` | `1.5 (I·U)²` |
| Length scale `L` | about `0.07 × (nozzle height)` |
| `omega` | `sqrt(k) / (Cμ^0.25 · L)`, with `Cμ = 0.09` |

Use small ambient values of `k` and `omega`. Otherwise the jet shear layer gets far too much eddy viscosity.

---

## 5. Things to watch

- **Wall resolution:** k-ω SST wants `y+ ≈ 1` for a low-Re treatment. `omegaWallFunction` also copes with coarser meshes, but the nozzle wall boundary layer still needs reasonable resolution.
- **Shock cells:** SST is known to over-mix and shorten the shock-cell train in underexpanded jets. Expect a different jet length than in the laminar run. This laminar vs SST comparison is a useful result in itself.
- **Restarting:** The laminar run was not settled, so starting fresh with the new fields is a reasonable option. Restarting from the latest time also works if you add `k`, `omega`, `nut` and `alphat` to that time directory.
- **Cost:** Run in parallel (`decomposePar`, then `mpirun -np N <solver> -parallel`) and consider a smaller or graded domain to cut run time.

---

