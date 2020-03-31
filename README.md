
<!-- README.md is generated from README.Rmd. Please edit that file -->

# infohet

<!-- badges: start -->

<!-- badges: end -->

The goal of infohet is to calculate the information in heterogenity of
scRNA-seq data.

## Installation

And the development version from [GitHub](https://github.com/) with:

``` r
# install.packages("devtools")
devtools::install_github("mcaseySoton/infohet")
```

## Example

This is a basic example which shows you how to solve a common problem:

``` r
library(infohet)
library(ggplot2)
load("~/OneDrive - University of Southampton/Data/10x/CountsMatrix")
load("~/OneDrive - University of Southampton/Data/10x/Identity")

MinTotal <- 100
TestTotalStep <- 0.2
NumTrials <- 50
InfoThreshold <- 0.25

Total <- Matrix::rowSums(CountsMatrix)
if(any(Total < MinTotal)){
   CountsMatrix <- CountsMatrix[-which(Total < MinTotal),]
   Total <- Total[-which(Total < MinTotal)]
}

Range <- range(log10(Total))
TotalCounts <- round(10^seq(1, Range[2], TestTotalStep))


Het <- get_Het(CountsMatrix)
HetAdj <- subtract_HetSparse(Het, CountsMatrix)

NullHet <- simulate_Hom(CountsMatrix, TotalCounts, NumTrials)
NullHet <- subtract_HetSparse(NullHet, CountsMatrix)

NullHetUnif <- simulate_Hom(CountsMatrix, TotalCounts, NumTrials, depth_adjusted = F)
NullHetUnif <- subtract_HetSparse(NullHetUnif, CountsMatrix)

Threshold <- NullHet+InfoThreshold

HetDataFrame <- data.frame(log10(Total), Het, HetAdj, NullHet, Threshold, HetAdj > Threshold, NullHetUnif)
colnames(HetDataFrame) <- c("log10_Total_nUMI", "Het", "Total_Heterogeneity", "Null_Model", "Threshold", "Selected", "Uniform_Null")

OG <- ggplot(HetDataFrame, aes(x = log10_Total_nUMI, y = Total_Heterogeneity, colour = Selected)) + geom_point() +
  geom_line(aes(y = Null_Model), colour = "black", size = 1.5) +
  geom_line(aes(y = Threshold), colour = "purple", size = 1.5) +
  geom_line(aes(y = Uniform_Null), colour = "green", size = 1.5) +
  theme(legend.position = "none") +
  ylab("Het")
OG
```

<img src="man/figures/README-example-1.png" width="100%" />

``` r
library(patchwork)
Grouping <- Identity
  
GroupedCounts <- group_Counts(CountsMatrix, Grouping)
HetMacro <- get_HetMacro(CountsMatrix, Grouping, GroupedCounts)
HetMicro <- get_HetMicro(CountsMatrix, Grouping, GroupedCounts, reduced = F)
HetMicroAdj <- subtract_HetSparse(HetMicro[,"Average"], CountsMatrix)

HetDataFrame <- cbind(HetDataFrame, HetMacro, HetMicro[,"Average"], HetMicroAdj)
colnames(HetDataFrame)[ncol(HetDataFrame)-1] <- c("HetMicro")


NumPermutes <- 1
HetMacroPermutedMat <- matrix(NA, nrow = nrow(CountsMatrix), ncol = NumPermutes)
HetMicroPermutedMat <- matrix(NA, nrow = nrow(CountsMatrix), ncol = NumPermutes)
HetMicroPermutedAdjMat <- matrix(NA, nrow = nrow(CountsMatrix), ncol = NumPermutes)

for(i in 1:NumPermutes){
  Grouping <- Grouping[sample(ncol(CountsMatrix), replace = F)]
    
  GroupedCounts <- group_Counts(CountsMatrix, Grouping)
  HetMacroPermutedMat[,i] <- get_HetMacro(CountsMatrix, Grouping, GroupedCounts)
  MicroTemp <- get_HetMicro(CountsMatrix, Grouping, GroupedCounts, reduced = F)
  HetMicroPermutedMat[,i] <- MicroTemp[,"Average"]
  HetMicroPermutedAdjMat[,i] <- subtract_HetSparse(HetMicroPermutedMat[,i], CountsMatrix)
}

HetMacroPermuted <- rowMeans(HetMacroPermutedMat)
HetMicroPermuted <- rowMeans(HetMicroPermutedMat)
HetMicroPermutedAdj <- rowMeans(HetMicroPermutedAdjMat)

HetDataFrame <- HetDataFrame[,1:10]
HetDataFrame <- cbind(HetDataFrame, HetMacroPermuted, HetMicroPermuted, HetMicroPermutedAdj)
colnames(HetDataFrame)[ncol(HetDataFrame)-1] <- "HetMicroPermuted"

Ylims <- c(min(HetMicroAdj), log(ncol(CountsMatrix)))

g1 <- OG + ylim(Ylims)
g2 <- ggplot(HetDataFrame, aes(x = log10_Total_nUMI, y = HetMicroAdj)) + geom_point() + ylim(Ylims) + xlab("") + ylab("") +
  ggtitle("Micro (Model)")
g3 <- ggplot(HetDataFrame, aes(x = log10_Total_nUMI, y = HetMacro)) + geom_point() + ylim(Ylims) + xlab("") + ylab("") +
  ggtitle("Macro (Model)")
g4 <- ggplot(HetDataFrame, aes(x = log10_Total_nUMI, y = HetMicroPermutedAdj)) + geom_point() + ylim(Ylims) + xlab("") + ylab("") +
  ggtitle("Micro (Permuted)")
g5 <- ggplot(HetDataFrame, aes(x = log10_Total_nUMI, y = HetMacroPermuted)) + geom_point() + ylim(Ylims) + xlab("") + ylab("") +
  ggtitle("Macro (Permuted)")

g1 | ((g2 | g3) / (g4 | g5))
```

<img src="man/figures/README-unnamed-chunk-2-1.png" width="100%" />
