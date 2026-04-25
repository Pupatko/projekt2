somatic variant calling

v tomto projekte analyzujeme dáta (FASTQ) od pacienta s rakovinou kosti (asi) , mame k dispozicii vzorky aj zdraveho aj nadoroveho tkaniva toho isteho pacienta, ideme teda hladat mutacie v nadore , potom identifikujeme geny , ktore su zodpovedne za vznik ochorenia


POZNAMKY:
- data = exom (vraj)
- dlzka readu: 75bp


SETUP:
- priprava prostredia: staci spustit prikaz `conda env create -f environment.yml` a natiahnu sa vsetky kniznice, ktore budeme potrebovat. Potom aktivuj env cez `conda activate bondra`.
- do project/raw_data/ nakopiruj vstupne data zo zadania (4 subory: S11.T_R1/R2 - tumor, S11.C_R1/R2 - control, paired-end reads)


KROK 1: QC raw data (FastQC + MultiQC)
- spustime FastQC na vsetkych suboroch:
```
fastqc project/raw_data/S11.T_R1.fastq.gz project/raw_data/S11.T_R2.fastq.gz \
       project/raw_data/S11.C_R1.fastq.gz project/raw_data/S11.C_R2.fastq.gz \
       -o project/qc/fastqc/ \
       -t 4
```

- agregujeme do jedneho suboru cez MultiQC:
```
multiqc project/qc/fastqc/ -o project/qc/multiqc/
```

vystup: `project/qc/multiqc/multiqc_report.html` (tu si pozrieme kvalitu readov a podla toho sa rozhodneme ci treba trimming)

=> kvalita je insane good a mam pocit , ze sme ani nemali robit QC (mali) , ale aspon sme si to presli :>


KROK 1.5: (ak nemas ref. genom stiahnuty a naindexovany (trva to milion hodin)):
- stiahneme referencny genom:
```
wget https://hgdownload.soe.ucsc.edu/goldenPath/hg38/bigZips/hg38.fa.gz -P /home/USER_NAME/PROJECT_NAME/
```
- rozbalime:
```
gunzip /home/USER_NAME/PROJECT_NAME/hg38.fa.gz
```
- a naindexujeme >.<
```
bwa index /home/USER_NAME/PROJECT_NAME/hg38.fa
```


KROK 2: zarovnanie read-ov na referencny genom (BWA)
- spustime prikaz na zarovnanie Tumor vzorky
```
bwa mem -t 4 \
  -R "@RG\tID:Tumor\tSM:Tumor\tPL:ILLUMINA\tLB:lib1" \
  /home/USER_NAME/PATH_TO_REFERENCE_GENOME/hg38.fa \
  /home/USER_NAME/PROJECT_NAME/project/raw_data/S11.T_R1.fastq.gz \
  /home/USER_NAME/PROJECT_NAME/project/raw_data/S11.T_R2.fastq.gz | \
  samtools sort -o /home//USER_NAME/PROJECT_NAME/project/aligned/Tumor.bam
```

- naindexujeme Tumor BAM (potrebne pre random access napr. v IGV / GATK)
```
samtools index /home/USER_NAME/PROJECT_NAME/project/aligned/Tumor.bam
```

- to iste spravime aj pre Control vzorku - lisi sa len Read Group (SM:Control) a vstupne FASTQ subory (S11.C_R1/R2). Mutect2 podla SM rozlisuje ktora vzorka je tumor a ktora normal, takze SM musi byt iny:
```
bwa mem -t 4 \
  -R "@RG\tID:Control\tSM:Control\tPL:ILLUMINA\tLB:lib1" \
  /home/USER_NAME/PATH_TO_REFERENCE_GENOME/hg38.fa \
  /home/USER_NAME/PROJECT_NAME/project/raw_data/S11.C_R1.fastq.gz \
  /home/USER_NAME/PROJECT_NAME/project/raw_data/S11.C_R2.fastq.gz | \
  samtools sort -o /home/USER_NAME/PROJECT_NAME/project/aligned/Control.bam
```

- a indexujeme Control BAM
```
samtools index /home/USER_NAME/PROJECT_NAME/project/aligned/Control.bam
```

