Microresp pipeline
================
David Rodrigo Cajas
2025-03-18

## Important

``` r
# R version 4.2.2 (2022-10-31) -- "Innocent and Trusting"
# Copyright (C) 2022 The R Foundation for Statistical Computing
# Platform: x86_64-apple-darwin17.0 (64-bit)

# Author: David Rodrigo Cajas
```

This R script expects the following files to be in the same folder as
the script file to properly run:

- “Results_microresp_Calc.xlsx” as an excel spreadsheet containing the
  microresp spectrometer data in a single page per plate. The
  spreadsheet is assumed to follow this strict column order:
  - A: Well position in the agar plate of your Microresp system.
  - B: Comments (will be omitted from analyses).
  - C: Sample labels. Calibration points are expected to be labelled
    “CX”, samples “SX” and “H2O” is assumed for the blank.
  - D: Spectrometer reading at 570 nm before exposing the agar plate.
  - E: Spectrometer reading at 570 nm after exposing the agar plate.
  - I: List of sample names to be considered in analysis in “SX” format.
  - J: Sample identifier in the structure “var1_var2_var3_R”, where var
    are the variable short names and R is the replicate/block number.
- “experiment_metadata.xlsx” as an excel spreadsheet containing metadata
  for each experimental variable separated by pages, so each page has
  the metadata concerning 1 variable.
  - Column “A” values on each page are expected to match at lease 1 of
    the variables within the Column “J” in “Results_microresp_Calc.xlsx”
    file.
  - The number of pages of this file should correspond to the number of
    variables in the identifiers contained in the column “J” in
    “Results_microresp_Calc.xlsx” file.
- “co2cal.xlsx” as a spreadsheet of 2 columns, where:
  - Column “A” is expected to match the sample labels in column “C” of
    “experiment_metadata.xlsx” file for the calibration points.
  - Column “B” is expected to contain the %CO2 values for those points.
- “required_packages.rds” is an R object file containing the list of
  packages required to run this script.

Now let’s get to it…

## 0) Set up R session

### 0.1) Working directory and packages

First we need to install the Rstudioapi package

``` r
# (Install and) load Rstudio api package
if ("rstudioapi" %in% installed.packages()) {
  library(rstudioapi)
} else {
  install.packages("rstudioapi")
  library(rstudioapi)
} 
```

Now we will set the working directory to where the R markdown folder is

``` r
wd <- dirname(rstudioapi::getSourceEditorContext()$path)
# wd <- "/Users/davidcajasmunoz/Library/CloudStorage/GoogleDrive-dadavid.cajas@gmail.com/Mi unidad/Academia/Postgrado/UvA/Results and experiments track/Experiment 1 - Inoculants and amendments on crops and grasses/9-Analysis/2-Microresp/Samples"

setwd(wd)
```

Import the list of packages in “required_packages.rds” and install them

``` r
# Load list of required packages
required_packages <- readRDS("required_packages.rds")
# Install script's required packages
need_install <- required_packages[!(required_packages) %in% installed.packages()]
if (length(need_install) > 0) {
  install.packages(need_install)
}
```

\[Later on\] If you modified this code, don’t forget to update the list
of script’s required packages

``` r
# required_packages <- names(sessionInfo()$otherPkgs)
# saveRDS(required_packages, "required_packages.rds")
```

### 0.2) Importing working data

This chunk imports all pages inside the “Results_microresp_Calc.xlsx”
excel file as dataframes of 4 columns and 96 rows, replacing the names
of the rows for standardised ones

IMPORTANT: NOTE THAT THIS CODE ASSUMES THE COLUMNS ARE IN A PARTICULAR
ORDER:

- A = Well

- B = non relevant \[ignored\]

- C = Sample_ID

- D = Absorbance before

- E = Absorbance after

``` r
library(readxl)
library(dplyr)
```

    ## 
    ## Attaching package: 'dplyr'

    ## The following objects are masked from 'package:stats':
    ## 
    ##     filter, lag

    ## The following objects are masked from 'package:base':
    ## 
    ##     intersect, setdiff, setequal, union

``` r
library(stringr)
library(gtools)

# set the file containing the data
sourcefile <- "Results_microresp_Calc.xlsx"

for (i in 1:length(excel_sheets(sourcefile))) {
  # Import data
  
  df <- read_excel(sourcefile, sheet = i, range = "A1:E96", col_types = c("text", "skip", "text", "numeric", "numeric")) # import columns A to E and rows 1 to 96 from sheet i, skipping the second column and the other 4 in format chr, chr, int, int
  
  # Small adjustments
  
  colnames(df) <- c("well", "sample", "A570_0h", "A570_6h") # rename columns
  df <- mutate(df, 
               row = str_extract(well, "^[A-Z]"),
               col = factor(as.factor(str_extract(well, "\\d+")), levels = c(seq(1,12,1)))
  ) # add separate columns for the coordinates of the sample in the plate (contained in "well" column)
  
  # Wrap up
  
  assign(paste("mr",i, sep=""), df) # name output dataframe
  rm(df,i) # remove auxiliary "df" and "i" objects
}

# list all created dataframes so they can be easily called back
dfs <- mixedsort(ls(pattern = "^mr\\d+$"))
```

### 0.3) Import samples metadata

#### 0.3.1) Import metadata from samples processed in Microresp

``` r
library(readxl)
library(stringr)
library(tidyr)

#Import sample labels from Microresp spreadsheet

for (i in 1:length(excel_sheets(sourcefile))) {
  # Import data
  
  df <- read_excel(sourcefile, sheet = i, range = "I1:J96", col_names = FALSE, col_types = c("text", "text")) # import columns A to E and rows 1 to 96 from sheet i, skipping the second column and the other 4 in format chr, chr, int, int
  
  # Delete NA rows
  df <- df[complete.cases(df), ]
  
  # Small adjustments
  
  colnames(df) <- c("sample", "condition") # rename columns
  
  # add separate columns for each of the conditions encoded in the "J" column, now named "condition
  df <- separate(df, 
                 col = condition, 
                 into = c("plant", "soil", "treatment", "replicate"), 
                 sep = "_")
  
  # Wrap up
  
  assign(paste("mr",i,"_meta", sep=""), df) # name output dataframe
  rm(df,i) # remove auxiliary "df" and "i" objects
}
```

    ## New names:
    ## New names:
    ## New names:
    ## New names:
    ## New names:
    ## New names:
    ## New names:
    ## New names:
    ## New names:
    ## New names:
    ## New names:
    ## New names:
    ## • `` -> `...1`
    ## • `` -> `...2`

    ## Warning: Expected 4 pieces. Missing pieces filled with `NA` in 1 rows [1].

    ## New names:
    ## New names:
    ## New names:
    ## New names:
    ## New names:
    ## • `` -> `...1`
    ## • `` -> `...2`

``` r
# list all created metadata dataframes so they can be easily called back
mdfs <- mixedsort(ls(pattern = "^mr\\d+_meta$"))
```

#### 0.3.2) Import experiment metadata

Note that this script imports the metadata from
“experiment_metadata.xlsx” assuming that it’s separated by sheets called
“treatment_data”, “soil_data” and “plant_data”. The order is not
relevant.

