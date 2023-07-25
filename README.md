# Integrative Genome Analysis of Fusarium graminearum and Associated Toxin Genes
## Assembly
### Combining files
In the case of Nextseq500 Paired-end reads, every sample is made of 4 files for both forward (R1) and reverse (R2) reads (8 files per sample), which should be combined into two files: one containing all forward reads, and another containing all reverse reads.  
This can be done using:  
A loop-
```bash
for x in *<string>; do cat $x >> <combined.fastq>; done
```
Paremeter Expansion-
```bash
x="original_string_removed"
new_variable="${x%_removed}added"
echo $new_variable
```
Understanding these two tools is vital to avoid single-file manipulation and to iterate your commands over all your data quickly and easily.  
### Trimming
Trimming was achieved using Trimmomatic set to nextera adapters (adepters will differ with library preparation.)  
This was done with:
```bash
nohup bash -c 'for x in [*R1]; do trim_galore --paired --nextera "$x" "${x%[R1]}[R2]" -o [output/path]; done' &
```
Since trimming can be a lengthy procedure, 'no hangup' (nohup) was used. Nohup both allows a given command to run in the background, and prevents termination when when the terminal session ends, allowing commands to be run for days at a time, while appending the standard output to the nohup.out file.  
This is done using:
```bash
nohup <command> &
```
Note: Sometimes, a tool won't run with nohup, even when working without it. This can be fixed by writing a script, and executing it with './script_name', or by adding 'bash -c' inside nohup, but before the command.
```bash
nohup bash -c 'for x in *.fasta; mv "$x" "output/${x%.fasta}_out.fasta"; done' &
```
Note that the command is inside single quotes, and every path containing a called variable is in double quotes.
### Assembly with SPAdes
Several assemblers were tested, which will be discussed in the next section (QC: Assembly Quality), but for now, we will proceed with assemblage.  
SPAdes was ran using:
```bash
nohup bash -c 'for x in [*R1]; do spades.py -1 "$x" -2 "${x%1}2" --isolate -o [output/path].assembled; done' &
```
Each pair of files will generate one output directory in the designated folder. The final assembly is in the file 'contigs.fasta' within the given directory.
### QC: Assembly Quality
Once you have a file with all your contigs, the next step will be to extract QC data to understand what the assembler did well, and where it failed.
To do this, QUAST was used:
```bash
for x in *.assembled; do quast.py -o <output (don't forget to number Using parameter expansion)> --fungus $x/contigs.fasta; done
```
After running QUAST, rename all the report.tsv files to include the isolate number, and combine them all using:
```bash
for x in *.report.tsv; do echo Isolate ${x%.report.tsv} >> combined_report.tsv; cat $x >> combined_report.tsv; done
```
This will only work if all reports are named '#.report.tsv', but if very important for graph generation, as the order which the files are called in is not always continuous or ascending. This command combined all the reports into a single file, spacing them with the cooresponding isolate number, for easy identification. This file can be imported into Excel or Google Sheets, where the addition of a filter collumn makes searching for the desired statistics a trivial task.
![image](https://github.com/hugowicks/Integrative-Genome-Analysis-of-Fusarium-graminearum-and-Associated-Toxin-Genes/assets/140028237/87f78d3b-5997-4648-a268-f099c88b3d2b)
These are some useful statistics to compare assembly quality, I recommend searching N50 and L50.
### Gene Alignment
To search for gene homology in the assemblies, Blast was used.  
First, a local database was constructing using a combined assembly file.  
Note: If constructing a phylogenetic tree of the blast hits later on, it's very important to identify the contig number, and which isolate it belongs to. Because SPAdes doesn't print the input files' name into each contig's identifier line, we will have to do that.  
It can be done using:
```bash
output_file="combined_sequences.fasta"
for fasta_file in *.assembled; do
    source_file="${fasta_file%.assembled}"
    awk -v source_file="$source_file" '/^>/ {$0 = ">" source_file "_" substr($0, 2)} 1' "$fasta_file" >> "$output_file"
done
```
This will work if your files are named #.assembled, but you may easily change this command to your file names instead of renaming them.  
To construct the database, we use:
```bash
makeblastdb -in combined.fasta -dbtype nucl -out SPAdes.db
```
After downloading the reference files for all the desired genes, we generate hits using:
```bash
nohup bash -c 'for x in *.fasta
do blastn -query "$x" -db SPAdes.db -out "blast.out/${x%fasta}results" -outfmt "6 sseqid sseq" -evalue 1e-30;
done' &
```
This command must be ran with the database files in your working directory, where the *.fasta files are the desired genes. The '-outfmt' flag generates a tabular format output; 'sseqid' and 'sseq' retrieve only the subject identifier line, and the subject sequence, which we need for generating trees.  
To convert the blast output files to fasta format, we use:
```bash
for x in *.results; do awk -F'\t' '{print ">"$1"\n"$2}' $x > ${x%.results}.seq; done
```
### MSA (Multiple sequence alignment)
Before generating trees, the files must be aligned, as this may be too cumbersome for a software to perform locally. MSA was done with Clustal Omega using the following code:
```bash
nohup bash -c 'for x in *.seq; do clustalo --dealign --force -i "$x" -o "${x%.seq}.MSA.fasta"; done' &
```
Now, the files should be ready for tree generation.
### Generating trees
The resulting fasta files work with any tree generating program (I hope), but MEGA was used for its easy navigation, visualization, and setting customization.
Maximum Likelihood trees were generated using the 200 bootstrap replications, and the Kimura 2-parameted model.  
The resulting trees can be exported as image files, or newick (.nwk) files to be imported into other tree visualization software.
