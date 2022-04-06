Another Example - process a set of files

### Often we have a set of files we want to run through an application, arrays can run each file in a separate job with one script. Here are our files we wish to process:
```
$ ls -1 *.sam
sample1.sam
sampleA.sam
control.sam
```
The application we want to use is "samapp convert" which takes a sam format file and converts it into a bam format file. (in this example "samapp convert" is a fictitious command!)

samapp convert requires a option to tell it to output bam '-b' and it defaults to outputting to terminal, we want it to go to a file so will redirect output to a file ( > file.bam )

To run the command on one file would then require this format:
```
$ samapp convert -b sample1.sam > sample1.bam
```
To allow us to use a job array to perform this on phoenix hpc we need some scripting to generate our command for each sam file:
```
file=$(ls *.sam | sed -n ${SLURM_ARRAY_TASK_ID}p)
bamfile=$(basename -s .sam $file)    # remove .sam from file name and assign to bamfile
echo samapp convert -b ${bamfile}.bam $file 
```
When slurm runs a job array, it will cycle through the assigned job numbers, (array=1-2 will cycle from 1 to 2, array=4,5 will cycle through 4 then 5) and assign the job number to SLURM_ARRAY_TASK_ID. As we have three sam files, we would use array=1-3. Looking at our script the first line will be processed as:
```
SLURM_ARRAY_TASK_ID = 1   |  file=$(ls *.sam | sed -n ${SLURM_ARRAY_TASK_ID}p)  ->  file=$(ls *.sam | sed -n 1p)  -> file=sample1.sam
SLURM_ARRAY_TASK_ID = 2   |  file=$(ls *.sam | sed -n ${SLURM_ARRAY_TASK_ID}p)  ->  file=$(ls *.sam | sed -n 2p)  -> file=sampleA.sam
SLURM_ARRAY_TASK_ID = 3   |  file=$(ls *.sam | sed -n ${SLURM_ARRAY_TASK_ID}p)  ->  file=$(ls *.sam | sed -n 3p)  -> file=control.sam
```
Our second line of the script will be processed as:
```
bamfile=$(basename -s .sam $file)  -> bamfile=$(basename -s .sam sample1.sam)  ->   bamfile=sample1
```
and our scripts last line will run our command substituting the correct input and output filenames:
```
echo samapp convert -b ${bamfile}.bam $file   ->  echo samapp convert -b sample1.bam sample1.sam
```
Here is the slurm script- jobarray.sh:   (NOTE: we need to know the number of sam files, and update the array values)
```
#!/bin/bash
#SBATCH -p batch
#SBATCH -N 1
#SBATCH -n 1
#SBATCH --time=0-00:01:00
#SBATCH --mem=16GB
#SBATCH --gres=tmpfs:64G
# Notification configuration
#SBATCH --mail-type=NONE
#SBATCH --array=1-3

file=$(ls *.sam | sed -n ${SLURM_ARRAY_TASK_ID}p)
bamfile=$(basename -s .sam $file)    # remove .sam from file name and assign to bamfile
echo samapp convert -b ${bamfile}.bam $file
```
As this processes all sam files in the current directory, check this is what you want. Alternately run in a folder where you have placed the files requiring processing.

When the job is run, here are the task processes created:
```
#!/bin/bash
#SBATCH -p batch
#SBATCH -N 1
#SBATCH -n 1
#SBATCH --time=0-00:01:00
#SBATCH --mem=16GB
#SBATCH --gres=tmpfs:64G
# Notification configuration
#SBATCH --mail-type=NONE
#SBATCH --array=1-3

file=$(ls *.sam | sed -n 1p)
bamfile=$(basename -s .sam $file)   
echo samapp convert -b ${bamfile}.bam $file

	

#!/bin/bash
#SBATCH -p batch
#SBATCH -N 1
#SBATCH -n 1
#SBATCH --time=0-00:01:00
#SBATCH --mem=16GB
#SBATCH --gres=tmpfs:64G
# Notification configuration
#SBATCH --mail-type=NONE
#SBATCH --array=1-3

file=$(ls *.sam | sed -n 2p)
bamfile=$(basename -s .sam $file)   
echo samapp convert -b ${bamfile}.bam $file

	

#!/bin/bash
#SBATCH -p batch
#SBATCH -N 1
#SBATCH -n 1
#SBATCH --time=0-00:01:00
#SBATCH --mem=16GB
#SBATCH --gres=tmpfs:64G
# Notification configuration
#SBATCH --mail-type=NONE
#SBATCH --array=1-3

file=$(ls *.sam | sed -n 3p)
bamfile=$(basename -s .sam $file)   
echo samapp convert -b ${bamfile}.bam $file
```
### As this example just uses echo to create the output, (echo only for testing) which will be written to slurm logs, (one slurm log for each array job) we can test this on a set of any files. We can see how the SLURM_ARRAY_TASK_ID has been replace in each job task, and will then execute a different file (selected from the ls *.sam output) for each.

If you like you can test the process with the following commands:
```
$ mkdir jobarray
[]$ cd jobarray
[jobarray]$ touch sample1.sam
[jobarray]$ touch sampleA.sam
[jobarray]$ touch control.sam
```
put the slurm script into jobarray.sh with a text editor (copy from above), then submit it:
```
[jobarray]$ sbatch joabarry.sh
```
This will complete in a few seconds and place three slurm outputs in our directory:
```
[jobarray]$  ls -1
control.sam
joabarry.sh
sample1.sam
sampleA.sam
slurm-10212947_1.out
slurm-10212947_2.out
slurm-10212947_3.out
```
and checking the output, we see the commands created: (with echo removed from the scripts the command would be ran), {remember the "samapp" is not a real command}
```
[jobarray]$ cat slurm-10212947_3.out 
samapp convert -b sampleA.bam sampleA.sam

[jobarray]$ cat slurm-10212947_2.out 
samapp convert -b sample1.bam sample1.sam

[jobarray]$ cat slurm-10212947_1.out 
samapp convert -b control.bam control.sam
```