``` r
# Define source of experiment metadata

sourcemeta <- "experiment_metadata.xlsx"

for (i in excel_sheets(sourcemeta)) {
  # Import data
  
  df <- read_excel(sourcemeta, sheet = i) # import all experiment metadata in separated dataframes per sheet
  
  # Wrap up
  
  assign(i, df) # name output dataframe
  rm(df,i) # remove auxiliary "df" and "i" objects
}

# Add order to treatments
treatment_data$label <- factor(as.factor(treatment_data$label), levels = c("Control", "Disease suppression", "AMF", "Nitrogen fixation", "Phosphate solubilisation"))
treatment_data$applied_product <- factor(as.factor(treatment_data$applied_product), levels = c("No product", "Compete Plus", "MycorGran 2.0", "Vixeran", "NuelloPhos"))

# Small processing of soil data

numeric_cols <- names(soil_data)[sapply(soil_data, function(x) any(grepl("[0-9]", x)))] # Auxiliary object listing the columns that contain numbers
soil_data[numeric_cols] <- lapply(soil_data[numeric_cols], function(x) {
  x <- ifelse(grepl("^<", x), 0, x) # Identify values beginning with "<", which are below detection limitm and replace them with 0
  x <- as.numeric(x)  # Convert to numeric AFTER cleaning
  x
}) # This function replaces values that start with "<" with 0
rm(numeric_cols) # Remove auxiliary object

## On develop: Check soil texture and include in metadata if not there already.

# library(soiltexture)
# library(ggplot2)
# 
# TT.text(data.frame(
#   soil = soil_data$soil,
#   SAND = soil_data$sand_perc,
#   SILT = soil_data$silt_perc,
#   CLAY = soil_data$clay_perc
# ), geo = "USDA"
# )
```

#### 0.3.3) Extract relevant experiment metadata and add to Microresp metadata dataframes

``` r
# Define relevant metadata to extract. This can be modified for further customisation
pick_metadata <- c(colnames(soil_data[,c(3,6,10:14)]), # chosen metadata columns in soil_data
                   colnames(treatment_data[,c(3,5:14)]), # chosen metadata columns in treatment_data
                   colnames(plant_data[,c()])) # chosen metadata columns in plant_data

# Create a function to replace values in a common column. It will be used to replace labels
replace_values <- function(df1, df2, col_name, new_col_name) {
  df1 %>%
    left_join(df2, by = col_name) %>%  # Join with lookup table
    mutate(!!col_name := !!sym(new_col_name)) %>% # Replace original values with labels
    select(-!!sym(new_col_name))
} # replaces the values of the col_name column in the df1 for the values of the new_col_name in the df2, assuming col_name exist in both dataframes

# Add selected metadata and change labels to metadata dataframe

for (i in mdfs) {
  df <- get(i) # get the object
  
  # Apply formulas
  df <- df %>%
    
    # Add defined metadata
    
    left_join(select(plant_data, "plant",any_of(pick_metadata)), by = "plant") %>%
    left_join(select(soil_data, "soil",any_of(pick_metadata)), by = "soil") %>%
    left_join(select(treatment_data, "treatment",any_of(pick_metadata)), by = "treatment") %>%
    
    # Replace compressed labels for full size variables
    
    replace_values(plant_data[,1:2], "plant", "label") %>%
    replace_values(soil_data[,1:2], "soil", "label") %>%
    replace_values(treatment_data[,1:2], "treatment", "label")
  
  # Wrap up
  
  assign(i,df) # write back on the original dataframe
  rm(df,i) # remove auxiliary objects
  
}
```

#### 0.3.4) Paste Microresp metadata dataframes from 0.3.3) into Microresp dataframes imported in 0.2)

``` r
for (i in dfs) {
  df <- get(i) # get the object
  df_meta <- get(paste(i,"_meta",sep = ""))
  
  # Add metadata
  
  df <- left_join(df,df_meta,by = "sample")
  
  # Wrap up
  
  assign(i,df) # write back on the original dataframe
  rm(df,df_meta,i) # remove auxiliary objects
  
}

# Delete auxiliary objects if desired

# rm(pick_metadata,mdfs)
```

### 0.4) Import %CO2 calibration values

This is just stored separately in another file called “co2cal.xlsx”

``` r
calvalues <- read_excel("co2cal.xlsx", col_names = c("sample","CO2_per"), col_types =c("text","numeric"))
```

## 1) Data processing: Operations on created dataframes

### 1.1) Calculate normalised absorbance values and add calibration %CO2 values from “calvalues” object

The following formula is used to calculate normalised absorbance:

$$\
A_i = \frac{A_{t6}}{A_{t0}}*µ_{{A_{t0}}}$$

``` r
for (i in dfs) {
  df <- get(i) # get the object
  
  # Apply formulas
  
  df$AAdjusted <- (df$A570_6h / df$A570_0h)*mean(df$A570_0h) # Normalisation formula
  df$AAdjusted[is.infinite(df$AAdjusted) | is.nan(df$AAdjusted)] <- NA  # Replace NAs and infinite values for NA
  
  # Add calibration values
  
  df <- merge(df, calvalues, by = "sample", all.x = T, sort = F)
  
  # Error check and report
  
  missing_cal <- df$sample %in% c(paste0("C", 1:20),"H2O") & is.na(df$CO2_per)
  if (any(missing_cal)) {
    print(paste("[ Calibration error in:",i,"]","Missing calibration values for:", paste(df$sample[missing_cal], collapse = ", ")))
  }
  
  # Wrap up
  
  assign(i,df) # write back on the original dataframe
  rm(df,i, missing_cal) # remove auxiliary objects
}
```

For easier management, each Microresp dataset will be split into a
calibration (to fit the model) and a sample subsets (to analyse later).

``` r
## Split dataframes into "calibration" and "sample" subsets

for (i in dfs) {
  df <- get(i) # get the object
  
  is_sample <- grepl("^S[0-9]+$", df$sample) # logical vector where SX values are TRUE and the rest, FALSE
  cal <- df$sample %in% c(paste0("C", 1:20),"H2O") # logical vector where CX and H2O values are TRUE and the rest, FALSE
  
  # Create new dataframes based on the boolean vector just created
  
  df.c <- df[cal, ]
  df.s <- df[is_sample, ]
  
  # Store new dataframes
  
  assign(paste0(i, "_c"), df.c)
  assign(paste0(i, "_s"), df.s)
  
  # Wrap up

  rm(df,i, is_sample, cal, df.c, df.s) # remove auxiliary "df" and "i" objects
}
```

### 1.2) Fitting a model

#### 1.2.1) Create auxiliary objects

``` r
dfs_c <- mixedsort(ls(pattern = "^mr\\d+_c$"))
dfs_s <- mixedsort(ls(pattern = "^mr\\d+_s$"))
```

#### 1.2.2) Fit model

This approach seems a bit complex because it contains 3 nested loops.
Nonetheless, essentially it fits the data on each calibration dataset to
the calibration values imported assuming the following formula:

$$
\%CO2 = A+\frac{B}{1+D*A_i}
$$

The reason for the loops are as follows:

1.  The first loop (for{}) runs iteratively over all calibration
    datasets, so it gets a model for each one.
2.  The second loop (for{}) runs all different optimisation algorithms
    for the fitting function in an effort to ensure that a solution is
    found following the procedure in the third loop. After the third
    loop is finished, it will check if a “model” was stored. If yes, it
    will save it into a new object. If not, it will create a warning
    message.
3.  The third loop (tryCatch{}) gives a more robust approach to errors.
    tryCatch will “try” the code with the first algorithm defined in the
    previous loop. If it succeeds, it will jump directly to report the
    found model. If not, it will create a warning message.

