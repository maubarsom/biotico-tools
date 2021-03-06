#Make parameters
SHELL := /bin/bash

R1:= $(wildcard ../reads/*R1*.fastq.gz)
R2:= $(wildcard ../reads/*R2*.fastq.gz)

output_basename := P431_102

#Logging
log_name := $(output_basename)_$(shell date +%s).log
log_file := >( tee -a $(log_name) >&2 )

#Run params
threads:=16
adaptor_file:= adapters/truseq_custom.fa

#SGA parameters
sga_ec_kmer := 41
sga_cov_filter := 2

#Binary paths
TRIMMOMATIC:= java -jar /labcommon/tools/Trimmomatic-0.32/trimmomatic-0.32.jar

#Output basenames
nesoni_1 := nesoni_def
nesoni_2 := nesoni_custadapt
nesoni_3 := nesoni_match15mm2
nesoni_4 := nesoni_match15mm2_custadapt
nesoni_5 := nesoni_noadapt
trimmy_1 := trimmy
sga_1:= sga

nesoni_trimmy := nesoni_def_trimmy
nesoni_prinseq := nesoni_def_prinseq

#Output name generators (notice the = instead of := to set the appropriate directory)
nesoni_out_prefix = $(dir $@)$(output_basename)
trimmomatic_out = $(addprefix, $(dir $@)$(output_basename)_ , $(addsuffix .fq.gz R1 R1_single R2 R2_single ))

qf_suffixes:= R1.fq_fastqc.zip R2.fq_fastqc.zip sga.preqc sga.k17.hist sga.k17.hist.pdf

#Delete produced files if step fails
.DELETE_ON_ERROR:

#Avoids the deletion of files because of gnu make behavior with implicit rules
.SECONDARY:

all: $(addprefix $(nesoni_1)/$(output_basename)_, $(qf_suffixes))

#Folders
$(nesoni_1) $(nesoni_2) $(nesoni_3) $(nesoni_4) $(trimmy_1) $(nesoni_trimmy) $(nesoni_prinseq):
	mkdir -p $@

#*************************************************************************
#Calls to trimmers
#*************************************************************************
$(nesoni_1)/$(output_basename)_R%.fq.gz: $(R1) $(R2) | $(nesoni_1)
	nesoni clip --homopolymers yes --quality 20 --length 75 --out-separate yes \
		$(nesoni_out_prefix) pairs: $^ 2>> $(log_file)

$(nesoni_2)/$(output_basename)_R%.fq.gz: $(R1) $(R2) | $(nesoni_2)
	nesoni clip --adaptor-file $(adaptor_file) --homopolymers yes --quality 20 --length 75 \
		--out-separate yes $(nesoni_out_prefix) pairs: $^ 2>> $(log_file)

$(nesoni_3)/$(output_basename)_R%.fq.gz: $(R1) $(R2) | $(nesoni_3)
	nesoni clip --match 15 --max-errors 2 --homopolymers yes --quality 20 --length 75 \
		--out-separate yes $(nesoni_out_prefix) pairs: $^ 2>> $(log_file)

$(nesoni_4)/$(output_basename)_R%.fq.gz: $(R1) $(R2) | $(nesoni_4)
	nesoni clip --adaptor-file $(adaptor_file) --match 15 --max-errors 2 --homopolymers yes \
		--quality 20 --length 75 --out-separate yes $(nesoni_out_prefix) pairs: $^ 2>> $(log_file)

$(trimmy_1)/$(output_basename)_R%.fq.gz: $(R1) $(R2) | $(trimmy_1)
	$(TRIMMOMATIC) PE -threads 16 -phred33 -trimlog trimmomatic.log $^ $(trimmomatic_out) \
		ILLUMINACLIP:$(adaptor_file):2:30:10 MINLEN:75 2>> $(log_file)

#*************************************************************************
#Mixed quality filtering
#*************************************************************************
$(nesoni_trimmy)/R%.fq.gz: $(nesoni_1)/R1.fq.gz $(nesoni_1)/R2.fq.gz | $(nesoni_trimmy)
	$(TRIMMOMATIC) PE -threads 16 -phred33 -trimlog trimmomatic.log $^ $(trimmomatic_out) \
		ILLUMINACLIP:$(adaptor_file):2:30:10 MINLEN:75 2>> $(log_file)

$(nesoni_prinseq)/cosillo.fq: $(R1) $(R2) | $(nesoni_prinseq)
	prinseq-lite.pl -fastq $(R1) -fastq2 $(R2) -params prinseq_params.txt 2>> $(log_file)

#*************************************************************************
#SGA PREQC and QC steps
#*************************************************************************
#PREQC: First, preprocess the data to remove ambiguous basecalls
%_sga.fq: %_R1.fq.gz %_R2.fq.gz
	sga preprocess --pe-mode 1 -o $@ $^ 2>> $(log_file)

%_sga.fq: %_R1.fq %_R2.fq
	sga preprocess --pe-mode 1 -o $@ $^ 2>> $(log_file)

%_sga.fq: %_R1_001.fastq.gz %_R2_001.fastq.gz
	sga preprocess --pe-mode 1 -o $@ $^ 2>> $(log_file)

# PREQC: Build the index that will be used for error correction
# Error corrector does not require the reverse BWT
%_sga.sai: %_sga.fq
	cd $(dir $@) && sga index -a ropebwt -t $(threads) --no-reverse $(notdir $^) 2>> $(log_file)

# Perform error correction with a 41-mer.
# The k-mer cutoff parameter is learned automatically
sga_qc/$(sga_1).k$(sga_ec_kmer).ec.fq: %.k$(sga_ec_kmer).ec.fq : %.fq %.sai
	sga correct -k $(sga_ec_kmer) --discard --learn -t $(threads) -o $@ $< 2>> $(log_file)

# Index the corrected data
sga_qc/$(sga_1).k$(sga_ec_kmer).ec.sai: %.k$(sga_ec_kmer).ec.sai : %.k$(sga_ec_kmer).ec.fq
	sga index -a ropebwt -t $(threads) $^ 2>> $(log_file)

# Remove exact-match duplicates and reads with low-frequency k-mers
sga_qc/$(sga_1).k$(sga_ec_kmer).ec.filter.pass.fa: $(sga_1).k$(sga_ec_kmer).ec.fq $(sga_1).k$(sga_ec_kmer).ec.sai
	sga filter -x $(sga_cov_filter) -t $(threads) --homopolymer-check \
		--low-complexity-check $< 2>> $(log_file)

#*************************************************************************
#QUALITY VERIFICATION
#*************************************************************************
###########FASTQC############
%.fq_fastqc.zip: %.fq.gz
	fastqc --noextract -k 10 $^ 2>> $(log_file)

%.fq_fastqc.zip: %.fq
	fastqc --noextract -k 10 $^ 2>> $(log_file)

##########SGA PREQC###########
#Depends on the rules for creating the index
#Run SGA preqc
%.preqc: %.fq %.sai
	sga preqc -t $(threads) --force-EM $< > $@ 2>> $(log_file)
	cd $(dir $@) && sga-preqc-report.py $(notdir $@)

##############JELLYFISH 1############
# %_jf_17mer/k17_000: %.fq
# 	mkdir -p $*_jf_17mer
# 	jellyfish count -s 8G -C -m 17 -t $(threads) -o $(dir $@)k17 $^ 2>> $(log_file)

# %_jf_17mer/k17.jf: %_jf_17mer/k17_000
#	if [ `ls $(dir $^) | wc -l` -gt 1 ]; then jellyfish merge -o $@ $(dir $<)k17_* 2>> $(log_file); else ln -s $(dir $^ )/* $@; fi 	

# %.k17.hist: %_jf_17mer/k17.jf
# 	jellyfish histo -t $(threads) $^ > $@ 2>> $(log_file)

##############JELLYFISH 2############
%_jf_17mer/k17.jf: %.fq
	mkdir -p $*_jf_17mer
	jellyfish2 count -s 8G -C -m 17 -t $(threads) -o $@ $^ 2>> $(log_file)

%.k17.hist: %_jf_17mer/k17.jf
	jellyfish2 histo -t $(threads) $^ -o $@ 2>> $(log_file)

%.k17.hist.pdf: %.k17.hist
	Rscript plot_kmer_histogram.R $^ $@

# %_jf_31mer/k31_000: %.fq
# 	mkdir -p $*_jf_31mer
# 	jellyfish count -s 8G -C -m 31 -t $(threads) -o $(dir $@)k31 $^ 2>> $(log_file)

# %_jf_31mer/k31.jf: %_jf_31mer/k31_000
# 	jellyfish merge -o $@ $(dir $<)k31_* 2>> $(log_file)

# %.k31.hist: %_jf_31mer/k31.jf
# 	jellyfish histo $(threads) -f $^ > $@ 2>> $(log_file)


.PHONY: clean
clean:
	-rm *.fq.gz
	-rm *.fq_fastqc.zip #Fastqc
	-rm *.log #Makefile log
	-rm *.log.txt #Nesoni log
