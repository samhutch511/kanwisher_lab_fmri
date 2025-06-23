# Kanwisher Lab fMRI tutorial 

By Sam Hutchinson, feel free to email with questions: samhutch@mit.edu

This tutorial will walk you through a specific analysis of some Efficient Localizer data, but will show the general template for most of the analyses you will do in the lab. This is based on two sets of slides I made, one on [general fMRI analyses](https://docs.google.com/presentation/d/1JcAaVyMq0PLZW0SvOdoyjupPdqze7pA5VzY4pgGA_cQ/edit?usp=sharing) and another on how I set up the [pipeline on Engaging](https://docs.google.com/presentation/d/1ibxpqWIziyznhrbBj4jMWIN-6axbcHgNMaJBtp5g6i4/edit?usp=sharing).

### A note on pipelines
Our lab doesn't have a standard pipeline everyone uses, which is unusual. EvLab, for example, has everyone use the exact same code for processing their fMRI data using SPM. There are pros and cons to each approach. This tutorial will use the [FreeSurfer](https://surfer.nmr.mgh.harvard.edu/fswiki) pipeline I mainly wrote, by cobbling together code from Pramod, Ratan Murty (former Kanwisher Lab postdoc, current Georgia Tech professor), and Rui Xu (current postdoc in Desimone Lab).

## Table of contents
* [From the scanner to brain maps (first-level analyses)](https://github.mit.edu/kanlab/lab/wiki/fMRI-pre-processing-and-analysis#from-the-scanner-to-brain-maps-first-level-analyses)
> * [Setting up the directory](https://github.mit.edu/kanlab/lab/wiki/fMRI-pre-processing-and-analysis#setting-up-the-project-directory)
> * [Unpacking the data from the scanner](https://github.mit.edu/kanlab/lab/wiki/fMRI-pre-processing-and-analysis#getting-the-raw-data-dicoms-from-the-scanner)
> * [Anatomical reconstruction](https://github.mit.edu/kanlab/lab/wiki/fMRI-pre-processing-and-analysis#reconstructing-the-anatomical-scan)
> * [Opening FreeView, visualizing anatomicals](https://github.mit.edu/kanlab/lab/wiki/fMRI-pre-processing-and-analysis#visualizing-the-anatomicals-with-freeview)
> * [Running the contrast analyses](https://github.mit.edu/kanlab/lab/wiki/fMRI-pre-processing-and-analysis#running-the-analyses)
> * [Visualizing the FFA and language network](https://github.mit.edu/kanlab/lab/wiki/fMRI-pre-processing-and-analysis#visualizing-the-contrast-results)
* [From brain maps to bar plots (second-level analyses)](https://github.mit.edu/kanlab/lab/wiki/fMRI-pre-processing-and-analysis#from-brain-maps-to-bar-plots-second-level-analyses)
> * [Transforming parcels to subject-space](https://github.mit.edu/kanlab/lab/wiki/fMRI-pre-processing-and-analysis#transforming-parcels-to-subject-space)
> * [Loading and plotting data with Python](https://github.mit.edu/kanlab/lab/wiki/fMRI-pre-processing-and-analysis#loading-and-plotting-data-with-python)

## From the scanner to brain maps (first-level analyses)

### Getting started
1. We'll be working in the directory: `/orcd/data/ngk/001/shared/new_fmri_tutorial`
> * This should be empty, we'll populate it with what we need as we go
> * If this is not empty, you can just make a new empty directory to use wherever it's convenient for you
> * For the rest of the tutorial, I'll call this `{project_dir}`
2. You can find all the scripts you'll need to run the analyses in: `/orcd/data/ngk/001/shared/samhutch_fmri_scripts`
3. You'll need some specific things in your `.bashrc` file (located in your `/home/{your_user}` directory)
> * You can copy all of mine ([at this link](https://github.mit.edu/kanlab/lab/blob/master/sams_bashrc)), or just the parts defining the environment variables `FS_BINDS`, `FS_ENV`, `FS_BINDS_NOMATLAB`, the alias `freeview`, and the section loading the `modules`
> * You will need to define another environment variable, called `PROJ_DIR`, and set to the `{project_dir}` path
4. It will be helpful to have two terminals open, one logged into Engaging and the other logged in to OpenMind. It will be easier to learn about commands on OpenMind with `{command} --help` since we can just run FreeSurfer from the command line directly over there.

### Setting up the project directory
You'll need to make the following sub-directories in `{project_dir}`:

`analysis`
> * Will eventually contain files storing important variables

`data`

`data/dicoms`
> * Where you should unpack the scanner data, each subject should have their own directory

`parcels`
> * This is where the parcels we'll use will go (we'll explain when we get there)

`recons`
> * Where the anatomical reconstructions will end up, the recon script will populate this

`scripts`
> * Where you’ll have the code which runs the analyses
> * Run this command to copy all the scripts we'll need: `cp /orcd/data/ngk/001/shared/samhutch_fmri_scripts/* {project_dir}/scripts`

`paras_vis`

`paras_aud`
> * `paras_{exp}` are where you will keep the paradigm files (files that list the conditions and timing) for each experiment, each subject should have their own directory
> * We'll grab these from an already-existing subject for this tutorial. To do so, run these two commands:
> * `scp -r /orcd/data/ngk/001/projects/efficient_localizer/paras_vis/kaneff01 {project_dir}/paras_vis` 
> * `scp -r /orcd/data/ngk/001/projects/efficient_localizer/paras_aud/kaneff01 {project_dir}/paras_aud`

`vols_vis`

`vols_aud`
> * `vols_{exp}` are where the preprocessed functional data and GLM results from each experiment will end up, the experiment scripts will populate these

### Getting the raw data (dicoms) from the scanner
1. Log in to **OpenMind**, since the first command you'll need here currently doesn't work on Engaging
> * In your **OpenMind `.bashrc`**, you'll need to add this line: `module load openmind/mrimages/0.1`
> * Then run the command: `source .bashrc`
2. Run the command: `SessionFinder --s kaneff01`
> * `kaneff01` is the subject we'll be working with, but in practice you'll list the name you used during the scanning session
> * This command should list a path that looks something like: `/mindhive/MRI/Prisma-166193-20240605-161030-000814`
> * For now, we'll just copy the dicoms from an existing directory already on Engaging, since copying them from here takes a while
3. Log back into Engaging, and **get in an interactive session**
> * You can get an interactive session for one hour by running this command: `srun -p mit_normal -N 1 -n 1 --mem=50GB --time 01:00:00 --pty /bin/bash`
> * You can request more time by changing the `--time` flag, but one hour should be enough
> * See [here](https://orcd-docs.mit.edu/running-jobs/overview/) for more info on SLURM on Engaging
4. Navigate to `{project_dir}/data/dicoms`
5. Make a new directory here called `kaneff01` and `cd` into it
6. Run the following command: `rsync -av /orcd/data/ngk/001/projects/efficient_localizer/data/dicoms/kaneff01/dicom ./`
> * Normally, that first path is where you'd put the output of `SessionFinder`
> * You should see a bunch of files named things like `MR.1.3.12...` start flying by. These are all the individual raw images from the scanner. This will take a few minutes to complete.
> * Once this is done, you should see a new subdirectory: `{project_dir}/data/dicoms/kaneff01/dicom`
7. Now we need to get a summary of which images go with which runs. To get this, run the command: `singularity exec $FS_BINDS_NOMATLAB $FS_ENV /orcd/data/ngk/001/fmri_software_images/freesurfer-image-v2.sif unpacksdcmdir -src dicom -targ ./ -scanonly dicom.info`
> * This passes the `unpacksdcmdir` command to our **FreeSurfer container (see [here](https://docs.google.com/presentation/d/1ibxpqWIziyznhrbBj4jMWIN-6axbcHgNMaJBtp5g6i4/edit?usp=sharing) for more on this)**
> * This will take another few minutes. Once this is done, you should see a file called `dicom.info`
8. Run the command `cat dicom.info`
> * This should print out a list of runs, with columns: run_num, run_name, status, dimensions, dicom_name
> * Find the last run which was an **anatomical scan (which you can tell by its name having something like T1 or MPRAGE**; in our case it's **run 6**. Note this number, and we'll compile that run in the next step.
9. Unpack (compile raw dicoms into a coherent file) the anatomical scan by running this command: `singularity exec $FS_BINDS_NOMATLAB $FS_ENV /orcd/data/ngk/001/fmri_software_images/freesurfer-image-v2.sif unpacksdcmdir -src dicom -targ {todays_date_YYYYMMDD}_unpackdata -fsfast -run 6 3danat nii 001.nii.gz`
> * Note that we indicated run 6 as the one to unpack!
> * This should result in a subdirectory called `{todays_date_YYYYMMDD}_unpackdata`. Under that subdirectory, you should find the file `/3danat/006/001.nii.gz`. This `.nii.gz` file is the anatomical scan which FreeSurfer will use to reconstruct the whole brain in the next section.

### Reconstructing the anatomical scan

1. Normally, you'd use the `run_recon.sh` script which we'll walk through below, but for now **do not actually run that script** as it will take several hours to complete. For this tutorial, we'll just copy an already-completed anatomical.

2. Navigate to the `{project_dir}/scripts` directory, and run the command `cat run_recon.sh`. This prints out the code in the reconstruction script, which we'll walk through here.
> * First, you'll see all the header information we need to include for SLURM to run this on a compute node in the background. When you actually run this script yourself, you'll want to change the `--mail-user` flag to your own email (not that I don't love updates on everyone's projects, but it's more useful for you to know).
> * Next, you'll see the three input arguments the script takes: `subj_id` (in this case, it would be `kaneff01`), `run_num_three_digit` (in this case, it would be `006`), and then the date in `YYYYMMDD` format.
> * Next, you'll see that it's loading the modules and setting the variables the actual command call needs. You don't need to worry about these.
> * Finally, it passes the `recon-all` command to the FreeSurfer container, with all of the relevant info.
> * If you were to actually run this script, which **you should not do for the purposes of this tutorial**, you would do so by entering the command: `sbatch run_recon.sh kaneff01 006 {YYYYMMDD}`
> * **Crucially**, when running this for real, you will need to make a subdirectory called `reconlog` in `{project_dir}/scripts`, or else the code will fail without any obvious errors.
> * You would also be able to check the status of your job by running this command: `squeue -u {your_user}`. This is generally a useful thing to do, since it will actually show you all of your running jobs on the cluster.

3. For now though, instead of actually running the script, we'll just copy over the anatomical we've already reconstructed, with this command: `scp -r /orcd/data/ngk/001/projects/efficient_localizer/recons/kaneff01 {project_dir}/recons`

4. When this completes, you should see the directory: `{project_dir}/recons/kaneff01`, with a bunch of subdirectories, including `label`, `mri`, `surf`, etc. This is the completed anatomical reconstruction!

### Visualizing the anatomicals with Freeview

1. Make sure you've copied the alias `freeview` from [my `.bashrc` file](https://github.mit.edu/kanlab/lab/blob/master/sams_bashrc)
2. Follow the instructions on slides 4-7 of [this presentation](https://docs.google.com/presentation/d/1ibxpqWIziyznhrbBj4jMWIN-6axbcHgNMaJBtp5g6i4/edit?usp=sharing) to get an interactive Xfce session with Engaging OnDemand.
> * You probably don't need 24 hours and 50GB of memory, so set those to something more reasonable like 5 hours with 10GB for now.
3. To open a terminal in your Xfce session, click `Applications` --> `Terminal Emulator`.
4. To open Freeview, the standard FreeSurfer visualization tool, simply type: `freeview` in the terminal.
5. You should now see a GUI that looks like this: ![empty freeview screen](https://github.com/samhutch511/kanwisher_lab_fmri/blob/main/tutorial_figures/empty_freeview.png)

#### Visualizing "in the volume"

6. To open the anatomical scan, click on `File` --> `Load Volume`, then the file icon.
7. This should bring up your home directory, and if you've copied by `.bashrc`, then one of the subdirectories here should be called `data`, and this is where you'll find the files pointed to by your `$PROJ_DIR` environment variable (which should be set to the `{project_dir}`).
8. Click `data` --> `recons` --> `kaneff01` --> `mri` --> **`orig.mgz`**. This should take you back to a menu which looks like this: ![anatomical loading menu](https://github.com/samhutch511/kanwisher_lab_fmri/blob/main/tutorial_figures/load_anat.png)
9. Make sure the "Color map" is set to "Grayscale," then click "Okay". This should load in the anatomical scan, which looks like this: ![anatomical image](https://github.com/samhutch511/kanwisher_lab_fmri/blob/main/tutorial_figures/anat.png)
10. If your view is set to something different, you can change the layout and orientation with these buttons along the top of the window: ![buttons to change view](https://github.com/samhutch511/kanwisher_lab_fmri/blob/main/tutorial_figures/layout.png)
11. I like to set these to "1 & 3 Horizontal" and "Coronal," respectively, which is what you see in the screenshot above.
12. The image which has the blue, green, and red bars can be rotated around by clicking and dragging, and you can slide these bars by clicking and dragging them to change which slice you're viewing. Try playing around with this! If you want to reset to the initial view, press the button which has the two blue arrows pointing at each other along the top, next to the view-change buttons.
13. By clicking anywhere on the main image, you can move the red cursor to that point, which will show that point in the other slices as well. 

#### Visualizing "on the surface"

14. What we just looked at is called the "volume," since it is a 3D shape which we can move through (with the sliders).
15. We can also look at the "surface" of the brain, which is a reconstruction based on this anatomical file. We'll go through how to load that now.
16. Click the "Close Volume" button (the face with the red X) beneath the "Volumes" window on the left to clear the anatomical scan.
17. Click `File` --> `Load Surface`, and navigate to `data` --> `recons` --> `kaneff01` --> `surf`, then open the file **`lh.inflated`**. Select "3D" from the view-change buttons (looks like a 3D face), and you should see this: ![red and green surface](https://github.com/samhutch511/kanwisher_lab_fmri/blob/main/tutorial_figures/red_green_surf.png)
18. This is someone's left hemisphere! The "inflated" part means that the sulci (valleys between the peaks, or gyri) have been expanded to be on a flat surface. From the left-side buttons, click "Curvature" and select "Binary". This should load the brain in a more standard black-and-white, which looks like this: ![black and white surface](https://github.com/samhutch511/kanwisher_lab_fmri/blob/main/tutorial_figures/gray_surf.png)
19. By clicking and dragging the brain around, you can rotate it to see different views.
20. Try loading a right-hemisphere surface, as well as some different kinds of surfaces, like `pial` or `orig`!

### Running the analyses!

1. Go ahead and close Freeview by clicking the "Exit" button in the top-right corner.
2. Make sure you've loaded the `matlab` module, either by copying that section from my `.bashrc` or by running `module load mit/matlab/2020a` in your Xfce terminal.
3. Navigate to the `{project_dir}`, and open Matlab by simply typing `matlab`. You should see something which looks like this: ![blank matlab screen](https://github.com/samhutch511/kanwisher_lab_fmri/blob/main/tutorial_figures/blank_matlab.png)

#### Paradigm (para) files
4. Let's open a para file to talk a bit more about what's going on there. Double-click on `paras_vis/kaneff01/kaneff01_1_vis.para` to open it. It should look like this: 

![para file](https://github.com/samhutch511/kanwisher_lab_fmri/blob/main/tutorial_figures/para.png)
> * If you haven't already watched [Idan Blank's excellent videos on fMRI analyses](https://www.youtube.com/watch?v=qgKm3EayUWY), they will help a lot in making sense of what's going on here (as well as the later analyses). I recommend highly!
5. This file tells FreeSurfer what kind of stimulus we presented to the subject, when we presented it, and for how long. The three columns encode all this information:
> * The first column, in seconds, lists the onset of each condition.
> * The last column, also in seconds, lists the duration of each condition.
> * The middle column corresponds to the **condition IDs** which we're using to code what we showed in the scanner. You will have a different ID number for each condition. **0 always refers to fixation**, or when we were not showing anything on the screen, and FreeSurfer will use this as a baseline. In this experiment, 1 refers to showing videos of **faces**, 2 refers to videos of **scenes**, 3 refers to **bodies**, 4 to **objects**, and 5 to **words (superimposed on a scrambled-object background)**.
> * To get a sense of what this experiment looks like, [watch this video](https://www.dropbox.com/scl/fi/hb3jkdr5ulx4ffh1p3rpw/test_5x5_nw_quilt_fixed.mov?rlkey=of8dq18vx2uvylmiom9ngzkky&st=hp8kx9ir&dl=0) without sound. The conditions are not in the same order I listed them above, since this is a snippet from a later run.
6. Note the `_1_` in the para file name, and also that there are files with `_2_`, `_3_`, `_4_`, and `_5_` in the para file directory. These refer to the **runs** of the experiment, and **each run needs its own separate para file.**
7. It's best practice to have your experiment script (what you actually run during the scan) output para files for each subject and each run.

#### Using the analysis scripts
8. In Matlab, open the `data` --> `dicoms` --> `kaneff01` --> **`dicom.info`** by double-clicking on it. You should see this: ![dicom.info](https://github.com/samhutch511/kanwisher_lab_fmri/blob/main/tutorial_figures/kaneff01_dicom_info.png)
> * We'll be referring to this as we fill in the information in the scripts, so keep it open!
9. Now, navigate to `{project_dir}/scripts`. You should see the four main scripts we'll be using, called:
> * `batch_process_rx.m`: this is where we'll fill in most of the info for the analyses
> * `prep_analysis_rx.m`: this is where we tell FreeSurfer what kind of analyses to do, as well as what **contrasts** to run
> * `make_L2_rx.m`: this will create the data structure which holds all the variables we need under `{project_dir}/analysis`
> * `block_analysis_rx.m`: this is what actually runs the analyses we specify!
10. Open `batch_process_rx.m`. It should look like this: ![batch_process_rx.m](https://github.com/samhutch511/kanwisher_lab_fmri/blob/main/tutorial_figures/batch_process.png)
11. If you scroll down a bit, you'll see a commented-out example of how to include a subject. A few things to note:
> * `subj_id` should be a string that matches the `recon`, `data/dicoms`, and `paras_{experiment}` folders
> * `tasks` should be a cell array of strings, with the names matching the `{experiment}` in the `paras_{experiment}` folders
> * `tasks_runs` should be a cell array of double arrays, where each double array lists the runs (found in `dicom.info`) associated with each task. Here, we can see that the subject did two runs of a language localizer in runs 23 and 24.
> * The rest of the variables are blank, since we didn't do any field maps or DTI runs. These are beyond the scope of this tutorial, so we'll skip those for now.
12. First, a bit more info about the experiments we'll be analyzing in this tutorial. This is data from [the Efficient Localizer](https://github.mit.edu/kanlab/lab/blob/master/documents/effloc_submission.pdf), which uses simultaneous video and audio. We've already gone through the visual conditions in the para file above. The auditory conditions are: 1—**false belief stories (in English)**, 2—**false photo stories (in English)**, 3—**nonwords**, 4—**quilted speech**, and 5—**math problems**. If you want a sense for the simultaneous presentation, you can watch the video linked in the para section with audio on. Despite these being presented simultaneously, we'll analyze them separately, as the `vis` and `aud` experiments (which you should have para files for). Here's a schematic from our paper on the localizer: ![effloc_fig1](https://github.com/samhutch511/kanwisher_lab_fmri/blob/main/tutorial_figures/effloc_fig1.png)
13. Now let's fill in the info for the `vis` and `aud` experiments which we have para files for. Scroll back up to the un-commented subject information at the top of the code block.
> * `subj_id` should be `'kaneff01'`
> * `tasks` should be `{'vis', 'aud'}`
> * `tasks_runs` should be `{[8,9,10,11,12],[8,9,10,11,12]}`. Since the conditions were simultaneous, we use the same run numbers for both the `vis` and `aud` experiments. You can see in `dicom.info` that the "effloc" (short for Efficient Localizer) runs are listed as 7-12, but that run 7 has an `err` in the status column, while 8-12 say `ok`. That means there was some error during run 7 (I don't remember what it was now), so we won't use it.
14. Before we actually run these analyses, let's take a look at the other scripts and what they do. If you scroll to the bottom of `batch_process`, you'll see that it calls `prep_analysis_rx.m`. Go ahead and open `prep_analysis`.
15. You can ignore the first part of this script, but scroll down until you see the binary flags called `do_volume`, `do_surface`, etc. This is where we tell FreeSurfer what kinds of analyses we want to do. It should look like this: ![prep_analysis_flags](https://github.com/samhutch511/kanwisher_lab_fmri/blob/main/tutorial_figures/prep_analysis_flags.png)
16. For now, we want to set `do_volume`, `do_surface`, `do_preproc`, `make_analysis`, and `do_selxavg` to `1`, and to set the rest to `0`. In English, this means we want to preprocess the data (`do_preproc`), run a GLM with some specified parameters (`make_analysis`), and use the default FreeSurfer algorithm to run the GLM (`do_selxavg`), and do all of these both in the volume and on the surface (`do_volume` and `do_surface`).
17. Now scroll down a bit farther, and you should see where we're defining the **contrasts** we want to analyze. That section should look like this, but with fewer contrasts: ![visual_contrasts](https://github.com/samhutch511/kanwisher_lab_fmri/blob/main/tutorial_figures/visual_contrasts.png)
> * You'll need a different block like this for each experiment that you run, you should see blocks here for both `vis` and `aud`.
18. The first few entries here should be pretty self-explanatory, where you list the number of conditions (including fixation), the length of each condition, and the names of the para files.
19. The trickier part is defining your contrasts. Let's look at one particular line to break it down: 

`c = c+1; contrast.names{c} = ['F-O']; contrast.cidleft{c} = [1]; contrast.cidright{c} = [4];`
> * This defines a contrast of **Faces** over **Objects**, which should show us which parts of the brain are significantly **more active when the subject was watching videos of faces than when they were watching videos of objects**
> * First, we increment `c`, which is our index which keeps track of the number of contrasts we're running for this experiment.
> * Next, we name our contrast something useful to us, usually abbreviated forms of the condition names. For faces, we usually use `F` or `Fa`, and for objects, usually `O` or `Obj`. Note that the **positive contrast**, faces, is listed on the **left** in the name.
> * Next, we tell FreeSurfer which **condition ID numbers** (recall how we set up the para files) correspond to this contrast. For the positive (or `left`) contrast's condition id (`cid`), we put **`1`**, since that's how we coded the faces condition in the para files. For the negative condition id (`cidright`), we put **`4`**, since that's how we coded the objects condition.
20. Note how we've defined the other contrasts in `vis` and `aud` as well: we have an all-conditions-versus-fixation contrast for each, which is always good to run as a sanity check (to make sure you're getting visual and auditory regions, respectively), as well as a `FP-NW` (false-photo over nonwords) auditory contrast, which should reveal language-selective regions.
21. If you're feeling confident, try defining a scenes-versus-objects `vis` contrast and a false-belief-versus-false-photo `aud` contrast!
22. Scroll to the bottom of `prep_analysis`, and you should see that the script calls `make_L2_rx.m` and `block_analysis_rx.m` now. `make_L2` just saves out all the variables we've defined so far in a helpful data structure; we don't need to worry about it for this tutorial. You can go ahead and open `block_analysis_rx.m` now (we're almost done!).

#### Steps in `block_analysis`
There are a few key steps in this function, which you shouldn't have to make any changes to, but we'll walk through them here to understand what's going on.

23. **Lines 31-47, setting up the container call**: here, we define some strings which we'll append to later commands to make those run through our FreeSurfer container. You don't need to worry about the precise details here, but it's good to start getting familiar with how to make calls to the container.
24. **Lines 48-98, unpacking the functional runs**: just as we had to unpack the anatomical images into a coherent file, this part of the code does the same for the functional runs we specified in `batch_process`. Once this step completes, you should see directories for each run under `vols_{exp}` --> `{subj}` --> `bold` --> **`{runs}`**. 
> * In each of these files, which for us would be, for example, `vols_vis` --> `kaneff01` --> `bold` --> `008`, you should see a file called **`f.nii`**. This is our unpacked functional data!
25. **Lines 192-271, preprocessing the functional data**: once we have the **f.nii** files in each run's subdirectory, we need to clean them up a bit for further processing. There are two main functions in this section: **preproc-sess** and **mri_vol2vol**.
> * `preproc-sess` does two main things: motion-correction and smoothing. The `per-run` flag means that we're correcting for motion in the scan by transforming each time point to match the middle time point in each run. Once this step completes, you should see a file called **`fmcpr.nii.gz`** in the `bold` directory. The smoothing means that we're going to calculate a weighted average among each voxel's neighbor, and once this is done, you should see a file called **`fmcpr.sm3.nii.gz`**. The `sm3` part of that means that we're taking a weighted average over 3mm in space. We can change this, but 3 is a good place to start.
> * `mri_vol2vol` does what the name describes: it takes a volume MRI image (our `fmcpr.sm3.nii.gz`) and transforms it to the space of another volume, in our case, we will match our functional data to the anatomical image we looked at earlier (`orig.mgz`). This process is called **registering** the functional to the anatomical, and the output file will be called **`fmcpr_reg.sm3.nii.gz`**.
> * Breaking down the name to recap: we start with the **f** for "functional," append **mcpr** for "motion correction per-run," **reg** for "registered," and **sm3** for "smoothed 3mm".
26. **Lines 274-420**, setting the analysis parameters: once we have the **fmcpr_reg.sm3.nii.gz** files in each run's subdirectory, we can start preparing to actually analyze the data! The main functions in this section are `mkanalysis-sess` and `mkcontrast-sess`, and both provide FreeSurfer with some more details of what we want to do.
> * You'll see a bunch of flags under `mkanalysis-sess`, and most of these you won't ever have to change. These are pretty good defaults for a bunch of smaller details (run `mkanalysis-sess --help` on OpenMind to get more info on each).
> * `mkcontrast-sess` reads the contrast data structure we made in `prep_analysis_rx.m` into a format that FreeSurfer can work with.
27. **Lines 423-498**, running the GLM: here, we'll pass everything we've collected so far to a command called `selxavg3-sess`. This is FreeSurfer's default algorithm for calculating the GLM and contrast maps.
> * Once this is done, you should see a bunch of subdirectories under `vols_vis` --> `kaneff01` --> `bold`, named with your experiment, the smoothing parameters you used, and which split of the data was used to calculate it (by default, we run three GLMs: one using all the runs, one using only the odd runs, and one using only the even runs—we do this so we can [define regions and measure responses in independent data](https://pmc.ncbi.nlm.nih.gov/articles/PMC2841687/)).
> * For our data here, that means we should see subdirectories called `vis.sm3.all`, `vis.sm3.odd`, `vis.sm3.even`, and so on. The ones with hemisphere tags (`.lh` or `.rh`) are surface-based data, the ones without these hemisphere tags are the volume-based data.
> * Open `vis.sm3.all`. You should see files with a bunch of parameters of our analyses, as well as subdirectories which match the name of each contrast you specified. Open, for example, `F-O`, and you should see a bunch of `.nii.gz` files. These are the statistical maps you see plotted as blobs on brains in papers!
> * For our purposes here, we'll be plotting the significance values of the t-tests conducted based on our contrasts, which are stored in the `sig.nii.gz` file in each contrast's subdirectory.
28. **BONUS, CAN SKIP IF ONLY HERE FOR BASICS: Lines 502-591**, GLMsingle: `selxavg` is only one algorithm for calculating the GLM, and it makes some assumptions which may not hold in every situation we want to model. For example, it models each event as a "block," assuming we showed many trials of each condition back-to-back and not interleaved with other conditions. For the Efficient Localizer, this is true: we showed a bunch of videos of faces, then a bunch of videos of scenes, and so on. But this is not always the case! Sometimes, you'll be running "event-related" experiments, which have short trials of many different conditions interleaved with one another. In that case, you'll need an estimate of the response to each *trial*, rather than to each *block*. For these event-related experiments, or whenever else you'll need trial-level estimates, you'll want to run [the **GLMsingle** algorithm](https://www.youtube.com/watch?v=OvxOUn0-tn0) for your GLM. I found [this page](https://htmlpreview.github.io/?https://github.com/kendrickkay/GLMsingle/blob/main/matlab/examples/example2preview/example2.html) very helpful in learning to use GLMsingle and understanding the outputs.

#### The moment of truth!
29. Now, you can open `batch_process_rx.m`, double-check the information you've included there, and run the script! It will take a while to complete all the steps, but will print a continuous log to the command window in Matlab. I come back and command-f for "error" every 15 or 20 minutes to make sure things are progressing smoothly. Also be sure to monitor the output directory (`vols_{vis/aud}` --> `kaneff01` --> `bold`) for the files that indicate things are working (`f.nii`, `fmcpr.nii.gz`, `fmcpr.sm3.nii.gz`, `fmcpr_reg.sm3.nii.gz`, then the `{vis/aud}.sm3.all` directory with `{F-O/FP-NW}` containing `sig.nii.gz`).
30. Don't worry if you encounter errors here! Try your best to understand the error message and correct it, and if you get stuck try googling it or shoot me an email (samhutch@mit.edu).

### Visualizing the contrast results
1. The hard part's over! If you've made it this far and have all the correct output files, congratulate yourself and take a break: get up, stretch, go for a walk, get a coffe. Next we just get to see the results of all this work!
2. Get a new terminal in the Xfce session and open Freeview (see the above section on visualizing the anatomicals).

#### Visualizing the FFA in the volume
3. Open the `orig.mgz` volume the same way as before.
4. Now, click `File` --> `Load Volume`, then the file icon. Navigate to `vols_vis` --> `kaneff01` --> `bold` --> `vis.sm3.all` --> `F-O`, then double-click on `sig.nii.gz`. This should take you back to a menu that looks like this: ![load_f_o](https://github.com/samhutch511/kanwisher_lab_fmri/blob/main/tutorial_figures/load_f_o.png)
5. You want to change the "Color map" to "Heat," then click the green "Okay" button to load the image. That should look like this: ![f_o_nothresh](https://github.com/samhutch511/kanwisher_lab_fmri/blob/main/tutorial_figures/f_o_nothresh.png)
6. To threshold the significance maps at reasonable (uncorrected) values, we usually set the "min" parameter to `3` and the "max" parameter to `5`. This means that all p-values less-significant than 10^-**3** will be ignored, and all values more-significant than 10^**-5** will be shown in yellow, with the red-yellow gradient between these values. You should see those "min" and "max" settings on the gray panel to the left of the image.
7. Once you set those thresholds, the image should look like this: ![f_o_thresh](https://github.com/samhutch511/kanwisher_lab_fmri/blob/main/tutorial_figures/f_o_thresh.png)
8. Surf through the image (with the green slider) until you see a slice that looks like this: ![FFA](https://github.com/samhutch511/kanwisher_lab_fmri/blob/main/tutorial_figures/ffa.png)
9. That yellow blob on the left-ventral part of the brain is [the fusiform face area, or the FFA](https://pubmed.ncbi.nlm.nih.gov/9151747/)! The larger yellow blob a little higher on the brain is the part of the superior temporal sulcus that responds to dynamic videos of faces, or [the faceSTS](https://www.sciencedirect.com/science/article/pii/S1053811911003466)!
> * Note that because of radiological convention, the left-right axis is flipped in these images—you should see a little "R" on the left border of the image, and a "L" on the right border. So, the FFA we're looking at is really in the right hemisphere, but shows up here on the left side.

#### Visualizing the language network on the surface
10. Close both the `sig.nii.gz` and the `orig.mgz` volumes with the red X, same as before.
11. Open the left-hemisphere inflated surface.
12. Now, confusingly, instead of loading another surface to visualize the aud.sm3.all.lh results, we have to load them as an "Overlay." To do that, click on the "Overlay" dropdown menu in the gray left-hand column and select "Load generic," then click on the file icon. This will take you to the familiar menu, and navigate to `vols_aud` --> `kaneff01` --> `bold` --> `aud.sm3.all.lh` --> `FP-NW`, and double-click on `sig.nii.gz`. Then click "Okay" to load the map. It should look something like this: ![fp_nw_nothresh](https://github.com/samhutch511/kanwisher_lab_fmri/blob/main/tutorial_figures/fp_nw_nothresh.png)
13. Below the "Overlay" dropdown menu, you should now see a button called "Configure." Click on this, and it should open a menu that looks like this: ![configure](https://github.com/samhutch511/kanwisher_lab_fmri/blob/main/tutorial_figures/configure.png)
14. Now, just like in the volume data, set the "min" to `3` and the "max" to `5`. Then click "Apply," and close the "Configure" window. Now, your surface map should look like this: ![fp_nw_thresh](https://github.com/samhutch511/kanwisher_lab_fmri/blob/main/tutorial_figures/fp_nw_thresh.png)
15. This red-yellow collection of blobs is [the core language network](https://www.pnas.org/doi/10.1073/pnas.1112937108)!
> * For a more recent review of the language network, see [this paper](https://pubmed.ncbi.nlm.nih.gov/38609551/)!

## From brain maps to bar plots (second-level analyses)
### Transforming parcels to subject-space
1. To be more confident that the activations we see in the contrast analyses really correspond to the functional regions of interest that we think they do, we need to intersect these activation maps with group **parcels**. Parcels are maps derived from the contrast activations of many individual subjects. Parcel maps are usually binary, with values of 1 in voxels where the functional region of interest can be expected to lie and 0 everywhere else. Since these maps are aggregated data from many subjects, they exist (by default) in a **template-space**, and we need to convert the parcel from that template space to the individual **subject-space** for each of our subjects. Basically, since not everyone's brain is exactly the same, we need to learn how to warp this probabilistic parcel map to each person's unique brain. This involves some pretty intense linear algebra (recall that all these brain images are represented as matrices) that I won't get into here, but this is the logic underlying what we're doing.
2. This process will use three scripts, which you can find in `/orcd/data/ngk/001/shared/samhutch_fmri_scripts`: `parcel_transform_final.m`, `convert_parcels_final.m`, and `run_transform.sh`.
3. Similar to the anatomical reconstruction step above, **we won't actually run these scripts now**, since they take several hours to complete. Instead, we'll just copy some already-transformed parcels over to our `{project_dir}`. But first, some notes on what the scripts actually do if we were to run them:
> * `parcel_transform_final.m`: this script creates the transformation matrices from the template-spaces to the subject-space, by taking in a subject name and a "run number" between 1 and 3, which corresponds to the space you're converting from. The three template-spaces: 1) CVS space, 2) CVS-in-MNI152 space, and 3) fsaverage-space. When these complete, you should have three corresponding transformation files: 1) `{project_dir}/parcels/subj2cvs_tform/{subj}/final_CVSmorph_tocvs_avg35.m3z`, 2) `{project_dir}/parcels/subj2cvsmni152_tform/{subj}/final_CVSmorph_tocvs_avg35_inMNI152.m3z`, and 3) `{project_dir}/parcels/fsavg2subj_tform/{subj}/mni152_to_func.lta`. **You never actually run this script yourself, instead it is called by `run_transform.sh`!**
> * `run_transform.sh`: this is a bash script for actually running these transforms as a submitted batch job (since the first two runs take hours), similar to how we submitted the recon job above. The script takes a subject name and a run number to pass to `parcel_transform_final.m`. You can submit parallel jobs with run numbers 1 and 2 (to do the CVS and CVS-in-MNI152 transformations), but both of these have to complete before you can submit a job with run number 3 (to do the fsaverage transformation).
> * `convert_parcels_final.m`: this script takes in a subject name and actually creates the subject-space parcels. You can just run this in an interactive Matlab session; it will take around 30 minutes to complete. When this successfully completes, you should see your parcels under: `{project_dir}/parcels/{subj}/`.
> * **Note:** there are a couple of places in both the Matlab scripts that reference files under the `vols_vis` directory; for other projects, you'll just need to change the `vis` part to the name of your contrast directories and the rest should work fine! This is because we need to point the transforms to some functional data.
4. To copy the parcels to your `{project_dir}`, run the following command: `scp -r /orcd/data/ngk/001/projects/efficient_localizer/parcels {project_dir}`
5. We can visualize these subject-space parcels with `freeview` just like regular volume maps to make sure the process ran correctly. Try visualizing the `rh.ffa_functional.nii.gz` parcel as a heatmap volume, under `{project_dir}/parcels/kaneff01/julian_parcels/`. It should look like this: ![ffa_parcel](https://github.com/samhutch511/kanwisher_lab_fmri/blob/main/tutorial_figures/ffa_parcel.png)
> * Try overlaying the Faces > Object contrast results here—you should find that the majority of the positive activation we saw in the ventral temporal cortex earlier lands within this parcel!
> * These ventral visual pathway parcels are called the Julian parcels, named for [this paper](https://pubmed.ncbi.nlm.nih.gov/22398396/).

### Loading and plotting data with Python
1. To do the rest of our quantitative analyses, we'll be using Python. I prefer to do this in a Jupyter Lab session, which you can [follow the instructions here](https://orcd-docs.mit.edu/recipes/jupyter/#port-forwarding) to set up on Engaging. **Make sure you start the lab session from your `{project_dir}`!**
2. I've written a template notebook with a lot of helpful functions, called `froi_analyses.ipynb`, which you can find in `/orcd/data/ngk/001/shared/samhutch_fmri_scripts`. Go ahead and copy this notebook to your `{project_dir}/scripts` directory and open it with Jupyter.
3. The notebook uses the `nibabel` package to load the `.nii.gz` files created by Freesurfer as numpy arrays, which makes it super easy to do stats and incorporate with other packages.
4. Go ahead and run each cell, reading the comments carefully to understand what's going on. In brief, we want to select voxels from **even runs** of our experiment which show a significant Faces > Objects contrast, then measure their responses during the **odd runs** of our experiment. This helps us stay statistically legitimate and avoid double-dipping (i.e. makes sure we're not just measuring noise)!
5. By the end, you should end up with these plots, showing us that we have indeed found the FFA in our subject! ![bar_thresh](https://github.com/samhutch511/kanwisher_lab_fmri/blob/main/tutorial_figures/bar_thresh.png) ![bar_topn](https://github.com/samhutch511/kanwisher_lab_fmri/blob/main/tutorial_figures/bar_topn.png)