``` r
for (i in dfs_c) { 
  # this approach has 3 nested loops. This is the [1]
  
  df <- get(i) # get the object
  
  # Fit model: Nested for() [2] loop was implemented so different optimisation algorithms can by tried
  
  algorithms <- c("default", "port", "plinear") # List of algorithms to try
  model_found <- FALSE # Flag to track if a model was successfully fit
  
  for (algorithm in algorithms) { 
    
    # The modeling function is further nested [3] in the TryCatch function
    
    tryCatch({ # tryCatch prevents the loop from stopping if the modeling is unsuccessful
    
    # Model itself: It will get stored in an auxiliary "m" object
    m <- nls(CO2_per ~ a + b / (1 + d * AAdjusted), 
             data = df, 
             start = list(a = -2, b = -10, d = -6.8), # the values for a, b and d used as start are taken from the manual
             control = nls.control(minFactor = 1e-10, maxiter = 1e7),
             algorithm = algorithm) # Here is the list of algorithms created in the first for() loop
    model_found <- TRUE # If successful, the value for this flag will be changed to TRUE
    
    break # If the previous code lines did not halt, this line will break out of the [2] algorithm loop, effectively stopping it.
    
  } # If the code was halted before (ie: nls() function halted and no model was produced), then the following code will be run.
    
    , error = function(e) {
      cat("[ Error fitting model for", i,"using",algorithm, "algorithm ]", ":\n", conditionMessage(e), "\n")
    }
    , warning = function(w) {
      cat("[ Warning fitting model for", i,"]", ":\n", conditionMessage(w), "\n")
    })
  }
  
  # Wrap up
  
  if (exists("m")) { # If a model exists, a success message and store the model (if fit was successful)
    print(paste("Model for", i, "successful using",algorithm,"algorithm:"))
    print(summary(m))
    
    assign(paste0("mod_", i), m)
  } else { # If a model does not exist, print an error message.
    cat("[ Error: Failed to fit model for", i, "using all algorithms. Skipping. ]\n")
  }
  
  rm(df,i, m, algorithm, algorithms) # remove auxiliary objects
}
```

    ## [1] "Model for mr1_c successful using default algorithm:"
    ## 
    ## Formula: CO2_per ~ a + b/(1 + d * AAdjusted)
    ## 
    ## Parameters:
    ##   Estimate Std. Error t value Pr(>|t|)    
    ## a  -0.5162     0.1499  -3.444 0.001440 ** 
    ## b  -1.5165     0.4118  -3.683 0.000732 ***
    ## d  -3.3082     0.2023 -16.356  < 2e-16 ***
    ## ---
    ## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
    ## 
    ## Residual standard error: 0.3825 on 37 degrees of freedom
    ## 
    ## Number of iterations to convergence: 7 
    ## Achieved convergence tolerance: 4.905e-06
    ## 
    ## [ Error fitting model for mr2_c using default algorithm ] :
    ##  singular gradient 
    ## [1] "Model for mr2_c successful using port algorithm:"
    ## 
    ## Formula: CO2_per ~ a + b/(1 + d * AAdjusted)
    ## 
    ## Parameters:
    ##   Estimate Std. Error t value Pr(>|t|)    
    ## a  -2.2571     2.4432  -0.924    0.372    
    ## b  -1.0164     1.4207  -0.715    0.487    
    ## d  -1.1573     0.1499  -7.721 3.29e-06 ***
    ## ---
    ## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
    ## 
    ## Residual standard error: 0.6946 on 13 degrees of freedom
    ## 
    ## Algorithm "port", convergence message: relative convergence (4)
    ## 
    ## [1] "Model for mr3_c successful using default algorithm:"
    ## 
    ## Formula: CO2_per ~ a + b/(1 + d * AAdjusted)
    ## 
    ## Parameters:
    ##   Estimate Std. Error t value Pr(>|t|)    
    ## a -0.61938    0.08704  -7.116 7.86e-06 ***
    ## b -2.23224    0.32930  -6.779 1.30e-05 ***
    ## d -4.05357    0.15853 -25.570 1.68e-12 ***
    ## ---
    ## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
    ## 
    ## Residual standard error: 0.1324 on 13 degrees of freedom
    ## 
    ## Number of iterations to convergence: 6 
    ## Achieved convergence tolerance: 6.371e-08
    ## 
    ## [1] "Model for mr4_c successful using default algorithm:"
    ## 
    ## Formula: CO2_per ~ a + b/(1 + d * AAdjusted)
    ## 
    ## Parameters:
    ##   Estimate Std. Error t value Pr(>|t|)    
    ## a -0.19868    0.06878  -2.888   0.0127 *  
    ## b -0.60458    0.08670  -6.973 9.72e-06 ***
    ## d -4.32954    0.06248 -69.297  < 2e-16 ***
    ## ---
    ## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
    ## 
    ## Residual standard error: 0.1757 on 13 degrees of freedom
    ## 
    ## Number of iterations to convergence: 5 
    ## Achieved convergence tolerance: 8.11e-06
    ## 
    ## [1] "Model for mr5_c successful using default algorithm:"
    ## 
    ## Formula: CO2_per ~ a + b/(1 + d * AAdjusted)
    ## 
    ## Parameters:
    ##   Estimate Std. Error t value Pr(>|t|)    
    ## a -0.62129    0.06614  -9.394 3.70e-07 ***
    ## b -2.03125    0.22314  -9.103 5.29e-07 ***
    ## d -3.50143    0.09519 -36.785 1.58e-14 ***
    ## ---
    ## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
    ## 
    ## Residual standard error: 0.1 on 13 degrees of freedom
    ## 
    ## Number of iterations to convergence: 7 
    ## Achieved convergence tolerance: 2.386e-06
    ## 
    ## [1] "Model for mr6_c successful using default algorithm:"
    ## 
    ## Formula: CO2_per ~ a + b/(1 + d * AAdjusted)
    ## 
    ## Parameters:
    ##   Estimate Std. Error t value Pr(>|t|)    
    ## a -0.26260    0.06691  -3.925  0.00174 ** 
    ## b -0.84022    0.10972  -7.658  3.6e-06 ***
    ## d -4.35590    0.07485 -58.192  < 2e-16 ***
    ## ---
    ## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
    ## 
    ## Residual standard error: 0.1626 on 13 degrees of freedom
    ## 
    ## Number of iterations to convergence: 5 
    ## Achieved convergence tolerance: 5.743e-06
    ## 
    ## [1] "Model for mr7_c successful using default algorithm:"
    ## 
    ## Formula: CO2_per ~ a + b/(1 + d * AAdjusted)
    ## 
    ## Parameters:
    ##   Estimate Std. Error t value Pr(>|t|)    
    ## a  -1.2984     0.2534  -5.123 0.000196 ***
    ## b  -7.6044     3.0289  -2.511 0.026062 *  
    ## d  -6.0489     1.2112  -4.994 0.000245 ***
    ## ---
    ## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
    ## 
    ## Residual standard error: 0.2247 on 13 degrees of freedom
    ## 
    ## Number of iterations to convergence: 5 
    ## Achieved convergence tolerance: 4.036e-06
    ## 
    ## [1] "Model for mr8_c successful using default algorithm:"
    ## 
    ## Formula: CO2_per ~ a + b/(1 + d * AAdjusted)
    ## 
    ## Parameters:
    ##   Estimate Std. Error t value Pr(>|t|)    
    ## a  -1.3801     0.2098  -6.579 1.77e-05 ***
    ## b  -5.4077     1.4840  -3.644  0.00297 ** 
    ## d  -3.9367     0.4468  -8.812 7.64e-07 ***
    ## ---
    ## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
    ## 
    ## Residual standard error: 0.1515 on 13 degrees of freedom
    ## 
    ## Number of iterations to convergence: 6 
    ## Achieved convergence tolerance: 1.258e-06
    ## 
    ## [1] "Model for mr9_c successful using default algorithm:"
    ## 
    ## Formula: CO2_per ~ a + b/(1 + d * AAdjusted)
    ## 
    ## Parameters:
    ##   Estimate Std. Error t value Pr(>|t|)    
    ## a  -0.9041     0.2068  -4.372 0.000756 ***
    ## b  -4.0341     1.3017  -3.099 0.008462 ** 
    ## d  -5.2755     0.6334  -8.329 1.43e-06 ***
    ## ---
    ## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
    ## 
    ## Residual standard error: 0.249 on 13 degrees of freedom
    ## 
    ## Number of iterations to convergence: 7 
    ## Achieved convergence tolerance: 2.052e-06
    ## 
    ## [1] "Model for mr10_c successful using default algorithm:"
    ## 
    ## Formula: CO2_per ~ a + b/(1 + d * AAdjusted)
    ## 
    ## Parameters:
    ##   Estimate Std. Error t value Pr(>|t|)    
    ## a  -1.0449     0.2030  -5.148 6.74e-05 ***
    ## b  -2.6504     0.7011  -3.781  0.00137 ** 
    ## d  -3.4763     0.2513 -13.832 4.96e-11 ***
    ## ---
    ## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
    ## 
    ## Residual standard error: 0.225 on 18 degrees of freedom
    ## 
    ## Number of iterations to convergence: 8 
    ## Achieved convergence tolerance: 3.371e-06
    ## 
    ## [1] "Model for mr11_c successful using default algorithm:"
    ## 
    ## Formula: CO2_per ~ a + b/(1 + d * AAdjusted)
    ## 
    ## Parameters:
    ##   Estimate Std. Error t value Pr(>|t|)    
    ## a -0.63193    0.07548  -8.372 1.27e-07 ***
    ## b -1.98046    0.26051  -7.602 5.03e-07 ***
    ## d -4.13416    0.13230 -31.249  < 2e-16 ***
    ## ---
    ## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
    ## 
    ## Residual standard error: 0.1267 on 18 degrees of freedom
    ## 
    ## Number of iterations to convergence: 6 
    ## Achieved convergence tolerance: 1.194e-06
    ## 
    ## [1] "Model for mr12_c successful using default algorithm:"
    ## 
    ## Formula: CO2_per ~ a + b/(1 + d * AAdjusted)
    ## 
    ## Parameters:
    ##   Estimate Std. Error t value Pr(>|t|)    
    ## a  -2.1963     0.2567  -8.557 9.27e-08 ***
    ## b -34.7089    23.7458  -1.462    0.161    
    ## d -14.2587     7.7123  -1.849    0.081 .  
    ## ---
    ## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
    ## 
    ## Residual standard error: 0.1221 on 18 degrees of freedom
    ## 
    ## Number of iterations to convergence: 5 
    ## Achieved convergence tolerance: 5.24e-07
    ## 
    ## [ Error fitting model for mr13_c using default algorithm ] :
    ##  singular gradient 
    ## [ Error fitting model for mr13_c using port algorithm ] :
    ##  Convergence failure: singular convergence (7) 
    ## [ Error fitting model for mr13_c using plinear algorithm ] :
    ##  singular matrix 'a' in solve 
    ## [ Error: Failed to fit model for mr13_c using all algorithms. Skipping. ]

    ## Warning in rm(df, i, m, algorithm, algorithms): object 'm' not found

    ## [1] "Model for mr14_c successful using default algorithm:"
    ## 
    ## Formula: CO2_per ~ a + b/(1 + d * AAdjusted)
    ## 
    ## Parameters:
    ##   Estimate Std. Error t value Pr(>|t|)    
    ## a -0.49432    0.08442  -5.856 1.52e-05 ***
    ## b -1.62958    0.26731  -6.096 9.26e-06 ***
    ## d -3.34875    0.11885 -28.177 2.42e-16 ***
    ## ---
    ## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
    ## 
    ## Residual standard error: 0.1616 on 18 degrees of freedom
    ## 
    ## Number of iterations to convergence: 7 
    ## Achieved convergence tolerance: 4.897e-07
    ## 
    ## [1] "Model for mr15_c successful using default algorithm:"
    ## 
    ## Formula: CO2_per ~ a + b/(1 + d * AAdjusted)
    ## 
    ## Parameters:
    ##   Estimate Std. Error t value Pr(>|t|)    
    ## a  -0.9096     0.1408  -6.462 4.43e-06 ***
    ## b  -3.9693     0.9425  -4.212 0.000524 ***
    ## d  -4.2796     0.3782 -11.316 1.29e-09 ***
    ## ---
    ## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
    ## 
    ## Residual standard error: 0.1618 on 18 degrees of freedom
    ## 
    ## Number of iterations to convergence: 4 
    ## Achieved convergence tolerance: 5.002e-06
    ## 
    ## [1] "Model for mr16_c successful using default algorithm:"
    ## 
    ## Formula: CO2_per ~ a + b/(1 + d * AAdjusted)
    ## 
    ## Parameters:
    ##   Estimate Std. Error t value Pr(>|t|)    
    ## a -0.78746    0.08451  -9.318 2.62e-08 ***
    ## b -3.25024    0.47983  -6.774 2.40e-06 ***
    ## d -4.13635    0.20486 -20.191 8.17e-14 ***
    ## ---
    ## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
    ## 
    ## Residual standard error: 0.1156 on 18 degrees of freedom
    ## 
    ## Number of iterations to convergence: 6 
    ## Achieved convergence tolerance: 7.702e-07
    ## 
    ## [1] "Model for mr17_c successful using default algorithm:"
    ## 
    ## Formula: CO2_per ~ a + b/(1 + d * AAdjusted)
    ## 
    ## Parameters:
    ##   Estimate Std. Error t value Pr(>|t|)    
    ## a -0.49432    0.08442  -5.856 1.52e-05 ***
    ## b -1.62958    0.26731  -6.096 9.26e-06 ***
    ## d -3.34875    0.11885 -28.177 2.42e-16 ***
    ## ---
    ## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
    ## 
    ## Residual standard error: 0.1616 on 18 degrees of freedom
    ## 
    ## Number of iterations to convergence: 7 
    ## Achieved convergence tolerance: 4.897e-07

