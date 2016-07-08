------------

> ## Objectives
>
> * Give an example of a full job execution workflow with mothur


------------

In the following example we will run [mothur](http://www.mothur.org/), 
according to the [SOP](http://www.mothur.org/wiki/MiSeq_SOP), on a set of 
bacterial samples that have had their 16S rRNA gene sequences on a MiSeq.

# Setup
First, we need to get our sequences onto flux. Most likely, you had your 
samples sequenced at the UM Med School and they put your sequences on an MBox. 
Unless they have changed their setup, you will need to download these sequences
to a local machine and then use Globus (recommended) or Cyberduck (fine for only a few
samples) to transfer these files onto your FLUX account.

**IMPORTANT**: You should keep a safe version of these files in a long-term storage
location like a flux nfs drive. Then use Globus to transfer a copy to the Scratch
drive for processing. 

# Mothur batch script 

[Here](https://github.com/DenefLab/flux-tools/blob/gh-pages/scripts/mothur.batch) is a link to an example mothur batch file, following the SOP. Please refer
to the mothur wiki for an in depth explanation of the commands. 

You can copy it to your current flux location with:

```{r}
wget https://github.com/DenefLab/flux-tools/raw/gh-pages/scripts/mothur.batch

```
You will **definitely** need to edit the following lines of the file:

Change your input, output, and tempdefault locations.
Tempdefault signifies a location that mothur will search for file that are not in the
input directory. This is a good place to leave your databases for alignment/taxonomy.
```{r}
set.dir(input=/scratch/lsa_fluxm/michberr/HABS)
set.dir(output=/scratch/lsa_fluxm/michberr/HABS)
set.dir(tempdefault=/scratch/lsa_fluxm/vdenef/Databases/ssu_rRNA/Mothur/)
```

Change the **file** argument to a file that contains all your sample names 
and links to the forward and reverse reads. See the example stability.file in the SOP.
Set the **processors** argument to the number of processors you will request in your PBS script.
```{r}
make.contigs(file=habs.files, processors=30)
```

Change the **reference** argument to the name of the file you want to align to
```{r}
align.seqs(fasta=current, reference=silva.seed_v119.pcr.v4.unique.align)
```

Change the **reference** and **taxonomy** arguments to the files you are using for your analysis
```{r}
classify.seqs(fasta=current, count=current, reference=silva.nr_v119.pcr.v4.align, taxonomy=silva.nr_v119.tax, cutoff=60)
```

Change the **groups** argument to the names of the mock groups included in your run.
They should be separated by a '-'
```{r}
remove.groups(count=current, fasta=current, taxonomy=current, groups=Mock1-Mock2)
```

You **might** want to edit several other lines of this file based on the needs of
your analysis. 

------------

## Mothur PBS script

[Here](https://github.com/DenefLab/flux-tools/gh-pages/scripts/mothur.pbs) is a link to an example PBS script to execute the mothur batch file. 
You can copy it to your FLUX location the same way as the batch file.

```{r}
wget https://github.com/DenefLab/flux-tools/raw/gh-pages/scripts/mothur.pbs
```

You will **definitely** need to edit the `-M` line: 

```{r}
#PBS -M your-email

```

If you are not in the Denef lab, you will need to edit the `-A` line:
```{r}
#PBS -A your-account
```


You **might** want to edit the `-l` line with a different memory or walltime
allocation based on the size of your dataset. Usually, for ~350 samples of moderate
diversity, it takes less than 24 hours on 30 processors. 

# Run the job

To begin running mothur we need to load the mothur module.

```{r}
module load med
module load mothur
```

Now it's time to execute!
```{r}
qsub mothur.pbs
```

We can check our job with:
```{r}
qstat -u your-uniq-name

```

Most likely, your script will fail the first few times you try to run it. Usually
the error is related to an incorrect path or filename or setting an impossible allocation
in your PBS script. Read your output/error file (the file ending in .o and your jobid) to debug.

```{r}
cat mothur-output-file
```

# Special mothur considerations

Some mothur commands are designed to run very efficiently on a large number of processors.
This is where running your job on FLUX will save you a LOT of time over a local machine.
However, the OTU picking stage can only run on a single processor and this is usually the most time consuming part of
the process. The more diversity you have, the longer this will take. This is good to keep in mind
as you are weighing processor and time allocation. Adding 40 processors to your job will only
help to a point. You should still add sufficient time for all of your OTUs to be calculated.

The most frustrating issue with mothur I have encountered is when you end up with 
many strange unclassified OTUs that you know should really be classified. This took 
us in the Denef lab almost a year to figure out. It turns out that a few extra files
are generated when you run the classification step. You need to delete these files
in between mothur runs, or move your reference and taxonomy files to a new location,
or you will get strange and wrong results. These are the problemmatic files:


* silva.nr_v119.silva.nr_v119.pcr.v4.8mer.numNonZero      
* silva.nr_v119.silva.nr_v119.pcr.v4.8mer.prob    
* silva.nr_v119.tree.sum    
* silva.nr_v119.tree.train    
* silva.nr_v119.pcr.v4.8mer                                                                                


