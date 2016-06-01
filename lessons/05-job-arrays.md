------------

> ## Objectives
>
> * Demonstrate how to run several jobs in succession using a for loop
> * Demonstrate how to run several jobs simultaneously using a PBS job array

------------

Often, you will want to run the same analysis on several samples. Rather than
submit a job for each of these samples, you would like to submit one job and
apply it to all of your samples. Depending on the requirements of the task, you can either write your script in a for loop or
submit a PBS job array.


# For loops

If your analysis is memory intensive, i.e. requires close to your full allocation,
you might use a for loop. For example, if you are assembling several
metagenomes, it would be best to assemble each one at a time, so that you can 
allocate all 40 processors from the Denef fluxm account to each sample. 

Here is an example of a shell script that looks for directories that start with
"Sample_" and then runs an analysis called my-analysis.sh on a fasta file in each
of those directories. 

```{r}
for i in $(find -maxdepth 1 -type d -name "Sample_*"); do
    myPath=${i}
    echo -e "[`date`]\tStarting analysis on ${i}"
    cd $myPath
    bash ./my-analysis.sh my.fasta
    cd - 
    echo -e "[`date`]\tFinished with ${i}"
done

echo "[`date`]\tDone!"

```

# Job arrays
For loops allow you to process many samples sequentially, but it would really speed
up the process if you could submit the same job on many samples at the same time. 
This is exactly what a PBS job array does.
Particularly, if your analysis is not that memory intensive, i.e. your allocation is large
enough to run several jobs at once, you will definitely want to use a job array.
 

To submit a job array, you only need to add one line to your PBS script with the `-t` argument. The `-t` argument takes a set of indices for your job array. For example, if you wanted to run your script on 5 samples:

```{}
#PBS -t 1-5
```

PBS will then store a variable, `$PBS_ARRAYID`, which takes on values from the specified range (1-5 in this case). If we then started our job with `qsub`, 5 different instances
of the job would be executed, each uniquely identified by a value from `$PBS_ARRAYID`. 

We can use the `$PBS_ARRAYID` variable to assign samples to our job array.
The way I like to do this is to create a file called job.conf that contains
the name of each sample directory on a separate line. It might look like this:

```{}
Sample_1
Sample_2
Sample_3
Sample_4
Sample_5
```

Then, I can use the following line to assign a different 
sample directory for each element of my job array:

```{}
sample=$(sed "${PBS_ARRAYID}q;d" job.conf)
```
This line extracts the nth line of job.conf where n is our `$PBS_ARRAYID` number.
This means that Sample_3 will run in the 3rd index of our job array. 


#### Restricting arrays

If you want to submit a job array with a large number of samples, but you
only want a few of them to run at a time, you can specify this with a `%`. 
For example the following line will run an array with 20 jobs, but will only
run 5 of them at a time:

```{}
#PBS -t 1-20%5
```


#### Job array example:

Below is an example of a PBS script that will execute an analysis
using the CRASS crispr-finding software 

```{}
####  PBS preamble
#PBS -N crass
#PBS -m abe

#PBS -l nodes=1:ppn=2,mem=50gb,walltime=200:00:00
#PBS -V

#PBS -A vdenef_fluxm
#PBS -l qos=flux
#PBS -q fluxm
#PBS -t 1-7
#PBS -j oe

####  End PBS preamble

#  Show list of CPUs you ran on, if you're running under PBS
if [ -n "$PBS_NODEFILE" ]; then cat $PBS_NODEFILE; fi

#  Change to the directory you submitted from
if [ -n "$PBS_O_WORKDIR" ]; then cd $PBS_O_WORKDIR; fi
pwd


# required module: lsa/crass

# Run CRASS script
bash ./crass-analysis.sh ${PBS_ARRAYID}


```
Notice that here I pass `$PBS_ARRAYID` as an argument to my crass-analysis.sh
script. 

The crass-analysis.sh script looks like this:

```{}
arrayid="$1"
sample=$(sed "${arrayid}q;d" job.conf)

cd ${sample}
echo -e "[`date`]\t Running CRASS on $sample"
crass dt_int.fasta
echo -e "[`date`]\t Finished with $sample"


```
In the first line `$PBS_ARRAYID` gets assigned to the variable `$arrayid`. In the second line
`$arrayid` is used to assign the correct sample directory name to `$sample`. 

#### Checking up on your array job
You can use all of the same commands (e.g., `qstat`, `qdel`, `qpeek`) with a job array as regular job. However, you will need to use bracket indexing to select a job from the array.

For example, for a job with id 19957691, I could peek at the output of the second
sample with this command:

```{r}
qpeek 19957691[2]

```

If I wanted to cancel all of the jobs in my job array I could do this:
```{r}
qdel 19957691[]

```