It is possible to export a model for later use (ie: a calibration plate)

``` r
saveRDS(mod_mr1_c,"microresp_ref_model.rds")
```

### 1.3) Estimate %CO2 values for samples

#### 1.3.0) Import previews models if wanted

If desired, a specific model can be imported here:

``` r
# mod_import <- readRDS("model3.rds")
```

#### 1.3.1) Perform calculations based on a given model

The constant values (A, B and D) from a given active model are used to
calculate %CO2 on the samples based on the same formula stated before.

``` r
# Define the active model
active_model <- mod_mr1_c

# Calculate %CO2 based on active model

for (i in dfs_s) {
  df <- get(i) # get the object
  
  # Apply formula and add data
  
  df$CO2_per <- coef(active_model)["a"] + coef(active_model)["b"] / (1 + coef(active_model)["d"] * df$AAdjusted) 
  
  # Wrap up
  
  assign(i,df) # write back on the original dataframe
  rm(df,i) # remove auxiliary objects
}

# Calculate %CO2 based on their own plate model

# mr2_s$CO2_per <- coef(mod_mr1_c)["a"] + coef(mod_mr1_c)["b"] / (1 + coef(mod_mr1_c)["d"] * mr2_s$AAdjusted)
# mr3_s$CO2_per <- coef(mod_mr1_c)["a"] + coef(mod_mr1_c)["b"] / (1 + coef(mod_mr1_c)["d"] * mr2_s$AAdjusted)
# mr4_s$CO2_per <- coef(mod_mr1_c)["a"] + coef(mod_mr1_c)["b"] / (1 + coef(mod_mr1_c)["d"] * mr2_s$AAdjusted)
```

