# MD for protein-ligand interaction
This procedure will help the user run the multiple steps of a molecular dynamics (MD) simulation using GROMACS.
The outline is:
1) Generate topology (ligand force field with ANTECHAMBER, charges with QM calculations (Mulliken charges)
2) Define box and solvate
3) Add ions
4) Energy minimization (EM)
5) Step 1 Equilibration, constant volume (NVT)
6) Step 2 Equilibration, constant pressure (NPT)
7) Production
8) Analysis

## From the beggining: keep your folders organized
Create a folder with the name of your protein (PDB ID, four letters/numbers code. Ex. 2zff) to work in it, this way you will keep your folders in order. You just need to go to the path you want to create your folder and run the following line:
 ```mkdir 2zff```
In this directory you have to create two new folders: topol and md. In topol we will do step 1, and in md we will do the rest
 ```mkdir md```
  ```mkdir topol```
  
## Step zero: source source GROMACS and AMBER
1. For this tutorial, we will be using the GROMACS and AMBER that are in Bell cluster (Purdue). These programs need some modules to be loaded before hand, use ```module spider module_name```. This will list the needed modules, just run ```module load module_name```.
Then you can do ``` module load gromacs/check_correct_version``` and ``` module load amber```. 
Now you will be able to use gmx_mpi (GROMACS) and AMBER commands.

## Step one: create the initial QM input
1. Download or build a coordinate file called mol.ext for a molecule named mol. The extension (ext) can be anything that is supported by ASE (xyz, pdb, gro, …).
2. Run the following command:
  ``` qforce mol.ext -o config ```
This will generate a folder in your ```/qforce``` folder named ```mol_qforce/```. There you will find the following files:  <br />
    I) ```ini.xyz```, this is a copy of your initial coordinate file.  <br />
    II) ```settings.ini```, file with all Q-Force options (for more information go to: https://qforce.readthedocs.io/en/latest/options.html).  <br />
    III) ```mol_hessian.inp```, your input file to run calculations to obtain your 'first' hessian.  <br />
 Note: when running this command with the ```-o config``` option, the files generated will be in QChem format. If ```-o config``` is not used, you will get files in Gaussian format.
    
## Step two: modify settings.ini file
1. There are a few things that need to be changed in the settings.ini file. First let's see the content of the file:
  ``` vi settings.ini ```
2. Go to the [qm] section, and check: ```software = qchem```. Then change ```scan_step_size = 15.0``` to ```scan_step_size = 5.0```
3. Look at [scan] section and verify that folder is Q-Force using as data base for already existing fragments. It should look like ```frag_lib = /depot/lslipche/data/qforce/qforce_fragments```
4. To safe the changes (write and quit):
  ```:x``` or ```:wq``` 

## Step three: obtain QM Hessian matrix
In QM calculations, a Hessian matrix is a mathematical tool used to analyze the behavior of a system. It represents the second derivative of the energy with respect to the positions of all particles in the system. It provides information about how the energy of the system changes as the positions of the particles change, helping to determine stable configurations or the path of least energy during processes like molecular dynamics simulations or optimization of molecular structures.
1. First we need to set our input with all the proper options:
 ``` vi mol_hesssian.inp```  <br />
	**For ```jobtype = opt```:** <br />
	__Add:__ ```sym_ignore = true```, ```no_reorient = true``` <br />
	**For ```jobtype = freq:```**  <br />
	__Add:__ ```mem_static =10000```, ```sym_ignore = true```, ```no_reorient = true``` <br />
	__Change:__ ```mem_total = 40000```  <br />
Note: sometimes you will need to increase the memory.
2. Now we will be running QM calculations in QChem. If you look to your input file ```mol_hessian.inp```.
  ``` vi mol_hesssian.inp```
  You will see two jobs will be done: first a geometry optimization (GO) and then a vibrational frequecies (VF) calculation.
3. To run the calculations in QM we will run the following command:
  ```qc60 -q lslipche -w 336:00:00 -ccp 16 mol_hessian.inp```
Here we are running QChem ```qc60``` command, with a maximum time of 14 days or 336 hours ```-w 336:00:00```, in 16 CPUs ```-ccp 16``` for the file ```mol_hessian.inp```.
4. Now we just have to wait until the job is done, and the output will be:  <br />
  I) ```mol_hessian.out```, here are the calculations for the GO and the VF.  <br />
  II) ```mol_hessian.stdout```, if there are any problems with your calculations, the errors will be shown here.  <br />
