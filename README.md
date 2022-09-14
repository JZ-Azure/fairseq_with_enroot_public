# Steps to run fairseq using enroot with slurm

Rescale CC cluster HPC partition  
First edit the cluster from CC portal and increase max cores to N\*96, where N is the number of nodes.  
Then on the scheduler node, run the command below
```bash
cyclecloud connect scheduler -c slurmcycle
```
```bash
/opt/cycle/slurm/cyclecloud_slurm.sh scale
/opt/cycle/slurm/cyclecloud_slurm.sh gres_conf >& /sched/gres.conf
/opt/cycle/slurm/resume_program.sh NODELIST
```

Before you can `enroot import`, you need to authenticate first. 
[Nvidia API Key Page](https://ngc.nvidia.com/setup/api-key)

On the worker node, run
```bash
mkdir -p $HOME/.config/enroot/
vim $HOME/.config/enroot/.credentials
```
```bash
cat $HOME/.config/enroot/.credentials 
```
```bash
machine nvcr.io login $oauthtoken password YOUR_KEY
```

Import docker file and make changes
```bash
enroot import docker://nvcr.io/nvidia/pytorch:21.10-py3
enroot create --name pytorch nvidia+pytorch+21.10-py3.sqsh
enroot list
enroot start --root --rw --mount .:/workspace pytorch
```
```bash
bash image.config
```
Exit the container and export the updated version
```bash
enroot export --output fairseq.sqsh pytorch
enroot create --name fairseq fairseq.sqsh
enroot list
```

SLURM script to run fairseq training using enroot and pyxis
```bash
#!/bin/sh
#SBATCH --job-name=fairseq_moe
#SBATCH --output=job.%J.out
#SBATCH --error=job.%J.err
#SBATCH --ntasks-per-node=1
#SBATCH --mem=0
#SBATCH --gpus-per-node=8
#SBATCH --cpus-per-task=96

SLURM_PINNING="--cpu-bind=mask_cpu:ffffff000000,ffffff000000,ffffff,ffffff,ffffff000000000000000000,ffffff000000000000000000,ffffff000000000000,ffffff000000000000"

EXECUTE_SCRIPT="launch.sh"
CONTAINER_IMAGE="/shared/home/hpcadmin/fairseq.sqsh"

srun $SLURM_PINNING --container-image $CONTAINER_IMAGE --container-mounts .:/workspace $EXECUTE_SCRIPT
```

Submit to different number of nodes.
```bash
sbatch -N 2 submit
```