### 1.4) Estimate microbial respiration rates

The following formula is used to calculate microbial respiration rate
based on the %CO2 detected on each well:

$$
Respiration_{µgCO_2-Cg^{-1}_{dry~soil}h^{-1}} = \frac{ \frac{\%CO_2}{100} \times V_{µL} \times \frac{44}{22.4} \times \frac{12}{44} \times \frac{273}{273+T_{°C}}} { SoilDwt_{g} \times t_{h} }
$$

Considering the following values:

- $\%CO_2$ is the estimated value calculated before (in % (v/v) ).

- $V_{µL}$ is the estimated headspace volume in the (calibration) system
  (in µL). In this case:

  - $V_{µL} = V_{Deep~well} + V_{Agar~well} - V_{Agar} - V_{Calibration~solution} = 1200 µL + 400 µL - 150 µL - 250 µL = 1200 µL_{Headspace}$

- $T_{°C}$ is the incubation temperature (in °C). In this case: 26 °C

- $SoilDwt_{g}$ is the soil dry weight (in g) added to each well. In
  this case: 0.5 g

- $t_h$ is the incubation time of the system (in h). In this case: 6 h.

Also Note that there are some constant values included:

- $µCO_2 = 44 g/mol$

- $µC = 12 g/mol$

- $V_{ideal~gas} = 22.4 L/mol$

``` r
for (i in dfs_s) {
  df <- get(i) # get the object
  
  # Set Microresp system values
  
  tem <- 26 # Temperature, in °C
  v <- 1200 # Headspace volume in µL
  w <- 0.5 # Dry soil weight per well in g
  tim <- 6 # Incubation time in h
  
  # Apply formula and add data
  
  df$Respiration_rate <-  ( (df$CO2_per/100)*v*(44/22.4)*(12/44)*(273/(273+tem)) ) / (w*tim)
  
  # Wrap up
  
  assign(i,df) # write back on the original dataframe
  rm(df,i,tem,v,w,tim) # remove auxiliary objects
} # Fixed system values are declared inside the for loop
```

### 1.5) Merge data and export.

All samples dataframes can be now merged into one, preserving a label
for their origin (the plate)

``` r
# Create an empty list to store the data frames. Don't worry, this is only needed for the following for loop and it will be removed after.
samples_list <- list()

# Loop through the sample data frames, add them to the (temporary) list and merge all samples data in a new dataframe called 'merged_samples'.
for (i in dfs_s) {
  df <- get(i) # get the object
  samples_list[[i]] <- df # Add the data frame to the list
  
  # Merge all data frames in the list into a single data frame
  
  merged_samples <- do.call(rbind, samples_list)
  
  # Wrap up
  
  rm(df,i) # remove auxiliary objects
  
}

merged_samples$plate <- rep(excel_sheets(sourcefile), times = sapply(samples_list, nrow)) # For this line to work, all sheets in the input excel file have to have at least 1 sample and a corresponding "mrx_s" dataframe

rm(samples_list) # remove the list object
head(merged_samples)
```

    ##          sample well A570_0h A570_6h row col     plant              soil
    ## mr1_s.49     S4   A9   1.539   1.046   A   9 Faba bean   Sandy high Phos
    ## mr1_s.50     S3  A10   1.504   1.100   A  10 Faba bean    Sandy low Phos
    ## mr1_s.51     S2  A11   1.501   1.105   A  11 Faba bean Sandy not managed
    ## mr1_s.52     S1  A12   1.448   1.116   A  12 Faba bean Sandy not managed
    ## mr1_s.61     S4   B9   1.534   0.653   B   9 Faba bean   Sandy high Phos
    ## mr1_s.62     S3  B10   1.516   1.078   B  10 Faba bean    Sandy low Phos
    ##                         treatment replicate origin_location    soil_texture
    ## mr1_s.49      Disease suppression         1        Hoge eng            Sand
    ## mr1_s.50                  Control         1           Joppe Sandy clay loam
    ## mr1_s.51      Disease suppression         1     Droevendaal      Loamy sand
    ## mr1_s.52 Phosphate solubilisation         1     Droevendaal      Loamy sand
    ## mr1_s.61      Disease suppression         1        Hoge eng            Sand
    ## mr1_s.62                  Control         1           Joppe Sandy clay loam
    ##          CaCO3_perc NOM_perc  pH P-CaCl2_mgP/kg P-AL_mgP/kg applied_product
    ## mr1_s.49        0.0      1.1 5.5            5.4       402.0    Compete Plus
    ## mr1_s.50        0.5      7.9 5.4            0.3        13.1      No product
    ## mr1_s.51        0.4      4.7 4.6            0.7       100.0    Compete Plus
    ## mr1_s.52        0.4      4.7 4.6            0.7       100.0      NuelloPhos
    ## mr1_s.61        0.0      1.1 5.5            5.4       402.0    Compete Plus
    ## mr1_s.62        0.5      7.9 5.4            0.3        13.1      No product
    ##              target_function         active_principle_1 active_principle_2
    ## mr1_s.49 Disease suppression Bacillus amyloliquefaciens   Bacillus pumilus
    ## mr1_s.50                <NA>                       <NA>               <NA>
    ## mr1_s.51 Disease suppression Bacillus amyloliquefaciens   Bacillus pumilus
    ## mr1_s.52     Nutrient supply    Pseudomonas fluorescens               <NA>
    ## mr1_s.61 Disease suppression Bacillus amyloliquefaciens   Bacillus pumilus
    ## mr1_s.62                <NA>                       <NA>               <NA>
    ##          active_principle_3     active_principle_4      active_principle_5
    ## mr1_s.49  Bacillus subtilis Bacillus licheniformis Azotobacter chroococcum
    ## mr1_s.50               <NA>                   <NA>                    <NA>
    ## mr1_s.51  Bacillus subtilis Bacillus licheniformis Azotobacter chroococcum
    ## mr1_s.52               <NA>                   <NA>                    <NA>
    ## mr1_s.61  Bacillus subtilis Bacillus licheniformis Azotobacter chroococcum
    ## mr1_s.62               <NA>                   <NA>                    <NA>
    ##              active_principle_6    active_principle_7 active_principle_8
    ## mr1_s.49 Trichoderma atroviride Trichoderma harzianum               <NA>
    ## mr1_s.50                   <NA>                  <NA>               <NA>
    ## mr1_s.51 Trichoderma atroviride Trichoderma harzianum               <NA>
    ## mr1_s.52                   <NA>                  <NA>               <NA>
    ## mr1_s.61 Trichoderma atroviride Trichoderma harzianum               <NA>
    ## mr1_s.62                   <NA>                  <NA>               <NA>
    ##          active_principle_9 AAdjusted   CO2_per Respiration_rate  plate
    ## mr1_s.49               <NA> 0.9550970 0.1859916        0.3638966 Test_3
    ## mr1_s.50               <NA> 1.0277779 0.1156414        0.2262549 Test_3
    ## mr1_s.51               <NA> 1.0345131 0.1098292        0.2148832 Test_3
    ## mr1_s.52               <NA> 1.0830538 0.0709064        0.1387299 Test_3
    ## mr1_s.61               <NA> 0.5981942 1.0329617        2.0210120 Test_3
    ## mr1_s.62               <NA> 0.9992496 0.1415056        0.2768587 Test_3

