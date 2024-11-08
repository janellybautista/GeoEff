# Produce Ntuple from DUNE FD MC files 

The following instruction is used to produce ROOT n-tuples from FD MC CAF files (mcc11): [FD Beamsim Requests](https://dune-data.fnal.gov/mc/mcc11/index.html). 

Output: " $\color{#FF0000}{myntuple.root}$ ". 

## 1. Log in & DUNE FNAL machines (dunegpvm*) environment setup:
```
kfnal                                      # Short for kinit -f <username>@FNAL.GOV. 
ssh -X janelly@dunegpvm01.fnal.gov      
exit                                       # Quit FNAL
```
or to directly access the DUNE FNAL machine through the vnc server, use
```
ssh -Y -L 5901:localhost:5901 janelly@dunegpvm01.fnal.gov
```
 [DUNE: SL7 to ALMA9](https://wiki.dunescience.org/wiki/SL7_to_Alma9_conversion):
```
/cvmfs/oasis.opensciencegrid.org/mis/apptainer/current/bin/apptainer shell --shell=/bin/bash \
-B /cvmfs,/exp,/nashome,/pnfs/dune,/opt,/run/user,/etc/hostname,/etc/hosts,/etc/krb5.conf --ipc --pid \
/cvmfs/singularity.opensciencegrid.org/fermilab/fnal-dev-sl7:latest
```

### First time environment setup only



```
cd /dune/app/users/$USER                                               
mkdir FDEff (first time only)
cd FDEff

source /cvmfs/dune.opensciencegrid.org/products/dune/setup_dune.sh
setup dunetpc v09_22_02 -q e19:debug
[optional if run interactively]:  setup_fnal_security                     # A FNAL grid proxy to submit jobs and access data in dCache via xrootd or ifdh.

mrb newDev
# The prompt ask you to run this:
source /dune/app/users/<your_username>/inspect/localProducts_larsoft_${LARSOFT_VERSION}_debug_${COMPILER}/setup
# For example, mine is: source /dune/app/users/janelly/FDEff/localProducts_larsoft_v09_22_02_debug_e19/setup

cd srcs
git clone https://github.com/weishi10141993/myntuples.git               # First time only, checkout the analysis code from GitHub

mrb uc                                                                  # Tell mrb to update CMakeLists.txt with the latest version numbers of the products.
cd ${MRB_BUILDDIR}                                                      # Go to your build directory
mrb z
mrbsetenv                                                               # Create the bookkeeping files needed to compile programs.
mrb b                                                                   # Compile the code in ${MRB_SOURCE}
```

Produce a ROOT nTuple by running on DUNE FD MC files ($\color{#FF0000}{Output~file}$ : $\color{#FF0000}{myntuple.root}$) 
```
cd /exp/dune/app/users/$USER/FDEff/srcs/myntuples/myntuples/MyEnergyAnalysis
lar -c MyEnergyAnalysis.fcl -n -1                                       # Obtain myntuple.root
# 10k evts take about 32 minutes
```

## 2. Enviroment setup when loging back

```
source /cvmfs/dune.opensciencegrid.org/products/dune/setup_dune.sh
setup dunetpc v09_22_02 -q e19:debug
source /dune/app/users/$USER/FDEff/localProducts_larsoft_v09_22_02_debug_e19/setup
mrbsetenv
cd /dune/app/users/$USER/FDEff/srcs/myntuples/myntuples/MyEnergyAnalysis
```

If added new package in ```srcs``` directory, do ```mrb uc``` and then recompile as above.

To commit changed code changes to remote repository:

```
git commit
git push
```


# Instruction for running translation and rotations on FD n-tuples

Prerequisite: [Produce Ntuple from DUNE FD MC files](https://github.com/weishi10141993/myntuples#produce-ntuple-from-dune-fd-mc-files). The produced FD n-tuples will be used as input files for the following program to run.

[First time only]
```
cd /dune/app/users/<your_username>
mkdir NDEff (first time only)
cd NDEff
git clone --recurse-submodules -b FD_Wei https://github.com/weishi10141993/DUNE_ND_GeoEff.git      # Get geoEff library
# Note for git version (git --version) before 2.13, use: git clone --recursive -b FD_Wei https://github.com/weishi10141993/DUNE_ND_GeoEff.git
cd DUNE_ND_GeoEff
source setup.sh                                                                                    # Necessary setups for build
cmake -DPYTHON_EXECUTABLE:FILEPATH=`which python` .
make -j geoEff                                                                                     # Build geoEff (can also use: make -j pyGeoEff)
```

To (re)compile
```
cd /dune/app/users/<your_username>/NDEff/DUNE_ND_GeoEff/
#
# In case you log out, need to source setup.sh to setup ROOT
#
source setup.sh         

# Compile program
cd app
make runGeoEffFDEvtSim                                                                       
```

To (re)run program,
```
cd ../bin
# Usage: ./runGeoEffFDEvtSim inputFDntuple
./runGeoEffFDEvtSim /dune/app/users/weishi/FDEff/srcs/myntuples/myntuples/MyEnergyAnalysis/myntuple.root
```
this will produce a root file containing throws and the hadron throw result.

If the source files in src are changed, recompile:

```
source setup.sh
cmake -DPYTHON_EXECUTABLE:FILEPATH=`which python` .
make -j geoEff    
```
