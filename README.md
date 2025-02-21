# Metabolomics and Genomics Association Analysis

This script demonstrates the process of associating metabolomics data with genomic data using PLINK for quality control and linear modeling for association testing. It includes data cleaning, normalization, outlier removal, and the extraction of significant SNPs, followed by association analysis between SNPs and metabolites. Finally, it generates a Manhattan plot to visualize the results.

## Prerequisites

- `data.table` library
- `dplyr` library
- `qqman` library

Install the necessary R packages if they are not already installed:

```r
install.packages("data.table")
install.packages("dplyr")
install.packages("qqman")
```

## Input Files

- `Qatari156_filtered_pruned.fam`: FAM file containing the sample information.
- `Synthetic_Metabolomics_Data.csv`: CSV file with metabolomics data.
- `pca_values.csv`: CSV file containing PCA values generated by PLINK.
- `assoc_file.qassoc`: PLINK association results.
- `snp_all_raw.raw`: PLINK raw genotype data.
- `snp_all_txt.map`: PLINK map file for genotypes.

## Steps

### 1. Quality Control with PLINK

Before starting the analysis, the following PLINK commands are used to filter the data:

```bash
# Step 1: Filter genotypes with a 0.05 missingness threshold
./plink --bfile Qatari156_filtered_pruned --geno 0.05 --make-bed --out Qatari156_filtered_pruned0.05geno

# Step 2: Filter for Hardy-Weinberg equilibrium with p-value threshold of 0.000001
./plink --bfile Qatari156_filtered_pruned0.05geno --hwe 0.000001 --make-bed --out Qatari156_filtered_pruned_hwe

# Step 3: Filter for minor allele frequency (MAF) threshold of 0.05
./plink --bfile Qatari156_filtered_pruned_hwe --maf 0.05 --make-bed --out Qatari156_filtered_pruned_maf
```

### 2. Prepare Data

Load and process the family data and metabolomics data:

```r
# Load FAM data
fam_data <- fread("Qatari156_filtered_pruned.fam", header = FALSE)
colnames(fam_data) <- c("Family_ID", "Sample_ID", "Paternal_ID", "Maternal_ID", "Sex", "Phenotype")

# Load metabolomics data and clean it
metabolites <- read.csv("Synthetic_Metabolomics_Data.csv")
metabolites = metabolites[1:156,]
metabolites$Sample = fam_data$Family_ID
metabolites_no <- metabolites[, -1]
```

### 3. Z-Score Normalization

Define a function to normalize the data using Z-scores:

```r
# Z-score normalization function
Zscore <- function(M) {
  return((M - mean(M, na.rm = TRUE)) / sd(M, na.rm = TRUE))
}

# Apply Z-score normalization to all metabolites
for (i in 1:ncol(metabolites_no)) {
  metabolites_no[, i] <- Zscore(metabolites_no[, i])
}
```

### 4. Outlier Removal

Replace outliers beyond mean ± 3 standard deviations with NA:

```r
# Function to replace outliers
replace_outliers <- function(x) {
  m <- mean(x, na.rm = TRUE)
  sd_x <- sd(x, na.rm = TRUE)
  lower_bound <- m - 3 * sd_x
  upper_bound <- m + 3 * sd_x
  x[x < lower_bound | x > upper_bound] <- NA
  return(x)
}

# Apply outlier removal to all metabolites
metabolites_no <- metabolites_no %>% 
  mutate(across(everything(), replace_outliers))

# Retain metabolites with less than 50% missing data
metabolites_no <- metabolites_no[, which(colMeans(!is.na(metabolites_no)) > 0.5)]
```

### 5. Principal Component Analysis (PCA)

Run PCA using PLINK and process the results:

```bash
# Run PCA on filtered genotypes
./plink --bfile Qatari156_filtered_pruned_maf --pca --out Qatari156_filtered_pruned_pca

# Format PCA results
awk '{$1=$1}1' OFS=',' Qatari156_filtered_pruned_pca.eigenvec > pca_values.csv
```

Load PCA values into R:

```r
# Load PCA values
pca_values <- read.csv("C:/My pc/Egcombio/IOSB/assignmet_matabolomics/pca_values.csv", header = FALSE)
pca_col <- c("FID", "IID", paste0("PC", 1:(ncol(pca_values) - 2)))
colnames(pca_values) <- pca_col

# Extract phenotype data and save it
phenotype <- pca_values[, 1:3]
write.table(phenotype, "phenotype.txt", row.names = FALSE, quote = FALSE)
```

### 6. Association Analysis

Perform association analysis using PLINK:

```bash
# Run association test using phenotype data
./plink --bfile Qatari156_filtered_pruned --assoc --pheno phenotype.txt --out assoc_file
```

Extract significant SNPs based on Bonferroni correction:

```r
# Load association results
qatar_assoc <- read.table("assoc_file.qassoc", header = TRUE)

# Bonferroni correction threshold
bonferroni_threshold <- 0.05 / 67735   # no=67735

# Filter significant SNPs
significant_snps_all <- qatar_assoc[qatar_assoc$P < bonferroni_threshold, ]
significant_snps_all <- significant_snps_all[order(significant_snps_all$P), ]
  # significant_snps <- significant_snps[1:500,] if you want to reduce number of snps

# Save significant SNPs
write.table(significant_snps_all, "significant_snps_all.txt", row.names = FALSE)

# Extract SNP names
snp_names_all <- significant_snps_all$SNP
write.table(snp_names_all, "snp_names_all.txt", row.names = FALSE, quote = FALSE, col.names = FALSE)
```

### 7. Genotype Data Preparation

Prepare SNP genotype data:

```bash
# Extract SNPs and create new datasets
./plink --bfile Qatari156_filtered_pruned --extract snp_names_all.txt --make-bed --out snp_all
./plink --bfile snp_all --recode A --out snp_all_raw
./plink --bfile snp_all --recode --out snp_all_txt
```

Load genotype data into R:

```r
# Load genotype data
Genotypes <- fread("snp_all_raw.raw", header = TRUE)
Genotypes_map <- fread('snp_all_txt.map')
colnames(Genotypes_map) <- c("CHR", "SNP", "P", "BP")

SNPs_mat <-  as.matrix(Genotypes[, -(1:6)])
```

### 8. Linear Model Association

Perform association analysis between metabolites and SNPs:

```r
# Create an empty dataframe to store results
association_result <- data.frame(SNP = character(), Metabolite = character(), p_value = numeric(), stringsAsFactors = FALSE)

# Loop over metabolites and SNPs to fit linear models
for (i in colnames(metabolites_no)) {
  for (j in 1:ncol(SNPs_mat)) {
    model <- lm(metabolites_no[, i] ~ SNPs_mat[, j])
    p_value <- summary(model)$coefficients[2, 4]
    association_result <- rbind(association_result, data.frame(SNP = colnames(SNPs_mat)[j], Metabolite = i, p_value = p_value))
  }
}

# Filter for significant associations
association_result <- association_result[association_result$p_value < 0.05, ]
association_result$SNP <- gsub("_.*", "", association_result$SNP)

# Save results to a CSV file
write.csv(association_result, "association_result.csv")
```

### 9. Manhattan Plot

Generate a Manhattan plot for the significant results:

```r
# Merge association results with genotype map
manhattan_data <- merge(association_result, Genotypes_map, by = "SNP")
Manhat <- manhattan_data[, c("SNP", "CHR", "BP", "p_value")]
colnames(Manhat)[4] <- "P"

# Plot Manhattan plot
manhattan(Manhat, annotatePval = 0.02, col = c("blue4", "orange3"))
```

### Thanks !