Export the fully processed, unfiltered dataset:

``` r
write.csv(merged_samples, "microresp_processed.csv", row.names = F) # Export as CSV file. The simplest but looses some structure, like factor columns.
saveRDS(merged_samples,"microresp_processed.rds") # Export RDS file, which is an exact copy of the R object but is only readable by specialised software.
```

The exported dataset contains:

- **Measurements** made for all samples (calibration points are
  excluded) in all plates included in the imported xlsx file.

- **Metadata** for all reads (provided sample IDs in results xlsx file
  match variables from metadata xlsx file).

- Estimation of **%CO2** and **microbial respiration rate** for each
  measurement based on a specific model selected in step 1.3.1).

## 2) Plotting data and model

### 2.1) Plot Model(s)

``` r
library(ggplot2)
library(ggpubr)
library(tidyr)
library(ggprism)

# define some auxiliary variables

active_model_x <- active_model[["call"]][["formula"]][[3]][[3]][[3]][[2]][[3]][[3]] # Extracts the name used for the X variable in the model (the absorbance)
active_model_y <- active_model[["call"]][["formula"]][[2]] # Extracts the name used for the Y variable in the model (the %CO2)
active_model_train_data <- active_model$data # Extracts the name of the dataframe used to train the model

## Color palette for this curve set

# Starting palette
cal_palette <- c("#9E0142", "#D53E4F", "#F46D43", "#FDAE61", "#FEE08B", "#E6F598", "#ABDDA4", "#66C2A5", "#3288BD", "#5E4FA2")
# Create color function from that palette
col_fn <- colorRampPalette(cal_palette)
# Create a given number of colors from the color function
cal_palette <- col_fn(10)
# Add names to the colors
names(cal_palette) <- paste0("con_plate",1:length(cal_palette))
# Add samples and main calibration curve
# Add "soil" and "cal"
cal_palette["soil"] <- "#FF0000"  # Red
cal_palette["cal"] <- "#0000FF"   # Blue
  
  # c("cal_plate" = "#47D7AC", 
  #            "soil" = "#2761C4", 
  #            "con_plate1" = "#CC4389", 
  #            "con_plate2" = "#EB8900", 
  #            "con_plate3" = "#FAD847")

# Only curves plot

ggplot(mr1, aes(y = CO2_per, x = AAdjusted)) +
  # Add calibration points layer
  geom_point(aes(color = "cal"), data = mr1_c) +
  # Add calibration model line layer
  geom_line(aes(y = fitted(mod_mr1_c), # gets the fitted data for the active model
                color = "cal"), 
            data = mr1_c) + # gets the dataframe that was used to train the model
  # Add calibration points layer
  geom_point(aes(color = "con_plate1"), data = mr2_c) +
  # Add calibration model line layer
  geom_line(aes(y = fitted(mod_mr2_c), # gets the fitted data for the active model
                color = "con_plate1"),
            data = mr2_c) + # gets the dataframe that was used to train the model
  # Add calibration points layer
  geom_point(aes(color = "con_plate2"), data = mr3_c) +
  # Add calibration model line layer
  geom_line(aes(y = fitted(mod_mr3_c), # gets the fitted data for the active model
                color = "con_plate2"), 
            data = mr3_c) + # gets the dataframe that was used to train the model
  # Add calibration points layer
  geom_point(aes(color = "con_plate3"), data = mr4_c) +
  # Add calibration model line layer
  geom_line(aes(y = fitted(mod_mr4_c), # gets the fitted data for the active model
                color = "con_plate3"), 
            data = mr4_c) + # gets the dataframe that was used to train the model
  # Add calibration points layer
  geom_point(aes(color = "con_plate4"), data = mr5_c) +
  # # Add calibration model line layer
  geom_line(aes(y = fitted(mod_mr5_c), # gets the fitted data for the active model
                color = "con_plate4"),
            data = mr5_c) + # gets the dataframe that was used to train the model
  # Add calibration points layer
  geom_point(aes(color = "con_plate5"), data = mr6_c) +
  # Add calibration model line layer
  geom_line(aes(y = fitted(mod_mr6_c), # gets the fitted data for the active model
                color = "con_plate5"),
            data = mr6_c) + # gets the dataframe that was used to train the model
  # Add calibration points layer
  geom_point(aes(color = "con_plate6"), data = mr7_c) +
  # Add calibration model line layer
  geom_line(aes(y = fitted(mod_mr7_c), # gets the fitted data for the active model
                color = "con_plate6"),
            data = mr7_c) + # gets the dataframe that was used to train the model
  # Add a line at %CO2 = 0
  geom_hline(yintercept = 0, color = "red") + 
  scale_color_manual(values = cal_palette) +
  theme_prism() + 
  labs(x = "Normalised A570", y = "Theoretical CO2 concentration (%)", title = "Calibration curves")
```

![](Microresp_pipeline_files/figure-gfm/unnamed-chunk-22-1.png)<!-- -->

``` r
## Plot for curve from active model and all samples

ggplot(mr1, aes(y = CO2_per, x = AAdjusted)) +
  # Add calibration points layer
  geom_point(aes(x = get(active_model_x), # gets the name used for the X variable in the model (the absorbance)
                 y = get(active_model_y), # gets the fitted data for the active model
                 color = "cal"), 
             data = mr1_c) + # gets the dataframe that was used to train the model
  # Add calibration model line layer
  geom_line(aes(x = get(active_model_x), # gets the name used for the X variable in the model (the absorbance)
                y = fitted(active_model), # gets the fitted data for the active model
                color = "cal"), 
            data = mr1_c) + # gets the dataframe that was used to train the model
  # Add sample points
  geom_point(aes(y = CO2_per, x = AAdjusted, color = "soil", shape = plate
                 ), data = merged_samples) +
  ## Restrict plot window (if wanted)
  # scale_y_continuous(breaks = seq(-5,30,5), minor_breaks = seq(-5,30,0.5), limits = c(-2,5)) +
  # scale_x_continuous(breaks = seq(0,10,0.25), limits = c(0.25, 1.5)) +
  # Add a line at %CO2 = 0
  geom_hline(yintercept = 0, color = "red", aes(alpha = 0.5)) + 
  scale_color_manual(values = cal_palette) +
  theme_prism() + 
  labs(x = "Normalised A570", y = "Theoretical CO2 concentration (%)", title = "calibration curve")
```

    ## Warning: `geom_hline()`: Ignoring `mapping` because `yintercept` was provided.

    ## Warning: The shape palette can deal with a maximum of 6 discrete values because more
    ## than 6 becomes difficult to discriminate
    ## ℹ you have requested 17 values. Consider specifying shapes manually if you need
    ##   that many have them.

    ## Warning: Removed 794 rows containing missing values or values outside the scale range
    ## (`geom_point()`).

![](Microresp_pipeline_files/figure-gfm/unnamed-chunk-22-2.png)<!-- -->

### 2.2) Plot samples