5. To check how your calculation is going you can take a look to the file with ```vi mol_hessian.out```. And check is your job is done with ```squeue -u username``` or ```squeue -A lslipche```. The last one is particullarly useful to check how much CPUs are available to use.

## Step four: create fragments
If your molecule contains flexible dihedrals and if the treatment of flexible dihedrals are not turned off, then fragments and the corresponding QM inputs are created for all unique flexible dihedrals inside the subdirectory fragments.
1. Here we will run Q-Force to create the fragments that we will use to scan the dihedral angles in our molecule. For this, we need to be in the ```/qforce``` folder.
2. Verify you have source qforce.env from Step zero, and run:
  ```qforce mol```
This command will create the ```/fragments``` folder inside ```mol_qforce/```. In this folder you will find the list of the fragments in format ```.inp```, in other words, input files to run QM calculations in QChem.

## Step five: flexible dihedral scan
After generating all the fragments, we need to do a potential energy surface (PES) scan of the flexible dihedral angles in our molecule. Q-Force facilitates this process by generating the fragments already in ```.inp``` format (check file and look at the jobtype). 
1. This means we need to run QChem again as we did in Step three:
  ```qc60 -q lslipche -w 336:00:00 -ccp 16 fragment.inp```
In this case Q-Force names the fragments after the structure of the fragment itself (eg. CC_H20C10_...), where the two first letter indicate the bondend atoms that are connected forming the flexible dihedral angle (2 and 3 in the image below).  <br />
![alt text](https://www.researchgate.net/profile/Ari-Mohammed/publication/276170064/figure/fig1/AS:670705398059031@1536920036978/a-Se-chain-molecules-and-the-definition-of-the-dihedral-angle-The-dihedral-angle-is_W640.jpg)
2. After obtaining the PES scan of all the fragments, we need to check all the calculations run without errors. Remember to check ```.out``` and ```.stdout``` files.
3. Then we will need to delete the ```.stdout``` files, if not, Q-Force will have errors: <br />
 ``` rm fragment.inp.stdout```
 ## Step six: generate the Force Field
 Now we have all the necessary QM results we need to do a Hessian matrix fitting for the flexible dihedral angles and finally generate our force field file ```mol.itp```.
 1. We just need to check ```qforce.env``` is loaded, go to ```/qforce``` folder and run:
  ```qforce mol```
 2. If everything is correct, we will obtain the following files:  <br />
   **Folder ```mol_qforce/```:** <br />
   I) ```frequencies.nmd```(MM vibrational modes that can be visualized in VMD). ```frequencies.pdf```, ```frequencies.txt``` (QM vs MM vibrational frequencies).   <br />
   II) ```gas.gro```, ```gas.top```.  <br />
   **Folder ```fragments/```:** <br />
   I) ```fragment/``` Fragments trajectories <br />
   II) ```fragment.npy``` and ```fragment.pdf```. QM vs MM dihedral profile(s) <br />

  
  ## Other actions:
  **To not use all the fragments:** <br />
1. Delete all the fragments you want/ need from your ```fragments/``` folder. (rm ```fragment.inp```, ```.out```, and ```.stdout```).
2. Go to the ```settings.ini``` file that is in the folder of your molecule ```mol_qforce/```, ```vi settings.ini``` and change the option in [scan] to ```avail_only = yes```
3. run ```qforce``` normally
  
  **Add new atoms:** <br />
1. Go to the following path and view ```opls.itp``` file: <br />
	```cd /depot/lslipche/apps/qforce/qforce/lib/python3.8/site-packages/qforce/data```<br />
	```vi opls.itp```<br />
3. Add the opls code (e.g. ```opls_123```), atom number (```atom_number```), molar mass, charge, sigma, and epsilon parameters (in gromacs units). This can be found in the opls web page, or in papers for atoms that are not reported. Save the changes ```:x```.
4. Now go to the folowing path ```cd /depot/lslipche/apps/qforce/qforce/lib/python3.8/site-packages/qforce/molecule``` and ```vi non_bonded.py```. Add the following line at the end of the conditional function with all the opls codes:<br />
          ```elif elem == atom_number:```<br />
              ```a_type = 'opls_123'```<br />
Check the formating is correct, and you are using spaces and not tabs.<br />
5. Save your changes ```:x```.

  
