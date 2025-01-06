# Single-cell sQTL workflow (2-stage): junction calling 

Updated 16 Dec 2024



## Getting Started 

### Prerequisites

python3, awk, cut

### Installing

Two more tools to be installed 

```
mkdir -p ~/software 
cd ~/software
git clone https://github.com/tibettiger/Junctions_quantification 
git clone https://github.com/davidaknowles/leafcutter
```

Add junctools to $PATH (for example, under ~/bin/)

```
ln -s Junctions_quantification/junctools ~/bin
```

Please test the software before proceeding to the pipeline. 



## Running the pipeline

Please run the following 4 steps one by one. 
Variables need to be defined accordingly (e.g. ${outDir} is the output directory)

### Extract intron junctions from BAM files (e.g. output by  Cell Ranger) 

For each bam file, 

\${exp} # identifier for each experiment

```
junctools junctions extract -a 8 -m 50 -M 500000 \
         -s RF ${bamFile} -o ${outDir}/junc/${exp}.junc_barcoded
```

### Prepare pseudo-bulk junction files (per cell type, per individual)

\${junc}=\${outDir}/junc/\${exp}.junc_barcoded

A sample metadata is in /example folder

```
awk -F'\\t' '{print $1 "\t" $3 "_" $4}' "${meta}" > "${meta}.tmp"

# Use awk to process the main file and split into groups
awk -F'\t' -v meta_tmp="${meta}.tmp" '
    BEGIN {
        while ((getline < meta_tmp) > 0) {
            BC = gensub(/-[0-9]+$/, "", "1", $1)
            groups[BC] = $2
        }
        close(meta_tmp)
    }
    {
        key = gensub(/-[0-9]+$/, "", "1", $13)
        if (key in groups) {
            group_file = "${outDir}/pseudobulk_junc/" groups[key] ".junc.tmp"
            print $0 >> group_file
        } else {
            print "No group found for " key
        }
    }
' "{$junc}"
```

Run this manually after all jobs complete 

```
for tmp_file in *.tmp; do
    final_file=$(basename "$tmp_file" .tmp)
    mv "$tmp_file" "$final_file"
done
```

### Cluster intron junctions

\${ct} \# cell type, as defined in the meta file

\${cohort} \# define the name of your cohort with an easily recognizable 

```
ls ${outDir}/pseudobulk_junc/*${ct}.junc | while read i 
do
    newname=`basename $i |  sed 's/\([0-9]*_[0-9]*\).*/\1/'`
    cut -f1-12 $i > ${outDir}/pseudobulk_junc_format/${ct}/${newname}.junc
done


ls ${outDir}/pseudobulk_junc_format/${ct}/*.junc > ${outDir}/pseudobulk_junc_format/juncfiles.${ct}.txt

mkdir -p ${outDir}/cluster/${ct}
cd ${outDir}/cluster/${ct} 
python ~/software/leafcutter/clustering/leafcutter_cluster_regtools.py -j ${outDir}/pseudobulk_junc_format/juncfiles.${ct}.txt 
       -m 0 -p 0 -l 500000 -o ${cohort}_cluster.${ct}
```



## Output

Send output count file to central analysis for joint junction quality control & filtering: 

\${cohort}_cluster.\${ct}_perind.counts.gz 



## Authors

* **Tian Chi** - *central analysis*
* **Zhang Yuntian** - *tool development* - [Junctools](https://github.com/tibettiger/Junctions_quantification)



## Acknowledgments

* Prof. Liu Boxiang



## Reference 

Tian, C., Zhang, Y., Tong, Y. et al. Single-cell RNA sequencing of peripheral blood links cell-type-specific regulation of splicing to autoimmune and inflammatory diseases. *Nat Genet* **56**, 2739–2752 (2024). https://doi.org/10.1038/s41588-024-02019-8


