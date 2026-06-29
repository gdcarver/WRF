# Draft GitHub Issue: BEP/BEP_BEM OpenMP one-time initialization race can cause floating-point crash in `module_sf_bep`

## Suggested title

OpenMP race in BEP/BEP_BEM one-time initialization can cause floating-point exception in `module_sf_bep:view_factors`

## Repository / version

- WRF version: `v4.6.1`
- Configuration: `wrf.exe` built with `smp` only (OpenMP, no MPI)

## Summary

We have been investigating intermittent OpenMP-only crashes in BEP urban physics. The failure is nondeterministic, becomes more likely with more OpenMP threads, and becomes easier to reproduce when the machine is busy. Re-running the same forecast often succeeds.

The traced failure is a floating-point exception in `module_sf_bep:view_factors`, and code inspection suggests that the root cause is an OpenMP race in the one-time initialization guarded by `if (first)` in both `module_sf_bep.F` and `module_sf_bep_bem.F`.

## Observed traceback

```text
#0  0x598d38b8a1e1 in ???
#1  0x598d38b897e5 in ???
#2  0xe85ce84532f in ???
#3  0x598d36f4e38f in __module_sf_bep_MOD_view_factors
	at /home/ubuntu/wrf/wrf-4.6.1/phys/module_sf_bep.f90:2844
#4  0x598d36f4f972 in __module_sf_bep_MOD_icbep_xy
	at /home/ubuntu/wrf/wrf-4.6.1/phys/module_sf_bep.f90:3369
#5  0x598d36f608bc in __module_sf_bep_MOD_bep
	at /home/ubuntu/wrf/wrf-4.6.1/phys/module_sf_bep.f90:408
#6  0x598d3878f4f2 in __module_sf_noahdrv_MOD_lsm
	at /home/ubuntu/wrf/wrf-4.6.1/phys/module_sf_noahdrv.f90:1499
#7  0x598d37ca1765 in __module_surface_driver_MOD_surface_driver._omp_fn.8
	at /home/ubuntu/wrf/wrf-4.6.1/phys/module_surface_driver.f90:2778
#8  0x598d38bd28ed in ???
#9  0xe85ce89caa3 in ???
#10 0xe85ce929c6b in ???
#11 0xffffffffffffffff in ???
```

## Why this looks like an OpenMP initialization race

`surface_driver` calls the Noah LSM path inside an OpenMP-parallel tile loop. That path calls `BEP`, which contains saved shared state and a one-time initialization block:

- `logical first`
- `data first/.true./`
- `save first,time_bep`
- many saved shared arrays populated by `init_para(...)` and `icBEP(...)`

The initialization is currently entered through an unguarded:

```fortran
if (first) then
   call init_para(...)
   call icBEP(...)
   first = .false.
endif
```

Because `first` and the initialized arrays are shared state, multiple OpenMP threads can enter this block concurrently on the first call. That allows one thread to observe partially initialized geometry arrays while another thread is still resetting/filling them.

The observed floating-point crash occurs later in `view_factors`, where denominators depend on initialized geometry (`z_u`, `nz_u`, etc.). With valid initialization, those denominators should not be zero.

## Candidate minimal fix

Make the one-time initialization thread-safe with a named OpenMP critical section and an inner recheck:

```fortran
if (first) then
!$OMP CRITICAL(BEP_INIT)
   if (first) then
      call init_para(...)
      call icBEP(...)
      first = .false.
   endif
!$OMP END CRITICAL(BEP_INIT)
endif
```

Apply the same pattern to `module_sf_bep_bem.F` with a separate critical-section name, for example `BEP_BEM_INIT`.

## Local branch with proposed fix

We have prepared a minimal patch on branch:

`bugfix-bep-init-threadsafe`

This branch contains two commits:

- `63b06518` `Codex: make BEP one-time init thread-safe`
- `f03fc7e6` `Codex: make BEP_BEM one-time init thread-safe`

## What we checked already

- Searched upstream issue titles and release notes for an existing `module_sf_bep` / `view_factors` / BEP floating-point fix and did not find a matching report.
- Diffed `v4.6.1..v4.7.1` and found no changes to `phys/module_sf_bep.F`.
- Checked later release notes and did not find a BEP-side fix corresponding to this crash path.

## Request

Please review whether the `if (first)` / `SAVE` initialization blocks in `module_sf_bep.F` and `module_sf_bep_bem.F` should be made thread-safe upstream for OpenMP builds.
