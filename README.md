# Guide for Molecular dynamics (MD) simulations with AMBER Software on ASU Sol cluster

## Initial steps with PDB

Follow the PyMOL guide here: [PyMOLGuide](https://github.com/John-Kazan/PyMOLGuide)

## Using VPN to connect to ASU network:

Follow the VPN guide here: [VPNGuide](https://github.com/John-Kazan/VPNGuide)

## Before you continue

- Change my id to yours in each command
- If you are using a pdb file other than `1btl.pdb`, rename it to `1btl.pdb` so you dont have to change anything else in the scripts

## After logging in:

`pwd` command shows me home directory `/home/ikazan`

change directory to scratch space

```
cd /scratch/ikazan
```

create a new directory here by using

```
mkdir -pv testdir1
```

change directory to the new one

```
cd testdir1/
```

```
ls
```

the directory is empty

`pwd` shows me the current working directory

copy the sol path: `/scratch/ikazan/testdir1`

open a new ternminal tab (this will be connected to your own computer)

go to the directory where you have the pdb file and copy the file to sol

```
scp ./1btl.pdb ikazan@login.sol.rc.asu.edu:/scratch/ikazan/testdir1/
```

we are going to switch the termnial window to the sol session one

we are going to start an interactive session by running

```
interactive
```

## Paremetrize Protein with LEaP

Prepare `tleap.in` input file:

```
vim tleap.in
```

press `i` to enter edit mode

copy and paste the text below

```
source leaprc.protein.ff14SB
source leaprc.water.tip3p
pdb_file = loadpdb 1btl.pdb
solvateBox pdb_file TIP3PBOX 16.0
charge pdb_file
addions pdb_file Na+ 0
addions pdb_file Cl- 0
saveAmberParm pdb_file 1btl.parm7 1btl.crd
quit
```

press `esc` button on keyboard and then type `:wq`

run:

```
module load amber/22v3
```

and then run:

```
tleap -f tleap.in
```

## Minimize Solvent

Prepare `minimization_solvent.in` input file:

```
vim minimization_solvent.in
```

press `i` to enter edit mode

copy and paste the text below

```
# Energy minimization solvent
&cntrl
imin=1,
ntpr=10000,
ntr=1,
restraintmask='!(:WAT,Na+,Cl-)',
restraint_wt=10.0,
maxcyc=50000,
ncyc=25000,
ntc=1,
ntf=1,
ntb=1,
cut=12.0,
&end
/
```

press `esc` button on keyboard and then type `:wq`

## Two options to continue: 1) Interactive, 2) Batch (I recommend 2)

### 1) Interactive mode

make sure you have `interactive` session and run:

```
module load amber/22v3
``` 

run:

```
pmemd -O \
-i minimization_solvent.in \
-o minimization_solvent.out \
-p 1btl.parm7 \
-c 1btl.crd \
-r 1btl_minimization_solvent.rst7 \
-x 1btl_minimization_solvent.crd \
-ref 1btl.crd
```

### 2) Batch mode

Prepare `minimization_solvent_sbatch` file:

```
vim minimization_solvent_sbatch
```

press `i` to enter edit mode

copy and paste the text below

```
#!/usr/bin/env bash
#SBATCH -n 8
#SBATCH -p general
#SBATCH -t 7-00:00:00
#SBATCH -o slurm.%j.out
#SBATCH -e slurm.%j.err

module load amber/22v3

mpiexec.hydra -n 8 pmemd.MPI -O \
-i minimization_solvent.in \
-o minimization_solvent.out \
-p 1btl.parm7 \
-c 1btl.crd \
-r 1btl_minimization_solvent.rst7 \
-x 1btl_minimization_solvent.crd \
-ref 1btl.crd
```

press `esc` button on keyboard and then type `:wq`

submit the job to queue by running

```
sbatch minimization_solvent_sbatch
```

to check the status of the job run

```
squeue -u ikazan
```

## Minimize Solution

Prepare `minimization_solution.in` input file:

```
vim minimization_solution.in
```

press `i` to enter edit mode

copy and paste the text below

```
# Energy minimization solution
&cntrl
imin=1,
ntpr=10000,
maxcyc=100000,
ncyc=50000,
ntc=1,
ntf=1,
ntb=1,
cut=12.0,
&end
/
```

press `esc` button on keyboard and then type `:wq`

## Two options to continue: 1) Interactive, 2) Batch (I recommend 2)

### 1) Interactive mode

make sure you have `interactive` session and run:

```
module load amber/22v3
``` 
run:

```
pmemd -O \
-i minimization_solution.in \
-o minimization_solution.out \
-p 1btl.parm7 \
-c 1btl_minimization_solvent.rst7 \
-r 1btl_minimization_solution.rst7 \
-x 1btl_minimization_solution.crd
```

### 2) Batch mode

Prepare `minimization_solution_sbatch` file:

```
vim minimization_solution_sbatch
```

press `i` to enter edit mode

copy and paste the text below

```
#!/usr/bin/env bash
#SBATCH -n 8
#SBATCH -p general
#SBATCH -t 7-00:00:00
#SBATCH -o slurm.%j.out
#SBATCH -e slurm.%j.err

module load amber/22v3

mpiexec.hydra -n 8 pmemd.MPI -O \
-i minimization_solution.in \
-o minimization_solution.out \
-p 1btl.parm7 \
-c 1btl_minimization_solvent.rst7 \
-r 1btl_minimization_solution.rst7 \
-x 1btl_minimization_solution.crd
```

press `esc` button on keyboard and then type `:wq`

submit the job to queue by running

```
sbatch minimization_solution_sbatch
```

to check the status of the job run

```
squeue -u ikazan
```

## Heat up System

Prepare `heatup.in` input file:

```
vim heatup.in
```

press `i` to enter edit mode

copy and paste the text below

```
# Heat Up 100ps
&cntrl
imin=0,
ntx=1,
irest=0,
ntpr=5000,
ntwr=5000,
iwrap=1,
ntwx=5000,
ntr=1,
restraintmask='!(:WAT,Na+,Cl-)',
restraint_wt=10.0,
nstlim=50000,
dt=0.002,
ntt=3,
temp0=300.0,
tempi=0.0,
ig=-1,
tautp=1.0,
gamma_ln=2.0,
ntp=0,
taup=2.0,
ntc=2,
ntf=2,
ntb=1,
cut=12.0,
&end
/
```

press `esc` button on keyboard and then type `:wq`

## Two options to continue: 1) Interactive, 2) Batch (I recommend 2)

### 1) Interactive mode

make sure you have `interactive` session and run:

```
module load amber/22v3
``` 

run:

```
pmemd -O \
-i heatup.in \
-o heatup.out \
-p 1btl.parm7 \
-c 1btl_minimization_solution.rst7 \
-r 1btl_heatup.rst7 \
-x 1btl_heatup.crd \
-ref 1btl_minimization_solution.rst7
```

### 2) Batch mode

Prepare `heatup_sbatch` file:

```
vim heatup_sbatch
```

press `i` to enter edit mode

copy and paste the text below

```
#!/usr/bin/env bash
#SBATCH -n 8
#SBATCH -p general
#SBATCH -t 7-00:00:00
#SBATCH -o slurm.%j.out
#SBATCH -e slurm.%j.err

module load amber/22v3

mpiexec.hydra -n 8 pmemd.MPI -O \
-i heatup.in \
-o heatup.out \
-p 1btl.parm7 \
-c 1btl_minimization_solution.rst7 \
-r 1btl_heatup.rst7 \
-x 1btl_heatup.crd \
-ref 1btl_minimization_solution.rst7
```

press `esc` button on keyboard and then type `:wq`

submit the job to queue by running

```
sbatch heatup_sbatch
```

to check the status of the job run

```
squeue -u ikazan
```

## Production NPT (CPU)

Prepare `production_npt_cpu.in` input file:

```
vim production_npt_cpu.in
```

press `i` to enter edit mode

copy and paste the text below

```
# Production 100ps
&cntrl
imin=0,
ntx=5,
irest=1,
ntpr=5000,
ntwr=5000,
iwrap=1,
ntwx=5000,
nstlim=50000,
dt=0.002,
ntt=3,
temp0=300.0,
ig=-1,
tautp=1.0,
gamma_ln=2.0,
ntp=1,
taup=2.0,
ntc=2,
ntf=2,
ntb=2,
cut=12.0,
&end
/
```

press `esc` button on keyboard and then type `:wq`

## Two options to continue: 1) Interactive, 2) Batch (I recommend 2)

### 1) Interactive mode

make sure you have `interactive` session and run:

```
module load amber/22v3
``` 

run:

```
pmemd -O \
-i production_npt_cpu.in \
-o production_npt_cpu.out \
-p 1btl.parm7 \
-c 1btl_heatup.rst7 \
-r 1btl_production_npt_cpu.rst7 \
-x 1btl_production_npt_cpu.crd
```

### 2) Batch mode

Prepare `production_npt_cpu_sbatch`  file:

```
vim production_npt_cpu_sbatch
```

press `i` to enter edit mode

copy and paste the text below

```
#!/usr/bin/env bash
#SBATCH -n 8
#SBATCH -p general
#SBATCH -t 7-00:00:00
#SBATCH -o slurm.%j.out
#SBATCH -e slurm.%j.err

module load amber/22v3

mpiexec.hydra -n 8 pmemd.MPI -O \
-i production_npt_cpu.in \
-o production_npt_cpu.out \
-p 1btl.parm7 \
-c 1btl_heatup.rst7 \
-r 1btl_production_npt_cpu.rst7 \
-x 1btl_production_npt_cpu.crd
```

press `esc` button on keyboard and then type `:wq`

submit the job to queue by running

```
sbatch production_npt_cpu_sbatch
```

to check the status of the job run

```
squeue -u ikazan
```

## Production NPT (GPU)

Prepare `production_npt_gpu.in` input file:

```
vim production_npt_gpu.in
```

press `i` to enter edit mode

copy and paste the text below

```
# Production 20ns
&cntrl
imin=0,
ntx=5,
irest=1,
ntpr=5000,
ntwr=5000,
iwrap=1,
ntwx=5000,
ntwv=-1,
ioutfm=1,
nstlim=10000000,
dt=0.002,
ntt=3,
temp0=300.0,
ig=-1,
tautp=1.0,
gamma_ln=2.0,
ntp=1,
taup=2.0,
ntc=2,
ntf=2,
ntb=2,
cut=12.0,
&end
/
```

press `esc` button on keyboard and then type `:wq`

## Two options to continue: 1) Interactive, 2) Batch (I recommend 2)

### 1) Interactive mode

make sure you have GPU active session

```
interactive -G a100:1
```

run:

```
module load amber/22v3
``` 

run:

```
pmemd.cuda -O \
-i production_npt_gpu.in \
-o production_npt_gpu.out \
-p 1btl.parm7 \
-c 1btl_production_npt_cpu.rst7 \
-r 1btl_production_npt_gpu.rst7 \
-x 1btl_production_npt_gpu.nc
```

### 2) Batch mode

Prepare `production_npt_gpu_sbatch`  file:

```
vim production_npt_gpu_sbatch
```

press `i` to enter edit mode

copy and paste the text below

```
#!/usr/bin/env bash
#SBATCH -N 1
#SBATCH -n 6
#SBATCH -G 1
#SBATCH -p general
#SBATCH -t 1-00:00:00
#SBATCH -o slurm.%j.out
#SBATCH -e slurm.%j.err

module load amber/22v3

pmemd.cuda -O \
-i production_npt_gpu.in \
-o production_npt_gpu.out \
-p 1btl.parm7 \
-c 1btl_production_npt_cpu.rst7 \
-r 1btl_production_npt_gpu.rst7 \
-x 1btl_production_npt_gpu.nc
```

press `esc` button on keyboard and then type `:wq`

submit the job to queue by running

```
sbatch production_npt_gpu_sbatch
```

to check the status of the job run

```
squeue -u ikazan
```

# Analyze the MD simulation

Follow the MDEvaluationGuide guide here: [MDEvaluationGuide](https://github.com/John-Kazan/MDEvaluationGuide)
