# Joint demo script

While I'm here representing Open Force Field specifically, I'm going to spend the next few minutes showing off how some OMSF projects work with each
other in addition to diving slightly deeper into a few Open Force Field capabilities along the way. This demo will show off not just Open Force Field, but also other OMSF projects -  OpenFold, Open Free Energy, and OpenADMET. All of the Open Force Field and Open Free Energy capabilities will be things that you can run right now, though the OpenFold and OpenADMET work will be capabilities that haven't yet been formally released. 

If you want to follow along, the materials are available on GitHub at the following link:
https://github.com/j-wags/joint-demo

Let's say we're a team of computational chemists supporting a team of medicinal chemists designing ligands for
biological targets. We are interested in MCL-1, which is a well-known target for oncology, and trying to inhibit its
activity.

In the future we might start with an OpenFold3 prediction, which we could get from a one-liner with relatively few
arguments, like this ... 

_show pseudo one-liner_

... but until OpenFold 3 is released, I'm going to be using a result from a
pre-production model that was prepared ahead of time.

_show co-folding section_

Here we have the PDB record 5FDR in blue compared against an co-folding prediction in red. We can see excellent
agreement with the crystallographic result,
though to be aware that this particular structure was part of the PDB training set for this model

For now, because of the results we have available, let's just use a well-established crystal structure of this target.

_show protein_

Imagine we've been given a set of ligands via a CSV file of SMILES.

_show ligand SMILES dataframe and 2-D structures_

Visual inspection will show you this this is a congeneric series based around a bi-heterocyclic core, with
substitution of the heterocycle and elaborations at both ends.

Our overarching goal is to asses how strongly each ligand binds to MCL1, but we also care if they'd have toxic off-target
effects. Off-target effects fall under the bucket of adsorption, distribution, metabolism, excretion, and toxicity, or ADMET. In the future, you'll be able to use OpenADMET's CLI to evaluate this ligand set against a series of cytochrome P450 anti-targets, which are a class of proteins that break down small molecules, and which are common drivers of unwanted drug-drug interactions.

OpenADMET models are currently being developed, and in the future models will also improve as the OpenADMET project
starts to receive data from their partners at Octant and UCSF.

For the purposes of our exercise, lets zoom in on CYP3A4 inhibition since CYP3A4 is responsible for large proportion of
hepatic metabolism relative to other CYPs. For starters, let's use use a cutoff of a predicted IC50 of 2.5 micromolar,
which corresponds to a pIC50 of 5.6. This is not realistic for a production run, but it's acceptable for our uses
today.

_run Pandas cells_

With a little bit of Pandas, we can quickly get a subset of the original ligand set which has PIC50 below our threshold
value. We are left with 14 ligands out of our original 18.

Having filtered out ligands with CYP3A4 issues, we are now moving on to free energy calculations of the remaining
ligands, which we will do with OpenFE. There are a few ways to use OpenFE's tooling - there is a CLI which provides easy
access to the most commonly-used features and a Python API which enables a wider set of more advanced features and
options.  Here we'll be using the CLI, but there's lots of cool stuff in the API too.

With our target and ligands already prepared, we will be using the OpenFE CLI in three parts to end with delta G
values. Basically, we prepare, run, and analyze these free energy calculations with three commands.

We will make use of YAML files which encode settings used in these commands. Our settings file is relatively simple,
leaving most things at their defaults, but there are many other options that can be enabled. We'll explore a couple of
these later.

_run plan-rbfe-network call_

This sets up a series of alchemical transformations between ligands which will be used to compute relative binding free energies with a minimum of computational cost.
Based on our settings file, the Kartograph atom
mapper is used and a minimal spanning network is generated. Open Force Field's Sage model is used for small molecule parameters, so AM1-BCC
partial charges are used. We can see some valuable information is logged.

The main output of this command is a bunch of JSON files, each describing a particular transformation. There's also a
GraphML file which describes the network, and OpenFE has some convenience tools used to visualize them.

_run `draw_ligand_network` cells_

