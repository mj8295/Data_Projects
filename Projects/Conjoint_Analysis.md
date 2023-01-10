# Toy Horse Case

## Objective
The objective of this project is to perform a conjoint analysis to understand how consumers value different features of a product, in this case a toy horse. The project will involve analyzing consumer survey data using conjoint analysis techniques and interpreting the results to understand how consumers value different features. In particular, the analysis will consider the potential impact of competitors' responses, as well as the costs associated with offering different features, including both variable costs and fixed costs.

## Tools Used
- R
- R Markdown

## Concepts Used
[Conjoint Analysis](https://github.com/mj8295/Data_Projects/blob/74cdfa2d9156c8995cc85ca41cb64ce991034a9f/Concepts/Conjoint_Analysis.md)

## Workflow
### Part 0: Initial Setup
### Load the Data and the Needed Packages
```{r}
setwd("C:/Users/Max/Documents/MSBA/MSBA Fall B/GBA 424/Case 5")
load(file = "GBA424 Fall 2020 - Toy Horse Case Data.Rdata")
library(cluster)
library(fpc)
library(factoextra)
library(gridExtra)
```
### Inspect the Data
```{r dataInspection}
head(conjointData)
```
### Part 1: Set up the Data for Analysis
#### Create a matrix of all the possible product combinations
```{r}
# Creates a matrix of all possible product combinations
desmat = matrix(unlist(conjointData[,4:7]), ncol = 4, byrow = F)
```
#### Create a list of product attributes
```{r}
attributesMatrix = c("Low Price","Tall Size","Rocking","Glamour")
```
#### Assign the product attributes defined above to the matrix desmat defined earlier
```{r}
colnames(desmat) = attributesMatrix
```

#### Map the respondentData to the conjointData based on the ID field
Both of these files are found on the desktop and were provided to the analyst
```{r}
conjointDataDetail = merge(conjointData, respondentData, 'ID')
```

#### Create an array of the various respondants attributes
```{r}
# Creates an array for the various attributes
ageD = conjointDataDetail[, 8]
genderD = conjointDataDetail[, 9]
ratings = conjointDataDetail[, 3]
ID = conjointDataDetail[, 1]
```

### Part 2: Part-Utility Analysis
This section includes an analysis that calculates the perceived benefits of each attribute, known as part-utilities, at the individual level. These values will be used in our post-hoc segmentation process and to predict the missing ratings in incomplete profiles, resulting in a complete set of profile ratings.

#### Calculate sample size
This will be used to help fill out the partworths matrix that will be defined later
```{r}
sampsize = nrow(respondentData)
```
#### Add column for constant
```{r}
desmatf = cbind(rep(1,nrow(desmat)),desmat); 
```
#### Create an empty matrix
This matrix will be filled with partworths that will be found in the next code chunk
```{r}
partworths = matrix(nrow=sampsize,ncol=ncol(desmatf))
```

#### Fill out the partworth matrix
This will be done running a regression and applying each profile to it
```{r}
for(i in 1:sampsize){
  partworths[i,]=lm(ratings~desmat,subset=ID==i)$coef
}
```
#### Add column names to the partworths matrix
```{r}
colnames(partworths) = c("Intercept",attributesMatrix)
```
#### Predict the missing cells
This will prepare the data for the market simulation
```{r}
partworths.full = matrix(rep(partworths,each=16),ncol=5)
```
#### Create an array of the ratings
```{r}
pratings = rowSums(desmatf*partworths.full)
```
#### Combine mthe actual and predicted ratings
```{r}
finalratings = matrix(round(ifelse(is.na(ratings),pratings,ratings)),ncol = 16,byrow = T) 
```

### Part 3: Post-Hoc Segmentation
In this section, cluster analysis is used on the part-utilities, including the constant defined in part 2, to identify the optimal post-hoc segmentation scheme. Post-hoc segmentation involves analyzing data to identify segments after it has been collected, as opposed to a-priori segmentation where segments are identified before the analysis process. This analysis will generate several visualizations to help us determine the unique profiles and optimal number of clusters that make up the segments.

#### Create the clustTest function
This function will take the following arguments:
- toClust, the data to do kmeans cluster analysis
- maxClusts=15, the max number of clusters to consider
- seed, random number used to initialize the clusters
- iter.max, the max iterations for clustering algorithms to use
- nstart, the number of starting points to consider
The function is designed to yield a list of weighted sum of squares and the pamk output including optimal number of clusters to create visualizations need to print tmp
```{r}
clustTest = function(toClust,print=TRUE,scale=TRUE,maxClusts=15,seed=12345,nstart=20,iter.max=100){
  if(scale){ toClust = scale(toClust);}
  set.seed(seed);
  wss <- (nrow(toClust)-1)*sum(apply(toClust,2,var))
  for (i in 2:maxClusts) wss[i] <- sum(kmeans(toClust,centers=i,nstart=nstart,iter.max=iter.max)$withinss)
  ##gpw essentially does the following plot using wss above. 
  #plot(1:maxClusts, wss, type="b", xlab="Number of Clusters",ylab="Within groups sum of squares")
  gpw = fviz_nbclust(toClust,kmeans,method="wss",iter.max=iter.max,nstart=nstart,k.max=maxClusts) 
  #alternative way to get wss elbow chart.
    pm1 = pamk(toClust,scaling=TRUE)
  ## pm1$nc indicates the optimal number of clusters based on 
  ## lowest average silhoutte score (a measure of quality of clustering)
  #alternative way that presents it visually as well.
  gps = fviz_nbclust(toClust,kmeans,method="silhouette",iter.max=iter.max,nstart=nstart,k.max=maxClusts) 
  if(print){
    grid.arrange(gpw,gps, nrow = 1)
  }
  list(wss=wss,pm1=pm1$nc,gpw=gpw,gps=gps)
}
```
## Outcomes & Conclusions Made
