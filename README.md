# C3D.jl

[![version](https://juliahub.com/docs/C3D/version.svg)](https://juliahub.com/ui/Packages/C3D/Y0cAa)
[![pkgeval](https://juliahub.com/docs/C3D/pkgeval.svg)](https://juliahub.com/ui/Packages/C3D/Y0cAa)
[![CI](https://github.com/halleysfifthinc/C3D.jl/actions/workflows/CI.yml/badge.svg)](https://github.com/halleysfifthinc/C3D.jl/actions/workflows/CI.yml)
[![codecov](https://codecov.io/gh/halleysfifthinc/C3D.jl/branch/master/graph/badge.svg)](https://codecov.io/gh/halleysfifthinc/C3D.jl)
[![Project Status: Active – The project has reached a stable, usable state and is being actively developed.](https://www.repostatus.org/badges/latest/active.svg)](https://www.repostatus.org/#active)

C3D is a common file format for motion capture and other biomechanics related measurement systems (force plate data, EMG, etc). The goal of this package is to completely implement the [C3D file spec](https://www.c3d.org), and be compatible with files from major C3D producing programs (Vicon Nexus, etc.) where they might differ from or extend the C3D file spec.

Current test data is gathered from sample data found on the [C3D website](https://www.c3d.org/sampledata.html).
Pull requests welcome! Please open an issue if you have a file that is not being read correctly.

## Usage

### Reading data

Marker and analog data are accessed through the `point` and `analog` fields. Note that all data is converted to Float32 upon reading, regardless of the original type (eg DEC types).

```julia
julia> # The artifacts with the test data can only be used from the `C3D.jl` directory when `LazyArtifacts` has been loaded

julia> pc_real = readc3d(artifact"sample01/Eb015pr.c3d")
C3DFile("~/.julia/artifacts/318c299a26ba07c015fa86768512b677fbb7e64c/Eb015pr.c3d")
  0:9+0 frames
  26 points @ 50 Hz; 16 analog channels @ 200 Hz

  julia> pc_real.point["LTH1"]
450×3 Array{Float32,2}:
 0.0         0.0     0.0
 0.0         0.0     0.0
 0.0         0.0     0.0
 ⋮
 1.66667  2152.67  702.917
 3.58333  2159.0   702.833
 5.0      2168.08  702.25

julia> pc_real.analog["FZ1"]
1800-element Array{Float32,1}:
 -20.832
 -21.576
 -20.832
   ⋮
 -20.088001
 -21.576
 -22.32
```

#### Point residuals, invalid and calculated points

According to the C3D format documentation, invalid data points are signified by setting the residual word to `-1.0`. This convention is respected in C3D.jl by changing the residual and coordinates of invalid points/frames to `missing`. If your C3D files do not respect this convention, or if you wish to ignore this for some other reason, this behavior can be disabled by setting keyword arg `missingpoints=false` in the `readc3d` function. Convention is to signify calculated points (e.g. filtered, interpolated, etc) by setting the residual word to `0.0`.

```julia
julia> bball = readc3d(artifact"sample16/basketball.c3d")
C3DFile("~/.julia/artifacts/042cc43a45ace35e97473c6cf0d08e25f1c73fcb/basketball.c3d")
  0:1+9 frames
  22 points @ 25 Hz

julia> bball.point["2003"]
34×3 Array{Union{Missing, Float32},2}:
 missing  missing  missing
 missing  missing  missing
 missing  missing  missing
  ⋮

julia> bball = readc3d("data/sample16/basketball.c3d"; missingpoints=false)
C3DFile("~/.julia/artifacts/042cc43a45ace35e97473c6cf0d08e25f1c73fcb/basketball.c3d")
  0:1+9 frames
  22 points @ 25 Hz

julia> bball.point["2003"]
34×3 Array{Union{Missing, Float32},2}:
  0.69115      0.987054    1.53009
  0.656669     1.00666     1.5854
  0.615803     1.02481     1.60467
   ⋮
```

Point residuals can be accessed using the `residual` field which is indexed by marker label.

```julia
julia> pc_real.residual["RFT2"]
450-element Array{Union{Missing, Float32},1}:
 10.333334f0
 10.333334f0
  9.666667f0
  ⋮
  2.0f0
  2.0f0
  2.0f0
```

### Accessing C3D parameters

The parameters can be accessed through the `groups` field. Specific groups are indexed as Symbols.

```julia
julia> pc_real.groups
Dict{Symbol,C3D.Group} with 5 entries:
  :POINT          => Symbol[:DESCRIPTIONS, :RATE, :DATA_START, :FRAMES, :USED, :UNITS, :Y_SCREEN, :LABELS, :X_SCREEN, :SCALE]
  :ANALOG         => Symbol[:DESCRIPTIONS, :RATE, :GEN_SCALE, :OFFSET, :USED, :UNITS, :LABELS, :SCALE]
  :FORCE_PLATFORM => Symbol[:TYPE, :ORIGIN, :ZERO, :TRANSLATION, :CORNERS, :USED, :ROTATION, :CHANNEL]
  :SUBJECT        => Symbol[:WEIGHT, :NUMBER, :HEIGHT, :DATE_OF_BIRTH, :GENDER, :PROJECT, :TARGET_RADIUS, :NAME]
  :FPLOC          => Symbol[:INT, :OBJ, :MAX]

julia> pc_real.groups[:POINT]
Symbol[:DESCRIPTIONS, :RATE, :DATA_START, :FRAMES, :USED, :UNITS, :Y_SCREEN, :LABELS, :X_SCREEN, :SCALE]
```

Parameter values can be accessed like this:

```julia
julia> pc_real.groups[:POINT][:USED]
26

julia> pc_real.groups[:POINT][:LABELS]
48-element Array{String,1}:
 "RFT1"
 "RFT2"
 "RFT3"
 ⋮
 ""
 ""
 ""

julia> # Or, if you know the type (and you need the type-stability)

julia> pc_real.groups[:POINT][Int, :USED]
26

```

# Advanced: Debugging

There are two main steps to reading a C3D file: reading the parameters, and reading the point and/or analog data. In the event a file read fails, the stacktrace will show whether the error happened in `_readparams` or `readdata`. If the error occurred in `readdata`, try only reading the parameters, optionally setting the keyword argument `validate` to `false`:

```julia
julia> pc_real = readc3d("data/sample01/Eb015pr.c3d"; paramsonly=true)
Dict{Symbol,C3D.Group} with 5 entries:
  :POINT          => Symbol[:DESCRIPTIONS, :RATE, :DATA_START, :FRAMES, :USED, :UNITS, :Y_SCREEN, :LABELS, :X_SCREEN, :SCALE]
  :ANALOG         => Symbol[:DESCRIPTIONS, :RATE, :GEN_SCALE, :OFFSET, :USED, :UNITS, :LABELS, :SCALE]
  :FORCE_PLATFORM => Symbol[:TYPE, :ORIGIN, :ZERO, :TRANSLATION, :CORNERS, :USED, :ROTATION, :CHANNEL]
  :SUBJECT        => Symbol[:WEIGHT, :NUMBER, :HEIGHT, :DATE_OF_BIRTH, :GENDER, :PROJECT, :TARGET_RADIUS, :NAME]
  :FPLOC          => Symbol[:INT, :OBJ, :MAX]

julia> pc_real = readc3d("data/sample01/Eb015pr.c3d"; paramsonly=true, validate=false)
Dict{Symbol,C3D.Group} with 5 entries:
  :POINT          => Symbol[:DESCRIPTIONS, :RATE, :DATA_START, :FRAMES, :USED, :UNITS, :Y_SCREEN, :LABELS, :X_SCREEN, :SCALE]
  :ANALOG         => Symbol[:DESCRIPTIONS, :RATE, :GEN_SCALE, :OFFSET, :USED, :UNITS, :LABELS, :SCALE]
  :FORCE_PLATFORM => Symbol[:TYPE, :ORIGIN, :ZERO, :TRANSLATION, :CORNERS, :USED, :ROTATION, :CHANNEL]
  :SUBJECT        => Symbol[:WEIGHT, :NUMBER, :HEIGHT, :DATE_OF_BIRTH, :GENDER, :PROJECT, :TARGET_RADIUS, :NAME]
  :FPLOC          => Symbol[:INT, :OBJ, :MAX]
```

If the error occurred in `readdata`, it is likely that there is an incorrect setting in one of the parameters. (If this is consistent among several files from the same vendor, open an issue and send an example file so I can fix whatever is causing the problem.)

If the error occurred in `_readparams`, try starting julia with `$ JULIA_DEBUG=C3D julia`. This will enable debug messages that may help narrow down the parameter causing the problem.

Please open an issue if you have a file that is being read incorrectly.

## Roadmap

I plan to eventually add support for saving files that have been modified and for creating new files, but this is not a use case that I require currently or in the foreseeable future. If this is important to you, open an issue or submit a PR!