If we had the right combination of GPUs and days to wait, we could run each of these simulations until they converge.
We would do this by calling `openfe quickrun` on each JSON file, which itself store each result in a JSON file. We
don't have an army of GPUs or a time machine, so I'm using some pre-computed results.

_run gather call_

After all simulations are run, the final step is to run `openfe gather` to get our relative binding free energies back.
With default options and the recent version 1.4.0, we get a pretty table of the dG of each ligand along with
corresponding uncertainty values.

So far we've taken a set of ligands and a known target, filtered out potential ADMET liabilities with OpenADMET's CLI,
and used OpenFE's CLI to predict relative binding free energies using OpenFF's small molecule force field Sage.

_pause_


Next, I want to dig deeper into some of these tools we've talked about so far in our HTS pipeline, and then finish up
with a few examples of other use cases enabled by OpenFF force fields and infrastructure.

Let's start by digging into the OpenFE CLI a little bit more.

Earlier when we ran `openfe plan-rbfe-network`, one of the steps involved assigning partial charges separately from how
other force field parameters are applied. AM1-BCC can occasionally give inconsistent results or meaningfully different
results with different hardware or software versions. Free energy calculations are also particularly sensitive to seemingly
small partial charge differences. So computing them once for each ligand helps mitigate these reproducibility issues.
There is a separate command in the CLI which controls this step.

_run openfe charge-molecules cell_

There are a number of other options that can be tinkered with in this command, just like other CLI calls, such as using
OpenEye to generate AM1-BCC ELF10 charges or using Open Force Field's new NAGL tool to generate charges using a graph neural network. These options can be passed in through
the `settings.yaml` file. Let's try using NAGL with just a single core, as opposed to the 

_run openfe charge-molecules cells with NAGL options_

With NAGL, we only used a single core to assign charges, as opposed to the 8 using explicit AM1BCC, and it still finished faster.

Next, let's talk about network planning, something else a practitioner can tinker with via the settings file. Open Free Energy's Konnektor tool
has has implemented a bunch of different networks, and a subset of them have been integrated into OpenFE. The default
network, which we used earlier, is a minimal spanning tree, which minimizes the number of edges that can connect all
nodes in a network.

_show network visualization_

If we wanted to switch to, say, a radial or star map, which connects a single node to each other node like a wheel with
spokes, we can do that by defining it in the settings YAML and re-running the network planning command. This network
takes another argument for the central ligand, let's just use ligand 6.

_show radial.yaml, run plan-rbfe-network, show new visualization_

There are plenty more options exposed in the CLI and documented in the CLI reference, and further more functionality
available in the Python API.

Finally, I want to show off some things that we can do with OpenFF infrastructure. So far in this demo, OpenFF force
fields have been used by OpenFE under the hood, by default, for its small molecule parameters.  There's much more
functionality that can be accessed by interacting directly with OpenFF software, but it's also very easy to get on the
ground running for simple system. For starters, we'll show that with OpenFF it takes literal seconds to go from loading
a molecule into RDKit to visualizing a simulation trajectory.

Let's take the ligand which gave us the strongest dG molecule in SDF, ligand 3, which we can load into RDKit and have a quick look at.

_show ligand_3 cell_

We need to load a force field; here I'll use a recent version in the Sage line, version 2.2.1. From here, it's a
one-liner to prepare an OpenMM system and a little bit of boilerplate to get the simulation running. I've wrapped that
up into a separate function in the file `simulate.py` if you want to have a look. This function returns an NGLview
widget of the trajectory, and that's all it takes to run MD from RDKit.

_show lig_3 trajectory_ 

Next we are going to use OpenFF tooling to simulate protein-ligand complexes. The process is similar to running
simulations of a single molecule. The main difference is that we need to prepare a topology, which is simply a
collection of molecules, and that can take a couple of extra steps. Since this will run more slowly on my laptop, I'm
going to run these cells ahead of time so the simulation has time to run.

