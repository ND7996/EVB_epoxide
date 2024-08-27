# EMPIRICAL VALENCE BOND SIMULATION WITH GROMACS

- [EMPIRICAL VALENCE BOND SIMULATION WITH GROMACS](#empirical-valence-bond-simulation-with-gromacs)
  - [DOWNLOAD AND INSTALLATION OF RELEVANT SOFTWARES](#download-and-installation-of-relevant-softwares)
    - [AVOGADRO SOFTWARE](#avogadro-software)
    - [GAUSSIAN INSTALLATION GUIDE](#gaussian-installation-guide)
    - [GROMACS INSTALLATION GUIDE](#gromacs-installation-guide)
    - [MAESTRO INSTALLATION](#maestro-installation)
    - [AMBERTOOLS INSTALLATION GUIDE (AMBER22 on Ubuntu 22.10)](#ambertools-installation-guide-amber22-on-ubuntu-2210)
  - [THE WORK FLOW](#the-work-flow)


## DOWNLOAD AND INSTALLATION OF RELEVANT SOFTWARES

CAUTION!!!! THE FOLLOWING INSTALLATION GUIDES (AMBER, GROMACS, GAUSSIAN) ARE DESIGNED FOR LINUX SUB-SYSTEMS

### AVOGADRO SOFTWARE
The software is open sourced and can be downloaded from "https://sourceforge.net/projects/avogadro/". Accept the necessary privacy regulations in order to install the software.   

### GAUSSIAN INSTALLATION GUIDE

Gaussian is not open source. You can easily have access to it through a supercomputing center or through research persons who have a licence. 
In case it is a module on your supercomputing cluster, activate Gaussian by writing  "module load Gaussian"

### GROMACS INSTALLATION GUIDE
-Download GROMACS at: https://ftp.gromacs.org/gromacs/gromacs-2023.tar.gz
```
tar xfz gromacs-2023(3).tar.gz
cd gromacs-2023
mkdir build
cd build
cmake .. -DGMX_BUILD_OWN_FFTW=ON -DREGRESSIONTEST_DOWNLOAD=ON
make
make check
sudo make install
source /usr/local/gromacs/bin/GMXRC
```

### MAESTRO INSTALLATION


### AMBERTOOLS INSTALLATION GUIDE (AMBER22 on Ubuntu 22.10)
-If you do not have conda, download it by visiting:  https://docs.conda.io/en/latest/miniconda.html
-Install cmake and other packages if you do not have. See more at: https://cmake.org/download
```
sudo su
apt update
apt upgrade
apt install cmake
apt -y update
apt -y install tcsh make
apt install gcc
apt -y install gcc gfortran
apt -y install flex bison patch
apt -y install bc xorg-dev libbz2-dev wget
apt-get install gcc build-essential
```
- Download Ambertools (AmberTools22.tar.bz2 and Amber22.tar.bz) from https://ambermd.org/GetAmber.php into the amber directory
- Extract the files 
```
tar xvf  AmberTools22.tar.bz2
tar xvf Amber22.tar.bz2
- Compile and install Amber
- cd/amber/amber22_src/build
- ./run_cmake
```
If this does not work, install a few other necessary packages as follows.
```
sudo apt install pip
pip install numpy
python3 -m pip install scipy
pip install Cython
pip install setuptools
python3 setup.py install
pip install tk
python3 -m pip install -U pip
pip install matplotlib
sudo apt install python3-tk  
```

## THE WORK FLOW

1. Preparation of the system and building topologies
Calculate the relaxed geometries and ESP charges with Gaussian16 for the reacting moieties which include; cis stilbene oxide substrate, water molecule and Aspartate (Asp101) in reactant state (RS). That for the product state (PS) were equally  calculated; RRDiol or SSDiol, Deprotonated Aspartate (ASP101) and Protonated Aspartate(ASP132). In the case of reactants, We will obtain the following files: STO.log (substrate), H2O.log (water) and ASH.log (ASH101); and for PS we will have: RRD.log or SSD.log(substrate), ASP.log (ASP101), and ASH.log (ASH132). The commands used for the various files to get the .log and ESP charges was:
```
g16 input.com  (g16 STO.com)
g16 input.com  (g16 STO_gesp.com)
```
NOTE: The molecules were drawn on Avogadro so as to obatin their coordinates which will serve as an input for the gaussian calculation. 

2. Calculate the RESP charges with antechamber from AmberTools. The command below should be repeated for each of the output files from the Gaussian calculation. 
```
antechamber -fi gout -i STO_gesp.log -fo ac -o STO.ac -c resp -nc -1
```

3. Extract the pdb from the gaussian output file for all the reactants and products.
```
antechamber -fi gout -i STO_gesp.log -fo pdb -o STO.pdb -rn STO
```

4. Use the extracted pdb files to calculate the ffld_server parameters using maestro (get the .mae files and pass through the command below).
``` 
$PATH(ffld_server.exe) -imae STO.mae -version 14  -print_parameters -out_file STO.ffld
```

5. Convert the ffld_server parameters into GROMACS' OPLS-AA force field format using the python code (ffld2gmx.py). For cis-Stilbene oxide substrate we will obtain the following files: sto_atomtypes.opls, sto_vdw.opls, sto_bonds.opls, sto_angles.opls, sto_torsions.opls, and sto_impropers.opls.
```
ffld2gmx.py -n STO -f STO.ffld -a STO.ac
```

6. Add the content from sto_atomtypes.opls to atomtypes.atp, add 3ov_vdw.opls to ffnonbonded.itp, and 3ov_bonds.opls, sto_angles.opls, sto_torsions.opls, and sto_impropers.opls to ffbonded.itp file of GROMACS' force field. (For convenience, you can write only the parameters for RS in ffbonded.itp file and substitute the bonding types for the PS atom types inside ffnonbonded.itp with the corresponding bonding types for RS). In these files, the representation of H2O is changed to WHO.

7. Add dummy atom types for all EVB atoms in ffnonbonded.itp and atomtypes.atp files (see the files at the address indicated above)

8. Build the corresponding residue inside aminoacids.rtp file (see STO residue inside aminoacids.rtp); we must build only the residues corresponding to RS.
9. Add the content from `sto_types.opls` to `atomtypes.atp`, add `sto_vdw.opls` to `ffnonbonded.itp`, and `sto_bonds.opls`, `sto_angles.opls`, `sto_torsions.opls`, and `sto_impropers.opls` to `ffbonded.itp` file of GROMACS' force field (The parameters added to the ffbonded.itp is done only for the reactant state). 
10. Add dummy atom types for all EVB atoms in ffnonbonded.itp and atomtypes.atp for the reactant state.
11. Build the corresponding residue inside `aminoacids.rtp` file for the Reactant state. Edit the aminoacids.rtp file and the aminoacid.hbd file so that the residues of the protonated Aspartate and the protonated Histidine are represented in a similar manner.
12. Build GROMACS topology using the command;
```
gmx pdb2gmx -f STBO.pdb -o STBO-start.pdb -water spc -merge all
```
13. Build the periodic box using the command;
```
gmx editconf -f STBO-start.pdb -o STBO-box.pdb -c -d 2 -bt dodecahedron
```
14. Solvate the system.
```
gmx solvate -cp STBO-box.pdb -cs spc216.gro -o STBO-solv.pdb -p topol.top
```
15. Add ions
    Check the topol.top file and if some of the imprompers angles of STO are missing, copy them from the ffbonded.itp into the topol.top file.Then proceed with the following commands. 
```
echo > dummy.mdp
gmx grompp -f dummy.mdp -o dummy.tpr -p topol.top -c STBO-solv.pdb -maxwarn 1
gmx genion -s dummy.tpr -o STBO_ion.pdb -p topol.top -neutral
choose Group 16 (SOL)
```
17.   The qmatoms.dat file shhould be created. It should contain the various qatoms in the system, the moorse paramters and softcore potentials
18.   Topology files (51) for the various frames are created using the code gmx4evb.py  and the command below:
```
python gmx4evb.py -f 51 -r STO ASH -p RRD ASP
```
19. Successive Equilibration of the system is carried out using the topology file topo.000.top.