#### 2.2.1) Sanity check per plate

``` r
ggplot(filter(merged_samples, 
              #plate != "Plate 1" & plate != "Test_3"
              )
       , aes(x = sample, y = CO2_per, color = plate)) +
  geom_point() +
  geom_boxplot() +
  geom_hline(yintercept = 0, color = "red") + 
  theme_prism() + 
  labs(x = "Sample", y = "CO2 concentration (%)") 
```

![](Microresp_pipeline_files/figure-gfm/unnamed-chunk-23-1.png)<!-- -->

``` r
  # facet_wrap(~plate, strip.position = "bottom", nrow = 1) +
  # theme(panel.grid.minor = element_blank(),
  #       panel.grid.major.x = element_blank(),
  #       panel.spacing = unit(0,"cm"),
  #       strip.placement = "outside")
              

## Check how many samples fall in <0 values

samples_below_0 <- filter(merged_samples, CO2_per<0)
number_wells_below_0 <- nrow(samples_below_0)
number_samples_below_0 <- length(unique(samples_below_0$sample))
id_samples_below_0 <- unique(samples_below_0[order(samples_below_0$CO2_per, decreasing = T),"sample"]) # in %CO2 decreasing order
extreme_samples <- rbind(merged_samples[order(merged_samples$CO2_per, decreasing = T)[1],], # Highest value
                         merged_samples[order(merged_samples$CO2_per, decreasing = F)[1],]) # Lowest value

# plot only samples with %CO2 <0

ggplot(filter(merged_samples, sample %in% id_samples_below_0), aes(x = sample, y = CO2_per, shape = plate)) +
  geom_point() +
  # geom_boxplot() +
  geom_hline(yintercept = 0, color = "red") + 
  theme_prism() + 
  labs(x = "Sample", y = "CO2 concentration (%)")
```

    ## Warning: The shape palette can deal with a maximum of 6 discrete values because more
    ## than 6 becomes difficult to discriminate
    ## ℹ you have requested 7 values. Consider specifying shapes manually if you need
    ##   that many have them.

    ## Warning: Removed 5 rows containing missing values or values outside the scale range
    ## (`geom_point()`).

![](Microresp_pipeline_files/figure-gfm/unnamed-chunk-23-2.png)<!-- -->

``` r
# Since all samples who's %CO2<0 were from plate 1. This is a heatmap of Plate 1

ggplot(data = rbind(mr2_s), aes(x = col, y = row, fill = CO2_per)) +
  geom_tile() + 
  geom_text(aes(label = sample), alpha = 0.3) +
  scale_fill_gradient2(low = "#A64071", mid = "white", high = "#DEBB58", midpoint = 0) +  
  theme_minimal() +
  scale_y_discrete(limits = rev(unique(merged_samples$row))) + # so the A row goes at the top, as in the actual plate
  scale_x_discrete(position = "top") + # so the col names go on the top, as in the actual plate
  # scale_y_discrete(sec.axis = dup_axis(name = "Custom Labels", labels = c("Curve 1","Curve 2","Curve 3"))) +
  coord_fixed()
```

![](Microresp_pipeline_files/figure-gfm/unnamed-chunk-23-3.png)<!-- -->

``` r
# Done for all plates

heatmaps_samples <- list()

for (i in 1:length(dfs_s)) {
  df <- get(dfs_s[i]) # get the dataframe

  # Create the plots
  heatmaps_samples[[i]] <- ggplot(data = df, aes(x = col, y = row, fill = CO2_per)) +
    geom_tile() + 
    geom_text(aes(label = sample), alpha = 0.3) +
    # scale_fill_gradient(low = "#A64071", high = "#DEBB58") +
    scale_fill_gradient2(low = "#A64071", mid = "white", high = "#DEBB58", midpoint = 0) +
    theme_minimal() +
    scale_y_discrete(limits = rev(unique(merged_samples$row))) + # so the A row goes at the top, as in the actual plate
    scale_x_discrete(position = "top") + # so the col names go on the top, as in the actual plate
    # ggtitle(excel_sheets(sourcefile)[i]) +
    coord_fixed() 
    
  
  # Wrap up
  
  rm(df,i) # remove auxiliary objects
}

# library(gridExtra)
# grid.arrange(grobs = heatmaps_samples, ncol = 4) 
# print(heatmaps_samples)

ggplot(data = mr3_s, aes(x = col, y = row, fill = CO2_per)) +
  geom_tile() + 
  geom_text(aes(label = sample), alpha = 0.3) +
  scale_fill_gradient2(low = "#A64071", mid = "white", high = "#DEBB58", midpoint = 0) +  
  theme_minimal() +
  scale_y_discrete(limits = rev(unique(merged_samples$row))) + # so the A row goes at the top, as in the actual plate
  scale_x_discrete(position = "top") + # so the col names go on the top, as in the actual plate
  # scale_y_discrete(sec.axis = dup_axis(name = "Custom Labels", labels = c("Curve 1","Curve 2","Curve 3"))) +
  coord_fixed() 
```

![](Microresp_pipeline_files/figure-gfm/unnamed-chunk-23-4.png)<!-- -->

``` r
# Position dispersion plot
ggplot(merged_samples, aes(x = well, y = AAdjusted)) +
  geom_point(aes(color = sample), data = merged_samples) +
  geom_point(aes(color = sample), data = mr1_c) + 
  labs(y = "Normalised A570")
```

![](Microresp_pipeline_files/figure-gfm/unnamed-chunk-23-5.png)<!-- -->

``` r
# Position heatmap

 ggplot(data = merged_samples, aes(x = col, y = row, fill = AAdjusted)) +
  geom_tile() +
  scale_fill_gradient(low = "#A64071", high = "#DEBB58") + 
  theme_minimal() +
  scale_y_discrete(limits = sort(unique(merged_samples$row), decreasing=T)) + # so the A row goes at the top, as in the actual plate
  scale_x_discrete(position = "top") + # so the col names go on the top, as in the actual plate
  coord_fixed() 
```

![](Microresp_pipeline_files/figure-gfm/unnamed-chunk-23-6.png)<!-- -->

Later: test effect of the curve with lmer()

## 3) Data analysis

### 3.0) Import dataset

``` r
if (exists("merged_samples")){
  print("Dataset was already loaded")
  } else { merged_samples <- readRDS("microresp_processed.rds")
print("Dataset was loaded from microresp_processed.rds file")
}
```

    ## [1] "Dataset was already loaded"

### 3.1) Sub-setting like crazy: Relevant filters based on analyses performed in 2)

#### 3.1.1) Filter out unwanted reads and group data by relevant variables

``` r
working_reads <- merged_samples %>%
  filter(plate != "Plate 1" & plate != "Test_3" & col != "3") %>% # Filtering out Plate 1 and Test 3 plates, and the column next to the calibration curve
  group_by(soil, treatment, plant) # group by relevant variables
```

#### 3.1.2) Average multiple Microresp reads from the same sample