For starters, we'll load a PDB file of a protein-ligand complex into a `Topology` object, also passing a SMILES pattern
representing the ligand. If we have a look, that looks good.

_show protein-ligand complex, run solvation_

Since we don't want to simulate this in vacuum, we need to add some solvent. There are a few ways of doing this, and
for systems with only water and canonical residues, PDBFixer is a good choice.  Now we have the same protein-ligand
complex solvated in water with ions.

_show solvated protein-ligand complex, run next cell_

The next step is to load a force field appropriate for this system; we're going to use the same Sage force field for
the small molecule parameters and for the protein we'll use a SMIRNOFF port of ff14SB. One could slot in other force
fields here, for ligand, protein, or both chemistries, just by passing different file names to the class. The next step
is to make an Interchange object out of this topology and force field, which can take a few seconds so I'm just going
to load a serialized representation of this.

_show Interchange cell_

Finally, we can re-use some of that same boilerplate OpenMM code to get an MD simulation out of this state and then
again look at it with NGLview. This function prepares and runs an OpenMM simulation for 30 seconds, so we're not going
to get a long trajectory on this hardware but we do see some dynamics.

_show protein-ligand trajectory_

Here we've taken a protein-ligand complex in a PDB file and used OpenFF tooling to run an MD simulation.

_pause_

Sometimes your amino acids may be non-canonical, or even simply a small molecule covalently linked to a side chain.
OpenFF has a prototype post-translational modification workflow, enabled by new science and infrastructure, which
enables simulating these systems with a modest amount of preparation.

Let's say we have a flourescent dye tagged to our protein. We can load our dye as a standalone molecule like any other
small molecule and have a look. This molecule has has a maleimide group which can take part in a click reaction with
the thiol on a cysteine residue.

_run 2-D dye cell_

That reaction can be written up in SMARTS and visualized with RDKit.

_run RDKit reaction cell_

We're ultimately going to load this system as a PDB file through OpenFF Pablo, a new tool being developed for better
biopolymer loading in OpenFF infrastructure. A core feature of Pablo is support for custom residue
definitions. These can be defined in a few ways and in this case, it's easiest to make one from an OpenFF molecule. We
want the residue to represent the cysteine modified with the dye, so we need to get that into a single-molecule
representation. After reaction, we have a single-molecule representation of this modified residue with some atom names changed to match the PDB file.

That's just about all the set up we need to do. The last step is creating our residue definition from this molecule and
a canned definition of a peptide bond. We can pass this to Pablo's `topology_from_pdb` function and have a look at the
result.

_run 3d viz cell_

Now that we have a topology, we can do just what we did before - take it and a force field and pass it off to OpenMM,
then visualize the result. A final wrinkle here is that, due to the modified residue, NAGL was used to assign charges
to the protein, although ff14SB charges were backfilled onto standard residues so only the modified cysteined and dye
ended up with end up with NAGl-assigned charges.

Finally, I wanted to demonstrate one of the many powerful features of NAGL, which is how quickly it can assign partial
charges to large ligands. I've extracted the ligand from the 5FDR PDB record, which is larger than that ligands in the
screening workflow we went through earlier. AM1-BCC becomes much slower as the number of heavy atoms in a molecule
increases.

_show ligand cell_

Timing I did ahead of time shows that this takes more than three minutes with AmberTools.

_show NAGL timings cell_

NAGL can assign partial charges to the same molecule quickly enough that we can do it live. On average, the runtime is
less than one second.

In summary, we've demonstrated how OMSF tools can work together in a high-throughput screening pipeline. We also looked
more closely at some more advanced features offered by each project. This included
* a co-folding result generated by a pre-production OpenFold 3-style model
* a closer look at ADMET filtering with OpenADMET
* some of the free energy options exposed in the OpenFE CLI
* running protein-ligand complexes, including post-translational modifications, with OpenFF

This is just a small sample of what can be done with OMSF tools, however. We're excited to see what you can do with
them in the future.

Credits

... and for any questions, I'm happy to continue the discussion in the conference discord server.
