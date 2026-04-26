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

KROK 3: Postprocessing:
- markneme duplikaty, aby nam nezavadzali pri variant callingu
```
gatk MarkDuplicates \
  -I /home/USER_NAME/PROJECT_NAME/project/aligned/Tumor.bam \
  -O /home/USER_NAME/PROJECT_NAME/project/postprocessing/Tumor.markdup.bam \
  -M /home/USER_NAME/PROJECT_NAME/project/postprocessing/Tumor.markdup.metrics

gatk MarkDuplicates \
  -I /home/USER_NAME/PROJECT_NAME/aligned/Control.bam \
  -O /home/USER_NAME/PROJECT_NAME/postprocessing/Control.markdup.bam \
  -M /home/USER_NAME/PROJECT_NAME/postprocessing/Control.markdup.metrics
```

- BQSR (skipujeme, nepotrebne)

KROK 4: postalignment QC
- pre stats
```
samtools flagstat /home/bondra/projekt2/project/postprocessing/Tumor.markdup.bam > \
  /home/bondra/projekt2/project/qc/Tumor.flagstat.txt

samtools flagstat /home/bondra/projekt2/project/postprocessing/Control.markdup.bam > \
  /home/bondra/projekt2/project/qc/Control.flagstat.txt
```

- pre coverage
```
samtools coverage \
  /home/bondra/projekt2/project/postprocessing/Tumor.markdup.bam \
  > /home/bondra/projekt2/project/qc/Tumor.coverage.txt

samtools coverage \
  /home/bondra/projekt2/project/postprocessing/Control.markdup.bam \
  > /home/bondra/projekt2/project/qc/Control.coverage.txt
```

KROK 5: Variant calling

- chybali nam nejake subory, takze:
```
samtools faidx hg38.fa
gatk CreateSequenceDictionary -R hg38.fa
```

- potom mozeme spustit:
```
gatk Mutect2 \
  -R /home/bondra/projekt2/project/references/hg38/hg38.fa \
  -I /home/bondra/projekt2/project/postprocessing/Tumor.markdup.bam \
  -I /home/bondra/projekt2/project/postprocessing/Control.markdup.bam \
  --tumor-sample Tumor \
  --normal-sample Control \
  -O /home/bondra/projekt2/project/variants/Tumor_Control.vcf.gz \
  -L chr22
```

- odfiltrujeme vysledky od FP
```
gatk FilterMutectCalls \
  -R /home/bondra/projekt2/project/references/hg38/hg38.fa \
  -V /home/bondra/projekt2/project/variants/Tumor_Control.vcf.gz \
  -O /home/bondra/projekt2/project/variants/Tumor_Control.filtered.vcf.gz
```

DACO:
- zistujeme pocet variantov a potom si ich aj vypiseme a skusime ich interpretovat:
```
bcftools view -f PASS /home/bondra/projekt2/project/variants/Tumor_Control.filtered.vcf.gz | grep -v "^#" | wc -l
```
```
bcftools query -f '%CHROM\t%POS\t%REF\t%ALT\n' -i 'FILTER="PASS"'   /home/bondra/projekt2/project/variants/Tumor_Control.filtered.vcf.gz
```

KROK 6: Anotacia (asi pdf)