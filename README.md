
### Log in & DUNE FNAL machines (dunegpvm*) environment setup:
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
# 1. Produce Ntuple from DUNE FD MC files 

The following instruction is used to produce ROOT n-tuples from FD MC CAF files (mcc11): [FD Beamsim Requests](https://dune-data.fnal.gov/mc/mcc11/index.html). 

Output: " $\color{#FF0000}{myntuple.root}$ ". 

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

### Enviroment setup when loging back

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


# 2. Run translation and rotations on FD n-tuples

 The produced FD n-tuples  will be used as input files for the following program to run.  
 Input: " $\color{#FF0000}{myntuple.root}$"
 Output: " $\color{#FF0000}{Output _ FDGeoEff.root}$ "    .

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
./runGeoEffFDEvtSim /dune/app/users/janelly/FDEff/srcs/myntuples/myntuples/MyEnergyAnalysis/myntuple.root
```
This will produce a root output file containing throws and the hadron throw result. ($\color{#FF0000}{Output~file}$ " $\color{#FF0000}{Output _ FDGeoEff.root}$ ")

If the source files in src are changed, recompile:

```
source setup.sh
cmake -DPYTHON_EXECUTABLE:FILEPATH=`which python` .
make -j geoEff    
```
## Calculate FD event efficiency 

The output root file from running ```runGeoEffFDEvtSim``` can be used to calculate FD event hadron containment efficiency by running:

```
cd /dune/app/users/weishi/NDEff/DUNE_ND_GeoEff
source setup.sh
cd app
root -l -b -q FDEffCalc.C
# 5k evts: 10mins
```

The output root file from running ```runGeoEffFDEvtSim``` can also be used for lepton NN training.

## Event displays (This is actually the Rea ) 

The hadronic hits can be plotted in 2D event displays at FD via:
```
echo 'gROOT->ProcessLine(".L ReadHadronHitNtuple.cpp"); ReadHadronHitNtuple_FD()'| root -l -b
```

## Run on Grid

First get the work env setup:
```
cd /dune/app/users/weishi
wget https://raw.githubusercontent.com/weishi10141993/NeutrinoPhysics/main/setupNDEff-grid.sh --no-check-certificate
```

Suppose the input FD ntuples are in this directory,
```
/pnfs/dune/scratch/users/weishi/myFDntuples
```
write the list to txt file,
```
ls -d "/pnfs/dune/scratch/users/weishi/myFDntuples"/* | sed "s\/pnfs\root://fndca1.fnal.gov:1094/pnfs/fnal.gov/usr\g" > myFDntuples.txt
# it also changes pnfs to xrootd so that worker node can access
```
which is going to be sent to the grid in a tarball below.

Now make the tarball,
```
tar -czvf NDEff.tar.gz setupNDEff-grid.sh myFDntuples.txt
# Check the tarball *.tar.gz is indeed created and open with: tar -xf *.tar.gz
```

Get the running script,
```
wget https://raw.githubusercontent.com/weishi10141993/NeutrinoPhysics/main/run_NDEff_autogrid.sh --no-check-certificate

# this submits N jobs (N = number of input files, so each job runs 1 file)
jobsub_submit -G dune -N 1 --memory=500MB --disk=1GB --expected-lifetime=20m --cpu=1 --resource-provides=usage_model=DEDICATED,OPPORTUNISTIC,OFFSITE --tar_file_name=dropbox:///dune/app/users/weishi/NDEff.tar.gz --use-cvmfs-dropbox -l '+SingularityImage=\"/cvmfs/singularity.opensciencegrid.org/fermilab/fnal-wn-sl7:latest\"' --append_condor_requirements='(TARGET.HAS_Singularity==true&&TARGET.HAS_CVMFS_dune_opensciencegrid_org==true&&TARGET.HAS_CVMFS_larsoft_opensciencegrid_org==true&&TARGET.CVMFS_dune_opensciencegrid_org_REVISION>=1105&&TARGET.HAS_CVMFS_fifeuser1_opensciencegrid_org==true&&TARGET.HAS_CVMFS_fifeuser2_opensciencegrid_org==true&&TARGET.HAS_CVMFS_fifeuser3_opensciencegrid_org==true&&TARGET.HAS_CVMFS_fifeuser4_opensciencegrid_org==true)' file:///dune/app/users/weishi/run_NDEff_autogrid.sh

```
Reference:
3 files each 100 events:``` -N 3 --memory=500MB --disk=1GB --expected-lifetime=20m --cpu=1```
