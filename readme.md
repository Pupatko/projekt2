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

=> kvalita je insane good a mam pocit , ze sme ani nemali robit QC , ale aspon sme si to presli :>