``` r
# Average spectrophotometric readings and subsequent calculations.

working_samples <- working_reads %>%
    group_by(sample) %>%
    summarize(
      A570_0h = mean(A570_0h, na.rm = TRUE), # Absorbance before exposure
      A570_6h = mean(A570_6h, na.rm = TRUE), # Absorbance after exposure
      AAdjusted = mean(AAdjusted, na.rm = TRUE), # Normalised absorbance
      CO2_per = mean(CO2_per, na.rm = TRUE), # %CO2
      Respiration_rate = mean(Respiration_rate, na.rm = TRUE), # Respiration rate
    ) 

# Extract the data that comes from other sources (metadata), which is the same for all readings of the same sample.

working_meta <- working_reads %>%
  group_by(sample) %>%
  slice(1) %>% # Take the first row for each sample to get metadata
  select(-c(A570_0h, A570_6h, AAdjusted, CO2_per, Respiration_rate)) # Remove averaged columns

# Check back consistency and re-add grouping

if (identical(working_samples$sample,working_meta$sample)) { # First, check if the samples extracted from both processes match exactly
  working_data <- left_join(working_samples, working_meta, by = "sample")
  working_data <- group_by(working_data, soil, treatment, plant) # group by relevant variables)
} else { 
  print("The samples in the reads and metadata don't match! Check for NA values")
    }

head(working_data)
```

    ## # A tibble: 6 × 32
    ## # Groups:   soil, treatment, plant [6]
    ##   sample A570_0h A570_6h AAdjusted CO2_per Respiration_rate well  row   col  
    ##   <chr>    <dbl>   <dbl>     <dbl>   <dbl>            <dbl> <chr> <chr> <fct>
    ## 1 S100      1.43   1.12       1.10  0.0588            0.115 B4    B     4    
    ## 2 S101      1.34   0.998      1.05  0.0954            0.187 C7    C     7    
    ## 3 S102      1.36   1.03       1.06  0.0863            0.169 C4    C     4    
    ## 4 S103      1.36   1.01       1.04  0.103             0.201 D8    D     8    
    ## 5 S104      1.35   0.969      1.01  0.129             0.253 D4    D     4    
    ## 6 S105      1.44   1.06       1.04  0.114             0.224 E8    E     8    
    ## # ℹ 23 more variables: plant <chr>, soil <chr>, treatment <fct>,
    ## #   replicate <chr>, origin_location <chr>, soil_texture <chr>,
    ## #   CaCO3_perc <dbl>, NOM_perc <dbl>, pH <dbl>, `P-CaCl2_mgP/kg` <dbl>,
    ## #   `P-AL_mgP/kg` <dbl>, applied_product <fct>, target_function <chr>,
    ## #   active_principle_1 <chr>, active_principle_2 <chr>,
    ## #   active_principle_3 <chr>, active_principle_4 <chr>,
    ## #   active_principle_5 <chr>, active_principle_6 <chr>, …

#### 3.1.3) Export biologically relevant dataset, with filtered noise and averaged technical replicates

``` r
write.csv(working_data, "microresp_biofiltered.csv", row.names = F) # Export as CSV file. The simplest but looses some structure, like factor columns.
saveRDS(working_data,"microresp_biofiltered.rds") # Export RDS file, which is an exact copy of the R object but is only readable by specialised software.
```

#### 3.1.4) Other relevant sub-settings

### 3.2) Further analysis

#### 3.2.1) Respiration rate for all samples

``` r
# 5 color palette 
palette_5col <- c("#EB8900", "#47D7AC", "#FAD847", "#2761C4", "#962C5D", "#CC4389")
# palette_5col <- c("#EB8900", "#2761C4", "#47D7AC", "#CC4389", "#FAD847")

# Treatment palettes based on the 5 color palette
palette_treatments <- palette_5col
names(palette_treatments) <- levels(treatment_data$label)

palette_products <- palette_5col
names(palette_products) <- levels(treatment_data$applied_product)


# palette_treatments <- c("Control" = "#EB8900", 
#                         "AMF" = "#2761C4", 
#                         "Disease suppression" = "#47D7AC",  
#                         "Nitrogen fixation" = "#CC4389", 
#                         "Phosphate solubilisation" = "#FAD847")
# 
# palette_products <- c("No product" = "#EB8900", 
#                         "MycorGran 2.0" = "#2761C4", 
#                         "Compete Plus" = "#47D7AC",  
#                         "Vixeran" = "#CC4389", 
#                         "NuelloPhos" = "#FAD847")
 
ggplot(filter(working_data, plant == plant_data$label[1]) , aes(x = origin_location, y = Respiration_rate, color = applied_product)) +
  # geom_point() +
  geom_boxplot() +
  # geom_col(stat = "identity", position = "dodge") +
  theme_prism() + 
  scale_y_continuous(limits = c(0,1)) + 
  scale_color_manual(values = palette_products) +
  labs(x = "", y = expression(paste("Respiration rate (", mu, "gCO"[2], "-Cg"^"-1", "h"^"-1", ")"))) 
```

    ## Warning: Removed 1 row containing non-finite outside the scale range
    ## (`stat_boxplot()`).

![](Microresp_pipeline_files/figure-gfm/unnamed-chunk-29-1.png)<!-- -->

## 4) Statistics

``` r
library(tidyverse)
```

    ## ── Attaching core tidyverse packages ──────────────────────── tidyverse 2.0.0 ──
    ## ✔ forcats   1.0.0     ✔ readr     2.1.5
    ## ✔ lubridate 1.9.4     ✔ tibble    3.2.1
    ## ✔ purrr     1.0.4     
    ## ── Conflicts ────────────────────────────────────────── tidyverse_conflicts() ──
    ## ✖ dplyr::filter() masks stats::filter()
    ## ✖ dplyr::lag()    masks stats::lag()
    ## ℹ Use the conflicted package (<http://conflicted.r-lib.org/>) to force all conflicts to become errors

``` r
library(agricolae)

anova <- aov(Respiration_rate ~ plant * treatment * soil, data = working_data)
summary(anova)
```

    ##                       Df Sum Sq Mean Sq F value   Pr(>F)    
    ## plant                  2  2.216  1.1082  17.258 1.68e-07 ***
    ## treatment              4  0.049  0.0123   0.192    0.942    
    ## soil                   3  4.484  1.4946  23.277 1.61e-12 ***
    ## plant:treatment        8  0.071  0.0089   0.138    0.997    
    ## plant:soil             6  0.573  0.0954   1.486    0.186    
    ## treatment:soil        12  0.191  0.0159   0.247    0.995    
    ## plant:treatment:soil  24  0.303  0.0126   0.196    1.000    
    ## Residuals            157 10.081  0.0642                     
    ## ---
    ## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1

``` r
# Check assumptions (important!)
# a. Normality of residuals
plot(anova, 2) # Q-Q plot
```

    ## Warning: not plotting observations with leverage one:
    ##   99

![](Microresp_pipeline_files/figure-gfm/unnamed-chunk-30-1.png)<!-- -->

``` r
# b. Homogeneity of variance (Levene's test)
library(car)
```

    ## Loading required package: carData
    ## 
    ## Attaching package: 'car'
    ## 
    ## The following object is masked from 'package:purrr':
    ## 
    ##     some
    ## 
    ## The following object is masked from 'package:gtools':
    ## 
    ##     logit
    ## 
    ## The following object is masked from 'package:dplyr':
    ## 
    ##     recode

``` r
leveneTest(Respiration_rate ~ plant * treatment * soil, data = working_data)
```

    ## Levene's Test for Homogeneity of Variance (center = median)
    ##        Df F value Pr(>F)
    ## group  59  1.1788 0.2113
    ##       157

``` r
# Example: Tukey for treatment, assuming it was significant in the ANOVA
# HSD.test(working_data, "treatment", alpha=0.05)

# length(unique(working_samples$sample)) # How many unique samples are present in the final dataset?
unique(working_data$plant)
```

    ## [1] "Bare soil"   "Faba bean"   "Mixed grass"
