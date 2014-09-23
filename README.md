Andrew T. Martens

-----------------

This is a summary of how to use the code to perform the data analysis on the Linux (BASH) command line.
Original work was performed on a computer running Fedora 20 Linux in 2013-2014.
It helps to put the perl programs, genomes files, and so on in their own directories, but is not required.

1. Get a Sequence Read Archive (SRA) file from the Gene Expression Omnibus. Example: SRR1067765.sra

2. Convert the SRA files to FASTQ files

  fastq-dump ./SRR1067765.sra -O .

3. Trim the ends

  fastx_clipper -Q33 -a CTGTAGGCACCATCAAT -l 21 -c -n -v <SRRxxx.fq >SRRxxx.trimmed.fq

4. Build the Bowtie rRNA/tRNA database

  bowtie-build -f rrna-seqs.fa rrna_seqs

5. Concatenate the trimmed FQ files

  cat SRR1067765/SRR1067765.trimmed.fq SRR1067766/SRR1067766.trimmed.fq SRR1067767/SRR1067767.trimmed.fq SRR1067768/SRR1067768.trimmed.fq > redo_calcs/Sequencing_Reads/trimmed-data.fq

6. Use rRNA/tRNA database to filter out these sequences with Bowtie 1.0

  bowtie --quiet -p 8 -l 23 --un=Sequencing_Reads/norrna.fq rrna_seqs -q Sequencing_Reads/trimmed-data.fq > /dev/null

7. Align to genome. This means allow up to 2 mismatches in the first 23 nucleotides, no multi-mappers.

  bowtie -S -p 8 -l 23 -m 1 --sam-nohead e_coli_MG1655 -q norrna.fq > aligned.SAM

  Result:
  
  reads processed: 158173518
  
  reads with at least one reported alignment: 95854989 (60.60%)
  
  reads that failed to align: 54522442 (34.47%)
  
  reads with alignments suppressed due to -m: 7796087 (4.93%)
  
  Reported 95854989 alignments to 1 output stream(s)

8. Only keep the parts that matched the genome. (0 for unmapped, 1 for multi mapped)

  grep -v -E 'XM:i:[01]' aligned.SAM > matched.SAM

  Left with 95854989 reads

9. Make a "simplified" SAM file.

  Note: does not properly account for reads which are the "wrong" direction (these events are rare).

  ./Perl_Programs/fix_SAM_multi_chromosome.pl Genomes/ecoli_chromosome/ Sequencing_Reads/matched.SAM Sequencing_Reads/fps.simple

10. Process the PTT file to remove any ORFs which have in-frame stop codons.

  ./Perl_Programs/ptt_no_stop_multi_chromosome.pl Genomes/ecoli_chromosome/ Genomes/ecoli_ptt/ Genomes/ecoli_ptt_nostops/

11. Filter out any sequences which are not "elongation" sequences.

  a. Must be in protein coding sequences, in the PTT file (with no in frame stops)

  b. Must be within 10 codons from the ends
  
  Note: this runs into problems with overlapping ORFs, which fortunately are uncommon.

  ./Perl_Programs/filter_only_elong_footprints_multi_chromosome.pl Genomes/ecoli_ptt_nostops Sequencing_Reads/fps.simple Sequencing_Reads/elong.simple

12. Make heatmaps of extended footprint codon composition by length class. Frame matters.

  ./Perl_Programs/footprint_codon_composition_elongation_multi_chromosome.pl Genomes/ecoli_chromosome/ Genomes/ecoli_ptt_nostops/ Sequencing_Reads/elong.simple Plots/codon_heatmaps/ Plots/codon_heatmaps/len_tally.tsv

13. Extend the 5' ends to 45 nt, discard anything > 45 nt. Frame does not matter.

  ./Perl_Programs/footprint_extend_5prime_track_length_multi_chromosome.pl Genomes/ecoli_chromosome/ Genomes/ecoli_ptt_nostops/ Sequencing_Reads/elong.simple Sequencing_Reads/extended.simple

14. Make plots of AUGC content by position and length.

  ./Perl_Programs/extended_fps_augc_v2.pl Sequencing_Reads/extended.simple Plots/extended_augc/

15. Make GC heatmap plot.

  ./Perl_Programs/extended_fps_gc_content_v2.pl Sequencing_Reads/extended.simple Plots/gc_heatmap

16. Make 2d-heatmaps of footprint ends

  a. Find the endpoints and append to files (this means must manually delete output from a previous run)

  ./Perl_Programs/footprint_2d_heatmap-simple.pl Genomes/ecoli_chromosome/NC_000913.fna Genomes/ecoli_ptt_nostops/NC_000913.ptt Sequencing_Reads/elong.simple Plots/2d_heatmaps/endpoints/

  b. Tally the endpoints
  
  ./Perl_Programs/fp_coord_combine.pl Plots/2d_heatmaps/endpoints/ Plots/2d_heatmaps/endpoints_combined/

  c. Arrange them in a 2d matrix
  
  ./Perl_Programs/footprint_2d_heatmap_part2_nucleotide.pl Genomes/ecoli_ptt_nostops/NC_000913.ptt Plots/2d_heatmaps/endpoints_combined/ Plots/2d_heatmaps/output/

  Then, need to go manually in Gnuplot and make the plot with the appropriate x,y range etc.
  example:
  set datafile missing "-"
  set key off
  plot "rpsA-b0911.heat.txt" matrix with image

17. Compute the mRNA transcriptome codon counts:

  a. Use the same program as before to tally the ends of the sequencing reads. These are not footprints, but that's OK, for this application it doesn't matter. Must also simplify the SAM file, as before, with the fix_SAM perl program.
  
  ./Perl_Programs/footprint_2d_heatmap-simple.pl Genomes/ecoli_chromosome/NC_000913.fna Genomes/ecoli_ptt_nostops/NC_000913.ptt Sequencing_Reads/rnaseq.simple Plots/2d_heatmaps/mrna_endpoints/
  
  b. Take the endpoints and collapse them to 1-dimensional density over position information
  
  ./Perl_Programs/footprint_1d_density_codon_gnuplot.pl Plots/2d_heatmaps/mrna_endpoints/ Plots/1d_density/mrna/
  
  c. Compute the average density for each gene, the codon composition of each gene, and combine them for the transcriptome codon composition.
  
  ./Perl_Programs/mRNAseq_averages.pl Genomes/ecoli_ptt_nostops/NC_000913.ptt Genomes/NC_000913.ffn Plots/1d_density/mrna/ Plots/1d_density/mrna_rowstack Plots/1d_density/mrna_codon_totals.tsv Plots/1d_density/mrna_gene_avg_density.tsv

