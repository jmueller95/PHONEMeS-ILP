# PHONEMeS-ILP
ILP implementation of PHONEMeS - Enio GJERGA

**PHONEMeS** (**PHO**sphorylation **NE**tworks for **M**ass **S**pectrometry) is a method to model signalling networks based on untargeted phosphoproteomics mass spectrometry data and kinase/phosphatase-substrate interactions. 
Please see [Terfve et al.](http://www.nature.com/articles/ncomms9033) for an explanation of the methodolgy.

This repository contains the scripts for the ILP (Integer Linear Programming) implementation of the [PHONEMeS R package](https://github.com/saezlab/PHONEMeS/tree/master/Package) and accompanying scripts that implement the method. ILP is a mathematical optimisation algorithm in which the objective function and constraints are linear and the variables are integers.

### License

Distributed under the GNU GPLv3 License. See accompanying file [LICENSE.txt](https://github.com/saezlab/PHONEMeS-ILP/blob/master/LICENSE).

### Installation

Before using the method, please install the current R package for [PHONEMeS](https://github.com/saezlab/PHONEMeS). For installation, download the tar file of the package and type in R:

```R
# Install PHONEMeS from Github using devtools
# install.packages('devtools') # in case devtools hasn't been installed
library(devtools)
install_github('saezlab/PHONEMeS-ILP')
# or download the source file from GitHub and install from source
install.packages('path_to_extracted_CARNIVAL_directory', repos = NULL, type="source")
```

## Running PHONEMeS

The PHONEMeS library can be initialized by:

```R
library(PHONEMeS)
```

A list of tutorials/examples of the PHONEMeS package, can be found [here](https://github.com/saezlab/PHONEMeS-ILP-Examples)

### Prerequisites

PHONEMeS requires the interactive version of IBM Cplex or CBC-COIN solver as the network optimiser. The IBM ILOG Cplex is freely available through Academic Initiative [here](https://www.ibm.com/products/ilog-cplex-optimization-studio?S_PKG=CoG&cm_mmc=Search_Google-_-Data+Science_Data+Science-_-WW_IDA-_-+IBM++CPLEX_Broad_CoG&cm_mmca1=000000RE&cm_mmca2=10000668&cm_mmca7=9041989&cm_mmca8=kwd-412296208719&cm_mmca9=_k_Cj0KCQiAr93gBRDSARIsADvHiOpDUEHgUuzu8fJvf3vmO5rI0axgtaleqdmwk6JRPIDeNcIjgIHMhZIaAiwWEALw_wcB_k_&cm_mmca10=267798126431&cm_mmca11=b&mkwid=_k_Cj0KCQiAr93gBRDSARIsADvHiOpDUEHgUuzu8fJvf3vmO5rI0axgtaleqdmwk6JRPIDeNcIjgIHMhZIaAiwWEALw_wcB_k_|470|135655&cvosrc=ppc.google.%2Bibm%20%2Bcplex&cvo_campaign=000000RE&cvo_crid=267798126431&Matchtype=b&gclid=Cj0KCQiAr93gBRDSARIsADvHiOpDUEHgUuzu8fJvf3vmO5rI0axgtaleqdmwk6JRPIDeNcIjgIHMhZIaAiwWEALw_wcB). The [CBC](https://projects.coin-or.org/Cbc) solver is open source and freely available for any user. 

### References

[Terfve et al.](http://www.nature.com/articles/ncomms9033):

> Terfve, C. D. A., Wilkes, E. H., Casado, P., Cutillas, P. R., and Saez-Rodriguez, J. (2015). Large-scale models of signal propagation in human cells derived from discovery phosphoproteomic data. *Nature Communications*, 6:8033.

[Wilkes et al.](http://www.pnas.org/content/112/25/7719.abstract) (description of parts of the data)

> Wilkes, E. H., Terfve, C., Gribben, J. G., Saez-Rodriguez, J., and Cutillas, P. R. (2015). Empirical inference of circuitry and plasticity in a kinase signaling network. *Proceedings of the National Academy of Sciences of the United States of America,* 112(25):7719–24.

## Examples

### MTORi and PI3Ki-AKTi-MTORi Perturbation Analysis

Here we show the example from [Terfve et al.](http://www.nature.com/articles/ncomms9033) where PHONEMeS was used to model the perturbation effects of MTOR inhibition. 

We start first by loading the required packages and the necessary PHONEMeS inputs:

```R
# Load packages
library(BioNet)
library(igraph)
library(PHONEMeS)
library(hash)
library(dplyr)
library(readxl)
library(readr)

# Loading database and data-object
load(file = system.file("NetworKIN_noCSK_filt.RData", package="PHONEMeS"))
load(file = system.file("inputObj_Terfve.RData", package="PHONEMeS"))

```

Next we prepare the data objwcts which will be used as an input by PHONEMeS.

```R
# Preparation of the background network from the allD list of K/P-S interactions
bg<-new("KPSbg", interactions=allD, species=unique(c(allD$K.ID, allD$S.cc)))

# Creating the list where we show the experimental conditions
conditions <- list(c("AKT1 - Control", "AKT2 - Control"), c("CAMK1 - Control", "CAMK2 - Control"),
                   c("EGFR1 - Control", "EGFR2 - Control"), c("ERK1 - Control", "ERK2 - Control"),
                   c("MEK1 - Control", "MEK2 - Control"), c("MTOR1 - Control", "MTOR2 - Control"),
                   c("P70S6K1 - Control", "P70S6K2 - Control"), c("PI3K1 - Control", "PI3K2 - Control"),
                   c("PKC1 - Control", "PKC2 - Control"), c("ROCK1 - Control", "ROCK2 - Control"))

names(conditions) <- c("AKT1_HUMAN", "KCC2D_HUMAN", "EGFR_HUMAN", "MK01_HUMAN",
                       "MP2K1_HUMAN", "MTOR_HUMAN",  "KS6B1_HUMAN", "PK3CA_HUMAN",
                       "KPCA_HUMAN", "ROCK1_HUMAN")

# For each experimental condition we assign a perturbation target
targets.P<-list(cond1=c("AKT1_HUMAN", "AKT2_HUMAN"), cond2=c("KCC2A_HUMAN", "KCC2B_HUMAN", "KCC2C_HUMAN", "KCC2D_HUMAN"), cond3=c("EGFR_HUMAN", "ERBB2_HUMAN"), 
                cond4=c("MK01_HUMAN", "MK03_HUMAN", "MK14_HUMAN"), cond5=c("MP2K1_HUMAN", "MP2K2_HUMAN"), cond6=c("MTOR_HUMAN"), cond7=c("KS6B1_HUMAN", "KS6B2_HUMAN"), 
                cond8=c("PK3CA_HUMAN", "PK3CD_HUMAN", "MTOR_HUMAN"), cond9=c("KPCA_HUMAN", "KPCB_HUMAN", "KPCG_HUMAN", "KPCE_HUMAN"), cond10=c("ROCK1_HUMAN", "ROCK2_HUMAN"))

```

Then finally we perform the PHONEMeS analysis for the MTOR inhibition experiment (condition 6).

```R
# Select experimental condition
experiments <- c(6) # for MTORi case

# Running PHONEMeS - cplex
# Run PHONEMeS with multiple solutions from CPLEX
resultsMulti <- runPHONEMeS(targets.P = targets.P, conditions = conditions, inputObj = inputObj, experiments = experiments, bg = bg, solver = "cplex", nSolutions = 100, nK = "no", solverPath = path_to_executable_solver)
write.table(x = resultsMulti, file = "MTORi_sif_cplex.txt", quote = FALSE, sep = "\t", row.names = FALSE, col.names = TRUE)

nodesAttributes <- assignAttributes(sif = resultsMulti, dataGMM = inputObj, targets = targets.P[experiments], writeAttr = TRUE)

```

For running PHONEMeS by considering multiple conditions we can consider the case where we wish to retrieve consensus network solutions for when combining evidences from the PI3Ki-AKTi-MTORi inhibition experimental conditions. This analysis can be performed either by requesting multiple solutions directly from the CPLEX options or by performing PHONEMeS analysis multiple times by randomly downsampling the measurements and retrieving one solution for each iteration. These single solutions are then integrated into a combined network. With increasing number of experimental conditions, the number of possible solutions is expected to increase substantially and the Downsampling porcedure is advised to be used for these situations since it integrates single solutions in a more unbiased manner and the integrated network is sparser.

```R
# Select experimental condition
experiments <- c(1, 6, 8) # for PI3Ki-AKTi-MTORi case

# Running PHONEMeS - cplex
# Run PHONEMeS with multiple solutions from CPLEX
resultsMulti <- runPHONEMeS(targets.P = targets.P, conditions = conditions, inputObj = inputObj, experiments = experiments, bg = bg, solver = "cplex", nSolutions = 100, nK = "no", populate = 100, solverPath = path_to_executable_solver)
write.table(x = resultsMulti, file = "PI3Ki_AKTi_MTORi_sif_cplex.txt", quote = FALSE, sep = "\t", row.names = FALSE, col.names = TRUE)
resultsMulti <- runPHONEMeS_Downsampling(targets.P = targets.P, conditions = conditions, inputObj = inputObj, experiments = experiments, bg = bg, nIter = 100, nK = "no")
write.table(x = resultsMulti, file = "PI3Ki_AKTi_MTORi_sif_downsampling.txt", quote = FALSE, sep = "\t", row.names = FALSE, col.names = TRUE)

nodesAttributes <- assignAttributes(sif = resultsMulti, dataGMM = inputObj, targets = targets.P[experiments], writeAttr = TRUE)

```

### Time-Point Analysis for the Modelling of Endothelin Signalling

Here we show an example about how to perform PHONEMeS analsysis over phosphoproteomics time-course data. The example refers to the study from [Schaefer et al. 2019](https://www.embopress.org/doi/full/10.15252/msb.20198828). The PHONEMeS inputs used for this example have been prepared as described in detail in the separate [dedicated github repository](https://github.com/saezlab/EDN_phospho) of this study.

We start first by loading the required packages and the necessary PHONEMeS inputs for this example:

```R
# Load packages
library(BioNet)
library(igraph)
library(PHONEMeS)
library(hash)
library(dplyr)
library(readxl)
library(readr)

# Loading database and data-object
load(file = system.file("interactions_schaefer.RData", package="PHONEMeS"))
load(file = system.file("dataUACC257.RData", package="PHONEMeS"))

```

Next we prepare the data objwcts which will be used as an input by PHONEMeS.

```R
# Preparing background network as a PHONEMeS input
bg<-new("KPSbg", interactions=allD, species=unique(c(allD$K.ID, allD$S.cc)))
dataInput<-new("GMMres", res=GMM, IDmap=GMM.ID, resFC=GMM.wFC)

# Choose the conditions for each time-point
conditions <- list(c("tp_2min"), c("tp_10min"), c("tp_30min"), c("tp_60min"), c("tp_90min"))
names(conditions) <- c("tp_2min", "tp_10min", "tp_30min", "tp_60min", "tp_90min")

# Choose the targets for each time-point (EDNRB - the same)
targets.P <- list(tp_2min=c("EDNRB_HUMAN"), tp_10min=c("EDNRB_HUMAN"), tp_30min=c("EDNRB_HUMAN"), tp_60min=c("EDNRB_HUMAN"), tp_90min=c("EDNRB_HUMAN"))

# Next we assign the experimental conditions from which we get the measurements at each specific time-point
# In this case we have the same one experimental condition for each time-point
experiments <- list(tp1=c(1), tp2=c(2), tp3=c(3), tp4=c(4), tp5=c(5))

```

Next we perform the PHONEMeS analysis to obtain the time-course modelling of signalling. We perform this analysis 100 times where for each iteration we retain a random sample of measurements. Each solution at each iteration contains one time-course signalling model which in the end is then combined into one integrated network.

```R
# Running multiple time-point variant of PHONEMeS and retain only those interactions which have a weight higher than 20/appear at least 20 times in the
# separate solutions we have obtained out of the 100 runs we have set to perform (nIter=100).
set.seed(383789)
resultsMulti = runPHONEMeS_mult(targets.P = targets.P, conditions = conditions, inputObj = inputObj, experiments = experiments, bg = bg, nIter = 100)
nodeAttribudes <- assignAttributes(sif = resultsMulti[which(resultsMulti[, 2]>=20), ], dataGMM = inputObj, targets = targets.P, writeAttr = FALSE)

write.table(x = resultsMulti[which(resultsMulti[, 2]>=20), ], file = "ednrb_network.txt", quote = FALSE, sep = "\t", row.names = FALSE, col.names = TRUE)
write.table(x = nodeAttribudes, file = "ednrb_attributes.txt", quote = FALSE, sep = "\t", row.names = FALSE, col.names = TRUE)
```

### PHONEMeS Upside-Down Analysis