
## Model assessment
### Training-Testing random partitions
The prediction power of the model will be assessed using the training-testing (TRN-TST) random partitions approach. Data is randomly splitted into training and testing sets. Model parameters are estimated in training set and model is tested in TST set. 

## Data preparation
### Load data, generate G-matrix and create objects to store results
```r
# Load libraries
library(BGLR)
library(rrBLUP)

# Load data
data(wheat)
X <- wheat.X
Y <- wheat.Y

# Select a single environment. For instance, environment 2
y <- Y[,2]

n <- nrow(Y)
p <- ncol(X)

# Genomic relationship matrix
M <- scale(X)
G <- tcrossprod(M)/p

# Design matrix for individuals. In this case is a diagonal since there are no replicates
GID <- factor(rownames(Y),levels=rownames(Y))
Zg <- model.matrix(~GID-1)
```

## Running models

### 1. Variance components estimation
After running the 'data preparation' part, the code following code can be used to fit all models and to extract variance components and other parameters.

```r
set.seed(123)

# Matrix to store results. It will save the main parameters for each model
outVAR <- matrix(NA,nrow=6,ncol=5)
dimnames(outVAR) <- list(c("varU","varE","lambda","dfb","Sb","H2"),c("GBLUP1","GBLUP2","BRR","LASSO","BayesB"))

# Number of iterations and burn-in for Bayesian models
nIter <- 30000
burnIn <- 5000

# G-BLUP model using 'rrBLUP' package
fm <- mixed.solve(y=y,Z=Zg,K=G) 
outVAR[1,1] <- fm$Vu
outVAR[2,1] <- fm$Ve
outVAR[6,1] <- fm$Vu/(fm$Vu+fm$Ve)    # Heritability

# G-BLUP model using 'BGLR' package. Model RKHS with K=G
fm <- BGLR(y,ETA=list(list(K=G,model="RKHS")),nIter=nIter,burnIn=burnIn)
outVAR[1,2] <- fm$ETA[[1]]$varU
outVAR[2,2] <- fm$varE
outVAR[6,2] <- outVAR[1,2]/(outVAR[1,2] + outVAR[2,2])  # Heritability

# Bayesian Ridge Regression (BRR) using 'BGLR' package
fm <- BGLR(y,ETA=list(list(X=M,model="BRR")),nIter=nIter,burnIn=burnIn)
outVAR[1,3] <- fm$ETA[[1]]$varB*p    # Multiply by p to obtain the right varU as in G-BLUP
outVAR[2,3] <- fm$varE
outVAR[6,3] <- outVAR[1,3]/(outVAR[1,3] + outVAR[2,3])  # Heritability

# Bayesian LASSO model using 'BGLR' package
fm <- BGLR(y,ETA=list(list(X=M,model="BL")),nIter=nIter,burnIn=burnIn)
outVAR[2,4] <- fm$varE
outVAR[3,4] <- fm$ETA[[1]]$lambda

# Bayes B model using 'BGLR' package
fm <- BGLR(y,ETA=list(list(X=M,model="BayesB")),nIter=nIter,burnIn=burnIn)
outVAR[2,5] <- fm$varE
outVAR[4,5] <- fm$ETA[[1]]$df0
outVAR[5,5] <- fm$ETA[[1]]$S0

print(outVAR)
```

#### Results

|       |GBLUP <sup>1</sup>  | GBLUP <sup>2</sup>  | BRR | LASSO | Bayes B |
|-------|------|------|------|------|------|
|![](https://latex.codecogs.com/gif.latex?%5Csigma%5E2_u) |0.468 |0.490|0.491|  -  |  -  |
|![](https://latex.codecogs.com/gif.latex?%5Csigma%5E2_%5Cvarepsilon)  |0.574|0.576|0.575|0.589|0.572|
|![](https://latex.codecogs.com/gif.latex?%5Clambda)  |  -  |  -  |  -  |58.805|  -  |
|![](https://latex.codecogs.com/gif.latex?df_%5Cbeta)  |  -  |  -  |  -  |  -  |  5  |
|![](https://latex.codecogs.com/gif.latex?S_%5Cbeta)  |  -  |  -  |  -  |  -  |0.033|
|![](https://latex.codecogs.com/gif.latex?H%5E2_g) |0.449|0.460|0.461|  -  |  -  |

1: using 'rrBLUP'; 2: using 'BGLR' package

#
### 2. Replicates of partitions to obtain standard deviations of predictions
After running the 'data preparation' part, the code below runs repeated partitions to obtain mean and standard deviations of accuracies for all models. All the models will be run using 'BGLR' package.

```r
# Models
models <- c("GBLUP","BRR","LASSO","BayesB")

#==================================================
# User specifications
#==================================================
# Choose one model. 1:GBLUP; 2:BRR; 3:LASSO; 4:BayesB
mod <- 1

# Percentage of the data assigned to Testing set
percTST <- 0.3

# Number of replicates
m <- 10
#==================================================

# Creation of seed for repeated randomizations
set.seed(123)
seeds <- round(seq(1E3,1E6,length=m))

# Matrix to store results. It will save the corelation for each partition
outCOR <- matrix(NA,nrow=m,ncol=1)
colnames(outCOR) <- models[mod]

# Number of iterations and burn-in for Bayesian models
nIter <- 1200
burnIn <- 200

model <- models[mod]
nTST <- round(percTST*n)

for(k in 1:m)   # Loop for the replicates
{
    set.seed(seeds[k])
    indexTST <- sample(1:n,size=nTST,replace=FALSE)
    yNA <- y
    yNA[indexTST] <- NA
        
    if(model=="GBLUP")  ETA <- list(list(K=G,model="RKHS"))
    if(model=="BRR")    ETA <- list(list(X=M,model="BRR"))
    if(model=="LASSO")  ETA <- list(list(X=M,model="BL"))
    if(model=="BayesB") ETA <- list(list(X=M,model="BayesB"))

    fm <- BGLR(yNA,ETA=ETA,nIter=nIter,burnIn=burnIn)
    outCOR[k,1] <- cor(fm$yHat[indexTST],y[indexTST])
}

# Save results
save(outCOR,file=paste0("outCOR_",model,".RData"))
```

#### 2.1 Results
The code below will retrieve results for all models fitted previously

```r
models <- c("GBLUP","BRR","LASSO","BayesB")

OUT <- c()
for(mod in seq_along(models))
{
    filename <- paste0("outCOR_",models[mod],".RData")    
    if(file.exists(filename)){
        load(filename,verbose=T)
        OUT <- cbind(OUT,outCOR)
    }    
}

round(rbind(Mean=apply(OUT,2,mean),SD=apply(OUT,2,sd)),4)
boxplot(outCOR,ylab="Accuracy",xlab="Model")
```

|       |GBLUP  | BRR  | LASSO | Bayes B |
|-------|-------|--------|------|-------|
|Mean | 0.475  | 0.476  | 0.474 | 0.465 |
|SD  | 0.049 | 0.050 | 0.049 | 0.048 |

Boxplot of distribution of the accuracies by model

![](https://github.com/MarcooLopez/Genomic-Selection/blob/master/boxplot1.png)

#
* **[back](https://github.com/MarcooLopez/Genomic-Selection-Demo/blob/master/README.md)**
