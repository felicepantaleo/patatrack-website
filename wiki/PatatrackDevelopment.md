---
title: "Development for Patatrack"
author: "Andrea Bocci"
layout: wiki
resource: true
categories: wiki
activity:  instructions
---

## "Patatrack" CMSSW releases

### `CMSSW_11_2_X_Patatrack` stable releases

The Patatrack stable branch is based on `CMSSW_11_2_X`, and
supports CUDA 11.1.x and GCC 9.3.x.

`CMSSW_11_2_1_Patatrack` is available for the architectures:

  - `slc7_amd64_gcc900` (Intel/AMD, CentOS 7, GCC 9)
  - `slc7_aarch64_gcc9` (ARM, CentOS 7, GCC 9)
  - `slc7_ppc64le_gcc9` (Power,  CentOS 7, GCC 9)
  - `cc8_amd64_gcc9` (Intel/AMD, CentOS 8, GCC 9)


### `CMSSW_11_3_X_Patatrack` developments branch

The Patatrack development branch is based on `CMSSW_11_3_X`, and
supports CUDA 11.2 and GCC 9 and later.
There is a known problem using CUDA 11 with GCC 10 in C++17 mode;
NVIDIA is aware and working on a fix.

There are no Patatrack releases in the `CMSSW_11_3_X` series, as
the focus is now on the integration into the official CMSSW.
The `CMSSW_11_3_X_Patatrack` developments branch can be used on
top of the current 11.3.x prerelease, `CMSSW_11_3_0_pre3`.


### Installation area

The `CMSSW_11_1_X_Patatrack` and later releases are available on CVMFS, along
with the standard CMSSW releases:
```bash
export SCRAM_ARCH=slc7_amd64_gcc900
source /cvmfs/cms.cern.ch/cmsset_default.sh
scram list CMSSW_11_2_1_Patatrack

Listing installed projects available for platform >> slc7_amd64_gcc900 <<

--------------------------------------------------------------------------------
| Project Name  | Project Version          | Project Location                  |
--------------------------------------------------------------------------------

  CMSSW           CMSSW_11_2_1_Patatrack
                                         --> /cvmfs/cms.cern.ch/slc7_amd64_gcc900/cms/cmssw/CMSSW_11_2_1_Patatrack
```

## Create a working area for a Patatrack 11.2.x or later release

```bash
# set up the CMSSW environment
export SCRAM_ARCH=slc7_amd64_gcc900
source /cvmfs/cms.cern.ch/cmsset_default.sh
scram list CMSSW_11_2_1_Patatrack

# create a working area
cmsrel CMSSW_11_2_1_Patatrack
cd CMSSW_11_2_1_Patatrack/src

# load the environment
cmsenv

# set up the local git repository
git cms-init -x cms-patatrack
git branch CMSSW_11_2_X_Patatrack --track cms-patatrack/CMSSW_11_2_X_Patatrack
```

You should be able to work in the `from-CMSSW_11_2_1_Patatrack` branch as you
would in a normal CMSSW development area.

However, it is recommended to always use the `HEAD` of the chosen branch:
```bash
git checkout CMSSW_11_2_X_Patatrack

# check out the modified packages and their dependencies
git diff CMSSW_11_2_1_Patatrack --name-only --no-renames | cut -d/ -f-2 | uniq | xargs -r git cms-addpkg
git cms-checkdeps -a

scram b -j
```

If one of the local packages causes CUDA-related problems at runtime, it may be
necessary to rebuild all CUDA packages with:
```
cmsCudaRebuild.sh
```

## Working with older GPUs
CUDA is configured in CMSSW to support GPUs with Pascal (e.g. GeForce GTX 1080,
Tesla P100, ...), Volta (e.g. Titan V, Tesla V100), and Turing (e.g. RTX 2080,
Tesla T4, ...) architectures.
To work with GPUs based on older architectures, you need to reconfigure CUDA and
rebuild all CUDA code in CMSSW with:
```bash
cmsenv
cmsCudaSetup.sh
cmsCudaRebuild.sh
```


## Developing with the Patatrack branch
To work on further developments, it is recommended to start from the HEAD of the
`CMSSW_11_3_X_Patatrack` branch.

### Checkout the HEAD of the development branch

```bash
# set up the CMSSW environment
export SCRAM_ARCH=slc7_amd64_gcc900
source /cvmfs/cms.cern.ch/cmsset_default.sh

# create a working area
cmsrel CMSSW_11_3_0_pre3
cd CMSSW_11_3_0_pre3/src

# load the environment
cmsenv

# set up the local git repository
git cms-init -x cms-patatrack

# create a local development branch
git checkout cms-patatrack/CMSSW_11_3_X_Patatrack -b my_development_branch

# check out the modified packages and their dependencies
git diff $CMSSW_VERSION --name-only --no-renames | cut -d/ -f-2 | sort -u | xargs -r git cms-addpkg
git cms-checkdeps -a

# optionally, rebuild all CUDA code in the release
cmsCudaRebuild.sh
```


### Write code, compile, debug, commit, and push to your repository
```bash
...
scram b -j
...
git add ...
git commit
git push -u my-cmssw HEAD:my_development_branch
```


## Create a pull request

  - before your PR can be submitted, you should run a couple of checks to make
    the integration process much smoother:
    ```bash
    # recompile with debug information (for host code) and line-number information (for device code)
    scram b clean
    USER_CXXFLAGS="-g" USER_CUDA_FLAGS="-g -lineinfo" scram b -j

    # run your code under cuda-memcheck
    cuda-memcheck --tool initcheck --print-limit 1 cmsRun step3.py
    cuda-memcheck --tool memcheck  --print-limit 1 cmsRun step3.py
    cuda-memcheck --tool synccheck --print-limit 1 cmsRun step3.py
    ```

  - it is also possible to run more thorough, semi-automatic checks: see [Running the validation](PatatrackValidation.md)

  - open https://github.com/cms-patatrack/cmssw/

  - there should be box with the branch you just created and a green button
    saying "Compare & pull request":
    ![Compare & pull request](screenshot1.png "Compare & pull request")

  - click on it, and create a pull request as usual:
    ![Create a pull request](screenshot2.png "Create a request")

  - make sure to choose `CMSSW_11_3_X_Patatrack` as the target branch, **not**
    the `master` branch

