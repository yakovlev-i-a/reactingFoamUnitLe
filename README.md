# reactingFoamUnitLe

A combustion solver with chemical reactions, based on the standard
`reactingFoam` from OpenFOAM v2606, with a **single** modification:

> **The Lewis number is set to unity (Le = 1)** for species diffusion.

## Why this matters

In the standard `reactingFoam`, the diffusion term of the species mass-fraction
transport equation is written using the dynamic viscosity:

```cpp
- fvm::laplacian(turbulence->muEff(), Yi)
```

This gives an effective species diffusion coefficient of

$$D = \frac{\mu}{\rho} = \nu,$$

which actually imposes a **Schmidt number of $Sc = 1$**, not $Le = 1$.
The resulting Lewis number is then

$$Le = \frac{\alpha}{D} = \frac{\kappa/(\rho C_p)}{\mu/\rho}
     = \frac{\kappa}{\mu C_p} = \frac{1}{Pr} \approx \frac{1}{0.7} \approx 1.43.$$

For the `2S_CH4_BFER` mechanism (Franzelli et al., *Combustion and Flame* 159,
2012, 621–637), the original paper assumes a **unity Lewis number** ($Le = 1$).
To match the paper, mass diffusion must use the same coefficient as heat
diffusion:

$$D = \alpha = \frac{\kappa}{\rho C_p}.$$

## What was changed relative to the standard reactingFoam

The only functional change is in `YEqn.H`:

```cpp
// Before (standard reactingFoam, Le = 1/Pr ≈ 1.43):
- fvm::laplacian(turbulence->muEff(), Yi)

// After (Le = 1):
- fvm::laplacian(thermo.alpha(), Yi)
```

Here `thermo.alpha()` is the laminar thermal diffusivity of enthalpy
$\kappa/C_p$ in units of `[kg/m/s]`, which is exactly $\rho D$ when $Le = 1$.

Additionally, `Make/options` fixes the non-standard variable `$(LIB_SRC)` to
`$(FOAM_SRC)` (otherwise the include paths do not build in a standard OpenFOAM
environment).

## Building

```bash
# 1. Source the OpenFOAM environment
source /usr/lib/openfoam/openfoam2606/etc/bashrc

# 2. Go to the solver directory
cd reactingFoamUnitLe

# 3. Clean any previous build (optional)
wclean

# 4. Build
wmake
```

After building, the executable is placed at:

```
$FOAM_USER_APPBIN/reactingFoamUnitLe
```

## Running

In the `system/controlDict` of your case, set:

```
application     reactingFoamUnitLe;
```

and load the required libraries (for example, for the `2S_CH4_BFER` mechanism):

```
libs
(
    "$FOAM_USER_LIBBIN/libtwoSCH4Bfer.so"
    "$FOAM_USER_LIBBIN/libfixedValueEquivalenceRatioFvPatchFields.so"
);
```

Then run it like a normal `reactingFoam`:

```bash
# serial
reactingFoamUnitLe

# or in parallel (example: 5 processes)
mpirun -np 5 reactingFoamUnitLe -parallel
```

## Requirements

- OpenFOAM v2606 (built and tested on this version).
- GCC compiler (standard OpenFOAM environment).

## License

This code is based on `reactingFoam` from OpenFOAM, which is distributed under
the **GNU GPL v3** license. Accordingly, this solver is also distributed under
**GPL v3**.

## References

- Franzelli B., Riber E., Sanjosé M., Poinsot T.
  *A two-step chemical scheme for kerosene–air premixed flames*,
  Combustion and Flame 159 (2012) 621–637.
  (the `2S_CH4_BFER` mechanism, unity Lewis number assumption)
- OpenFOAM: https://www.openfoam.com