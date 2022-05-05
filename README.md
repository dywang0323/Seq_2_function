# this is resptory is to use the metagenome sequences to map the interested functional genes

Major steps:
1. prepare the metagenome data
2. map the sequencing reads to the reference 
3. calculate the RPKM

# 1. Prepare the metagenome data (bbtools)
1) Trim adaptors, optionally, reads with Ns can be disregarded by adding “maxns=0” and reads with really low average quality can be disregarded with “maq=8”

command:

${BBMAP_PATHWAY}/bbduk.sh in=${dataset_pathway}/input.fastq
out=${output_pathway}/output.fastq 
ref=${BBMAP_PATHWAY}/resources/adapters.fa 
ktrim=r k=23 
mink=11 hdist=1 tpe tbo

2)Remove synthetic artifacts and spike-ins by kmer-matching
This will remove all reads that have a 31-mer match to PhiX, allowing one mismatch. The “outm” stream will catch reads that matched a reference kmers, “stats” will produce a report of which contaminant sequences were seen, and how many reads had them

command

${BBMAP_PATHWAY}/bbduk.sh in=${trimed_dataset_pathway}/trimed.fastq
out=${output_pathway}/output_u.fastq outm=${output_pathway}/output_m.fastq ref=${BBMAP_PATHWAY}/resources/sequencing_artifacts.fa.gz ref=${BBMAP_PATHWAY}/resources/phix174_ill.ref.fa.gz k=31 hdist=1 stats=stats.txt

3)error correction
Low-depth reads can be discarded here with the”tossdepth”, or “tossuncorrectable” flags
For very large datasets, “prefilter=1” or “prefilter=2” can be added to conserve memory

command:

${BBMAP_PATHWAY}/tadpole.sh -Xmx32g in= output_u.fastq out=output_tecc.fastq filtermemory=7g ordered prefilter=1

# Map the sequencing reads to the reference (pullseq & bowtie2 & shrinksam)
1) Select reads that sequence length > 100 (pullseq)

command:

{PATHWAY_PULLSEQ}/pullseq -i ${patway_input}/assembly_data.fastq -m 100
> ${pathway_output}/sequence_min100.fastq
Deinterleave paired reads (bbtools)
${BBMAP_PATHWAY}/reformat.sh in=${pathway_input}/output_tecc.fastq out1=${pathway_output}/read1.fastq out2=${pathway_output}/read2.fastq
2)read mapping (bowtie2)
Create bowtie2 index file

command:

${PATHWAY_BOWTIE2}/bowtie2 -build ${pathway_input}/sequence_min100.fastq sequence_min10
Sequence alignment

command:

${PATHWAY_BOWTIE2}/bowtie2 -x sequence_min100 -1 {pathway_input}/read1.fastq -2 ${pathway_input}/read2.fastq -S {pathway_output}/ alignment.sam -p 19

You can obtain the number of aligned reads in the output file and the number of total read can be obtained in the log file of assembly
Mapping rate = the number of aligned reads/ the number of total reads

# Calculate the RPKM (scripts in the folder)
1) Count Mapped Reads

command

${Shrinksam_pathway}/shrinksam -i {pathway_input}/ alignment.sam > {output_pathway}/mapped.sam
{pathway_script}/add_read_count.rb mapped.sam sequence_min1000.fastq > mapped.fasta.counted

grep -e > mapped.fasta.counted > mapped.counted.result
sed -i “s/>//g” mapped.counted.result
sed -i “s/read_count_//g” mapped.counted.result

2) RPKM Calculation

command:

Python ${pathway_script}/parse_rpkm.py ${pathway_input}/mapped.counted.result ${pathway_input}/total_reads
