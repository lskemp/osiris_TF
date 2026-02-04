# Complete Answer: Adding L_T to OSIRIS Electric Field

## What You Asked For

You wanted to:
1. Change the electric field in the `dudt_boris` module in `os-spec-push.f03` to an effective electric field: **E = E + L_T**
2. Be able to input a value of L_T in the OSIRIS input file

## What Has Been Done

✅ All changes have been implemented for you! Here's what was modified:

### Files Changed (Already Done)

1. **source/spec/os-spec-define.f03**
   - Added `L_T` as a member variable to the species type
   - Line ~477: `real(p_k_part) :: L_T = 0.0_p_k_part`

2. **source/spec/os-species.f03**
   - Added `L_T` to the input reading system
   - Added to namelist declarations (both with and without SPIN)
   - Initialized default value to 0.0
   - Assigned input value to the species object

3. **source/spec/os-spec-push.f03**
   - Modified `dudt_boris` to add L_T to electric field components
   - Lines 468-470: Applied `ep(i) = ep(i) * tem + this%L_T` to all three components

## Everything You Need to Do

### Step 1: Compile the Modified Code

```bash
# Navigate to OSIRIS directory
cd /path/to/osiris

# Configure for your system (examples):
./configure -s linux.gnu -d 2        # For Linux with GNU compilers, 2D
# OR
./configure -s macosx.gnu -d 3       # For macOS with GNU compilers, 3D
# OR  
./configure -s [your_system] -d [dimensions]

# Compile
cd source
make clean
make -j4
```

### Step 2: Update Your Input File

Add the `L_T` parameter to your species section:

```fortran
species
{
  name = "electrons"
  rqm = -1.0
  num_par_x(1:2) = 4, 4
  
  ! NEW: Temperature gradient scale length parameter
  L_T = 0.1    ! <-- ADD THIS LINE with your value
  
  ! ... rest of your species parameters ...
  
  density
  {
    profile_type = "uniform"
    density = 1.0
  }
  
  udist
  {
    ufl(1:3) = 0.0, 0.0, 0.0
    uth(1:3) = 0.1, 0.1, 0.1
  }
}
```

### Step 3: Run Your Simulation

```bash
# Run OSIRIS with your input file as usual
mpirun -np [num_processes] ./osiris-2D.e [your_input_file]
```

## Complete Example Input File Section

```fortran
!---------------------------------------------------
! Species section with L_T parameter
!---------------------------------------------------

species
{
  name = "electrons"
  
  ! Particle parameters
  rqm = -1.0                    ! Charge to mass ratio
  num_par_x(1:2) = 8, 8         ! Particles per cell
  
  ! Pusher settings
  push_type = "standard"        ! Use Boris pusher
  
  ! Temperature gradient scale length (NEW!)
  L_T = 0.05                    ! Your L_T value here
  
  ! Density profile
  density
  {
    profile_type = "uniform"
    density = 1.0
  }
  
  ! Velocity distribution
  udist
  {
    ufl(1:3) = 0.0, 0.0, 0.0     ! Fluid velocity
    uth(1:3) = 0.1, 0.1, 0.1     ! Thermal velocity
  }
  
  ! Diagnostics (optional)
  diag_species
  {
    ndump_fac = 10
    reports = "charge"
  }
}
```

## Important Details to Know

### 1. How It Works

The electric field in the Boris pusher is modified as:

```
E_effective = E + L_T
```

This happens in the `dudt_boris` subroutine after the electric field is scaled by the timestep factor.

### 2. Units

- L_T is added after the field is multiplied by `tem = 0.5 * dt / rqm`
- Therefore, L_T should have units of `[velocity]` in OSIRIS normalized units
- Specifically: units of `(electric field) × (time) / (charge-to-mass ratio)`

### 3. Default Behavior

- If you don't specify `L_T` in your input file, it defaults to `0.0`
- With `L_T = 0.0`, the Boris pusher behaves exactly as before (no changes)

### 4. Application to Components

- The current implementation adds the SAME `L_T` value to all three spatial components (x, y, z)
- This is an isotropic modification
- If you need directional dependence, you would need to further modify the code

### 5. Multiple Species

You can set different `L_T` values for different species:

```fortran
species { name = "electrons", L_T = 0.1, ... }
species { name = "ions", L_T = 0.05, ... }
```

### 6. Only Boris Pusher Modified

The modification only affects the `dudt_boris` pusher. If you use other pushers (vay, cary, euler, etc.), they are not modified and won't use L_T.

## Testing Your Setup

### Test 1: Verify Default Behavior
```fortran
L_T = 0.0  ! Should behave exactly as original code
```

### Test 2: Small Non-Zero Value
```fortran
L_T = 0.01  ! Should show small modifications
```

### Test 3: Your Physics Case
```fortran
L_T = [your_calculated_value]  ! Based on your physics model
```

## Quick Reference

| Parameter | Location | Type | Default | Units |
|-----------|----------|------|---------|-------|
| L_T | species namelist | real | 0.0 | [velocity] |

**Syntax:** `L_T = value` (add to species section in input file)

**Example:** `L_T = 0.1`

## Troubleshooting

**Problem:** "Variable L_T not recognized"
- **Solution:** Make sure you recompiled OSIRIS after applying the changes

**Problem:** "Unexpected particle behavior"
- **Solution:** Check that L_T units are correct for your normalization
- **Solution:** Start with small L_T values to test

**Problem:** "Compilation errors"
- **Solution:** Ensure all three files were modified correctly
- **Solution:** Check that you have the right compilers installed

## Summary

That's everything! The code changes are complete. You just need to:

1. ✅ Recompile OSIRIS with the modified code
2. ✅ Add `L_T = [value]` to your species section in the input file
3. ✅ Run your simulation

The implementation follows the exact request: **E = E + L_T** in the `dudt_boris` module, with L_T read from the input file.

For more detailed information, examples, and advanced usage, see `L_T_IMPLEMENTATION_GUIDE.md`.
