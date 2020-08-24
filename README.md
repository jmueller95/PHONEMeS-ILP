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

Here we show the example from [Terfve et al.](http://www.nature.com/articles/ncomms9033) where PHONEMeS was used to model the perturbation effects of MTOR inhibition. We start first by loading the required packages and the necessary PHONEMeS inputs:

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

Next we...

```R
#Make the data objects that will be needed
bg<-new("KPSbg", interactions=allD, species=unique(c(allD$K.ID, allD$S.cc)))

conditions <- list(c("AKT1 - Control", "AKT2 - Control"), c("CAMK1 - Control", "CAMK2 - Control"),
                   c("EGFR1 - Control", "EGFR2 - Control"), c("ERK1 - Control", "ERK2 - Control"),
                   c("MEK1 - Control", "MEK2 - Control"), c("MTOR1 - Control", "MTOR2 - Control"),
                   c("P70S6K1 - Control", "P70S6K2 - Control"), c("PI3K1 - Control", "PI3K2 - Control"),
                   c("PKC1 - Control", "PKC2 - Control"), c("ROCK1 - Control", "ROCK2 - Control"))

names(conditions) <- c("AKT1_HUMAN", "KCC2D_HUMAN", "EGFR_HUMAN", "MK01_HUMAN",
                       "MP2K1_HUMAN", "MTOR_HUMAN",  "KS6B1_HUMAN", "PK3CA_HUMAN",
                       "KPCA_HUMAN", "ROCK1_HUMAN")

targets.P<-list(cond1=c("AKT1_HUMAN", "AKT2_HUMAN"), cond2=c("KCC2A_HUMAN", "KCC2B_HUMAN", "KCC2C_HUMAN", "KCC2D_HUMAN"), cond3=c("EGFR_HUMAN", "ERBB2_HUMAN"), 
                cond4=c("MK01_HUMAN", "MK03_HUMAN", "MK14_HUMAN"), cond5=c("MP2K1_HUMAN", "MP2K2_HUMAN"), cond6=c("MTOR_HUMAN"), cond7=c("KS6B1_HUMAN", "KS6B2_HUMAN"), 
                cond8=c("PK3CA_HUMAN", "PK3CD_HUMAN", "MTOR_HUMAN"), cond9=c("KPCA_HUMAN", "KPCB_HUMAN", "KPCG_HUMAN", "KPCE_HUMAN"), cond10=c("ROCK1_HUMAN", "ROCK2_HUMAN"))

```

Then finally...

```R
# Select experimental condition
experiments <- c(6) # for MTORi case

# Running PHONEMeS - cplex
# Run PHONEMeS with multiple solutions from CPLEX
resultsMulti <- runPHONEMeS(targets.P = targets.P, conditions = conditions, inputObj = inputObj, experiments = experiments, bg = bg, solver = "cplex", nSolutions = 100, nK = "no")
write.table(x = resultsMulti, file = "MTORi_sif_cplex.txt", quote = FALSE, sep = "\t", row.names = FALSE, col.names = TRUE)

nodesAttributes <- assignAttributes(sif = resultsMulti, dataGMM = inputObj, targets = targets.P[experiments], writeAttr = TRUE)

```

For multiple conditions...

```R
# Select experimental condition
experiments <- c(1, 6, 8) # for PI3Ki-AKTi-MTORi case

# Running PHONEMeS - cplex
# Run PHONEMeS with multiple solutions from CPLEX
resultsMulti <- runPHONEMeS(targets.P = targets.P, conditions = conditions, inputObj = inputObj, experiments = experiments, bg = bg, solver = "cplex", nSolutions = 100, nK = "no", populate = 100)
write.table(x = resultsMulti, file = "PI3Ki_AKTi_MTORi_sif_cplex.txt", quote = FALSE, sep = "\t", row.names = FALSE, col.names = TRUE)
resultsMulti <- runPHONEMeS_Downsampling(targets.P = targets.P, conditions = conditions, inputObj = inputObj, experiments = experiments, bg = bg, nIter = 100, nK = "no")
write.table(x = resultsMulti, file = "PI3Ki_AKTi_MTORi_sif_downsampling.txt", quote = FALSE, sep = "\t", row.names = FALSE, col.names = TRUE)

nodesAttributes <- assignAttributes(sif = resultsMulti, dataGMM = inputObj, targets = targets.P[experiments], writeAttr = TRUE)

```

### Time-Point Analysis
