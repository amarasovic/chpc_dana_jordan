## CHPC account 

Follow the steps here: https://www.chpc.utah.edu/userservices/accounts.php.

## SSH-ing to servers

Outside of the university network, you need to use VPN. For how to access SoC VPN, see [here](https://support.cs.utah.edu/index.php/misc/30-pa-vpn-setup#:~:text=Accessing%20the%20School%20of%20Computing's,Active%20Directory%20username%20and%20password).


SSH to `notchpeak` or `nothchpeak2` by entering: 
```
ssh uNID@notchpeak2.chpc.utah.edu
```
in a terminal. 

You will be prompted to give your uNID password. 


## Conda environments  

Download and install [Miniconda](https://docs.conda.io/en/latest/miniconda.html#linux-installers). On Mac, right-click on a suitable installer and select "Copy Link Address", then wget that link.  E.g,. on Linux 64-bit and python 3.8:

```
wget https://repo.anaconda.com/miniconda/Miniconda3-py38_23.5.2-0-Linux-x86_64.sh
bash Miniconda3-py38_23.5.2-0-Linux-x86_64.sh
```

Accept the license, install `miniconda` into the default location, initialize, exit, and ssh again.

Create and activate a Conda environment with a desirecd python version, e.g., 3.8: 

```
conda create -n <env_name> python=3.8
conda activate <env_name>
```

In a repo with code you're trying to run, you'll usually see a `requirements.txt` file with required packages. If so, you can run: 
```
pip install -r requirements.txt
```

You might get this error: `RuntimeError: CUDA error: no kernel image is available for execution on the device` if your cuda drivers and pytorch version do not match. I personally check cuda drivers by running `nvidia-smi` (upper right corner). You can go to https://pytorch.org/ and get a command for right drivers. E.g., for cuda drivers 11.6 I run: `pip install torch torchvision torchaudio --extra-index-url https://download.pytorch.org/whl/cu116`/

## Cuda drivers 

When SSH-ed on CHPC's servers/nodes, when you run 
`which nvcc` and cuda drivers are loaded, you'll see something like: `/uufs/chpc.utah.edu/sys/spack/linux-rocky8-nehalem/gcc-8.5.0/cuda-11.6.2-hgkn7czv7ciyy3gtpazwk2s72msbw6l2/bin/nvcc`

If not, you need to load them: 
* `module spider cuda` which cuda drivers are available through the module list 
* `module load cuda/11.8.0` load cuda 11.8
* `module list` to see that cuda is loaded 

If you didn't already, you need to install pytorch version for your cuda drviers' version, e.g., for 11.8: 
`pip3 install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu118`. Check https://pytorch.org/ for command you need (search "INSTALL PYTORCH").

## [CHPC](https://www.chpc.utah.edu/documentation/index.php) 

The list of nodes and corresponding GPU types can be found [here](https://www.chpc.utah.edu/documentation/guides/gpus-accelerators.php) under "GPU Hardware Overview > GPU node list".

**To view a list of accounts and partitions that are available to you, run command `myallocation`.**

SOTA GPUs are A100s. Check the comparison between GPUs [here](https://lambdalabs.com/gpu-benchmarks), and [nvidia](https://www.nvidia.com/en-us/data-center/a100/) says: 
> "A100 provides up to 20X higher performance over the prior generation and can be partitioned into seven GPU instances to dynamically adjust to shifting demands. The A100 80GB debuts the world's fastest memory bandwidth at over 2 terabytes per second (TB/s) to run the largest models and datasets". 


## Storage 

Model checkpoints can be huge so we shouldn't save them to our local directories. 

CHPC has scratch file systems (https://www.chpc.utah.edu/resources/storage_services.php) that are helpful. 

Make a directory in one of them: 

`mkdir /scratch/general/vast/<uNID>` 

and save your checkpoints and data there. 

Note however that "On these scratch file system, files that have not been accessed for 60 days are automatically scrubbed." 

## Tmux 

You need to use [tmux](https://github.com/tmux/tmux/wiki) to be able to get back to your experiments once you close your laptop or ssh connection breaks. [Tmux Cheat Sheet & Quick Reference](https://tmuxcheatsheet.com/). 

## Git 

For introduction to Git see [this](https://missing.csail.mit.edu/2020/version-control/). I make changes to my code locally, push them to a repo, pull the changes on server, and then run an experiment on server. 

## Other tips 

### Tip #1 

If you use [huggingface](https://huggingface.co/course/chapter1/1), downloaded models will be saved in a `.cache` folder in your home directory. It will quickly eat all of your home directory space, so what I do is: 

```
mkdir /scratch/general/vast/<uNID>/huggingface_cache
export TRANSFORMERS_CACHE="/scratch/general/vast/<uNID>/huggingface_cache"
```

You'll need to do this every time you run a job, so it's good to fix this in the code instead.

### Tip #2

Use `import pdb; pdb.set_trace()` or `breakpoint()` in python for debugging. 

### Tip #3 

Use `scancel <jobID>` to cancel your job.

## Queue jobs 

You need to write a batch script, something like this (pay attention to where you need to add your information):

```
#!/bin/bash
#SBATCH --account soc-gpu-np
#SBATCH --partition soc-gpu-np
#SBATCH --ntasks-per-node=32
#SBATCH --nodes=1
#SBATCH --gres=gpu:a100:1
#SBATCH --time=8:00:00
#SBATCH --mem=30GB
#SBATCH --mail-user=<your_email>
#SBATCH --mail-type=FAIL,END
#SBATCH -o <file_name>-%j

source ~/miniconda3/etc/profile.d/conda.sh
conda activate <env_name>

mkdir /scratch/general/vast/<your_unid>/huggingface_cache
export TRANSFORMER_CACHE="/scratch/general/vast/<your_unid>/huggingface_cache"
python run_summarization.py \
    --model_name_or_path t5-small \
    --do_train \
    --do_eval \
    --dataset_name cnn_dailymail \
    --dataset_config "3.0.0" \
    --source_prefix "summarize: " \
    --output_dir /scratch/general/vast/<uNID>/tst-summarization \
    --per_device_train_batch_size=4 \
    --per_device_eval_batch_size=4 \
    --overwrite_output_dir \
    --predict_with_generate
```

Save it as `<script name>.sh` and run: `sbatch <script name>.sh`.

## Interactive jobs 

Queuing in annoying if you're debugging. That's when interactive sessions are useful – you allocate a GPU for a certain amount of time, and it's yours until you exit. 

WARNING: Use sporadically. If you allocate a GPU but do not use it, you are wasting the resource! Until your interactive session is not over, other jobs are waiting. It's etiquette to queue jobs.

To use it, you can run something like:

1. `salloc -A soc-gpu-np -p soc-gpu-np -n 32 -N 1 --gres=gpu:a100:1 -t 1:00:00 --mem=40GB` 

2. `conda activate ...`

3. `export TRANSFORMER_CACHE="/scratch/general/vast/<your uNID>/huggingface_cache"`

4. `python ...`

## Checking jobs 

You can check the queue by running: `squeue` (for everything on CHPC). To see only your jobs in the queue you can run `squeue -u <your unid>`.

To log into the nodes where the job runs, you need to figure out the node on which  the job is running on (it will be in the output of the `squeue`),  shh to that node with

`ssh <your unid>@node<number>`.

Check how GPU memory is utilized by running: 

```
ml nvtop 
nvtop
```

If no GPU memory is being used that's a problem.

