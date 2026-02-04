# Guide: Adding L_T Parameter to OSIRIS Species

## Overview

This guide explains the implementation of the `L_T` (temperature gradient scale length) parameter in OSIRIS and how to use it in your simulations.

## What Was Implemented

The implementation adds a new input parameter `L_T` that modifies the electric field in the Boris pusher according to:

**E_effective = E + L_T**

This modification is applied to all three spatial components (x, y, z) of the electric field in the `dudt_boris` subroutine.

## Files Modified

### 1. `source/spec/os-spec-define.f03`
**Changes:** Added `L_T` as a member variable to the `t_species` type definition.

```fortran
! Temperature gradient scale length parameter
real(p_k_part) :: L_T = 0.0_p_k_part
```

**Location:** After line 475, following the radiation reaction parameters (`k_rr` and `rad_react`)

### 2. `source/spec/os-species.f03`
**Changes:** Modified the `read_input_species` subroutine to:
- Declare `L_T` as a local variable
- Add `L_T` to the input namelist (both `#ifdef __HAS_SPIN__` and `#else` branches)
- Initialize `L_T` with default value of `0.0_p_k_part`
- Assign the input value to `this%L_T`

### 3. `source/spec/os-spec-push.f03`
**Changes:** Modified the `dudt_boris` subroutine to add `L_T` to each electric field component after it has been scaled by the timestep factor.

```fortran
! Apply effective electric field modification with temperature gradient scale length
! Note: L_T should have units consistent with electric field * time / rqm
do i=1, np
  ep(1,i) = ep(1,i) * tem + this%L_T
  ep(2,i) = ep(2,i) * tem + this%L_T
  ep(3,i) = ep(3,i) * tem + this%L_T
end do
```

## How to Use L_T in Your Input File

### Basic Usage

Add the `L_T` parameter to the species namelist section of your OSIRIS input file:

```fortran
species
{
  name = "electrons"
  rqm = -1.0
  num_par_x(1:2) = 4, 4
  
  ! Temperature gradient scale length parameter
  L_T = 0.1
  
  ! ... other species parameters ...
}
```

### Multiple Species

You can set different `L_T` values for different species:

```fortran
species
{
  name = "electrons"
  rqm = -1.0
  num_par_x(1:2) = 4, 4
  L_T = 0.1
}

species
{
  name = "ions"
  rqm = 1836.0
  num_par_x(1:2) = 4, 4
  L_T = 0.0  ! or omit for default
}
```

### Default Behavior

If `L_T` is not specified in the input file, it defaults to `0.0`, which means no modification to the electric field (standard Boris pusher behavior).

## Important Notes on Units

### Unit Consistency

The `L_T` parameter is added to the electric field **after** it has been scaled by the factor:
```
tem = 0.5 * dt / rqm
```

where:
- `dt` is the simulation timestep
- `rqm` is the charge-to-mass ratio

Therefore, `L_T` should have units consistent with `E * dt / rqm`, where:
- `E` is the electric field
- This means `L_T` has units of `[velocity]` in normalized OSIRIS units

### Physical Interpretation

**Important Physical Consideration:** This implementation adds the same scalar value to all three electric field components (x, y, z). 

- If your physics requires directional dependence (e.g., only affecting one direction), you may need to further modify the implementation
- For anisotropic effects, consider whether `L_T` should be a vector rather than a scalar
- Verify that this formulation matches your intended physics model

## Example Input File

Here's a complete example of a species section with the `L_T` parameter:

```fortran
species
{
  name = "electrons"
  
  ! Basic parameters
  rqm = -1.0
  num_par_x(1:2) = 8, 8
  
  ! Pusher settings
  push_type = "standard"  ! or "simd", "vay", etc.
  
  ! Temperature gradient scale length
  L_T = 0.05
  
  ! Density profile
  density
  {
    profile_type = "uniform"
    density = 1.0
  }
  
  ! Temperature
  udist
  {
    ufl(1:3) = 0.0, 0.0, 0.0
    uth(1:3) = 0.1, 0.1, 0.1
  }
}
```

## Testing Your Implementation

### 1. Compile OSIRIS

After applying these changes, recompile OSIRIS:

```bash
./configure -s [your_system] -d [dimensions]
cd source
make clean
make
```

### 2. Run a Test Simulation

Start with a simple test case:
- Use `L_T = 0.0` and verify the simulation runs as before
- Set `L_T` to a small non-zero value and observe the effects
- Check that particle trajectories and fields behave as expected

### 3. Verify Physical Behavior

- Monitor particle energies and trajectories
- Check that the electric field modification produces expected results
- Compare with analytical predictions if available

## Troubleshooting

### Common Issues

1. **Compilation Errors**
   - Ensure all three files were modified correctly
   - Check that the Fortran syntax is correct (no missing commas in namelists)

2. **Input File Errors**
   - Verify `L_T` is spelled correctly (case-sensitive)
   - Ensure the value is a valid real number
   - Check that it's in the correct namelist section (`nl_species`)

3. **Unexpected Physical Behavior**
   - Verify the units of `L_T` are correct for your normalization
   - Check if the isotropic application (same value to all components) is appropriate for your physics
   - Consider starting with small values of `L_T` to test

## Advanced Considerations

### Modifying Other Pushers

This implementation only modifies the `dudt_boris` pusher. Other pushers available in OSIRIS include:
- `dudt_vay`
- `dudt_cond_vay`
- `dudt_cary`
- `dudt_euler`
- `dudt_fullrot`

If you need `L_T` functionality with other pushers, you would need to make similar modifications to those subroutines.

### Directional L_T

If you need different `L_T` values for different directions, you could:
1. Change `L_T` from a scalar to a vector (array of 3 components)
2. Modify the namelist to accept `L_T(1:3)`
3. Update the electric field modification to use the corresponding component

Example modification:
```fortran
! In os-spec-define.f03:
real(p_k_part), dimension(3) :: L_T = 0.0_p_k_part

! In os-spec-push.f03:
ep(1,i) = ep(1,i) * tem + this%L_T(1)
ep(2,i) = ep(2,i) * tem + this%L_T(2)
ep(3,i) = ep(3,i) * tem + this%L_T(3)
```

## References

For more information about OSIRIS:
- OSIRIS documentation
- Boris pusher algorithm
- Plasma simulation normalization conventions

## Questions or Issues?

If you encounter any problems or have questions about this implementation:
1. Check that your input file syntax is correct
2. Verify the compilation was successful
3. Review the physical interpretation and units
4. Consider whether the isotropic application is appropriate for your case
