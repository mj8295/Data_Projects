Conjoint Analysis
================
Maximilian Johnson
20 June 2022

## Objective

The objective of this project is to understand how consumers value different features of a product.  The project will involve analyzing consumer survey data using conjoint analysis techniques and interpreting the results tounderstand how consumers value different features. In particular, the analysis will consider the potential impact of competitors’ responses as well as the costs associated with offering different features, including both variable costs and fixed costs.

## Project Background
The company is a manufacturer of toy horses, competing with a single rival. Currently, the company offers two 18-inch rocking horses in the racing and glamor styles, both priced at $139.99. Meanwhile, its competitor sells a 26-inch rocking horse in the racing style, also priced at $139.99. The company is currently earning a profit of $96,000, while the competitor is earning $141,000. The company is looking to change its product offering to maximize profits.

To achieve this goal, a survey was conducted among consumers to rank 16 different profiles of toy horses. These profiles varied in terms of price (ranging from $139.99 to $119.99), size (18 inches and 26 inches), motion (rocking and bouncing) and styles (racing and rocking).  Using these data, an analysis will be performed to provide the company with a data driven recommendation.


## Tools Used
- R
- R Markdown

## Concepts Used

[Conjoint Analysis](https://github.com/mj8295/Data_Projects/blob/74cdfa2d9156c8995cc85ca41cb64ce991034a9f/Concepts/Conjoint_Analysis.md)

[Cluster Analysis](https://github.com/mj8295/Data_Projects/blob/467a88c3704484924407fcd1f28c71f71b500267/Concepts/ClusterAnalysis.md)

[A Priori Segmentation](https://github.com/mj8295/Data_Projects/blob/467a88c3704484924407fcd1f28c71f71b500267/Concepts/APrioriSegmentation.md)

[Post-Hoc Segmentation](https://github.com/mj8295/Data_Projects/blob/467a88c3704484924407fcd1f28c71f71b500267/Concepts/PostHocSegmentation.md)

[Disaggergate Choice Model]()

[First Choice Rule]()

## Workflow

### Part 0: Initial Setup

#### Part 0.1: Load the Data and the Needed Packages

``` r
setwd("C:/Users/Max/Documents/MSBA/MSBA Fall B/GBA 424/Case 5")
load(file = "GBA424 Fall 2020 - Toy Horse Case Data.Rdata")
library(cluster)
library(fpc)
library(factoextra)
library(gridExtra)
```


#### Part 0.2: Inspect the Data

``` r
head(conjointData)
```

    ##   ID profile ratings price size motion style
    ## 1  1       1       8     0    0      0     0
    ## 2  1       2       8     1    0      0     0
    ## 3  1       3      NA     0    1      0     0
    ## 4  1       4      14     1    1      0     0
    ## 5  1       5       8     0    0      1     0
    ## 6  1       6      NA     1    0      1     0

### Part 1: Set up the Data for Analysis

#### Part 1.1: Create a matrix of all the possible product combinations

``` r
# Creates a matrix of all possible product combinations
desmat = matrix(unlist(conjointData[,4:7]), ncol = 4, byrow = F)
```

#### Part 1.2: Create a list of product attributes

``` r
attributesMatrix = c("Low Price","Tall Size","Rocking","Glamour")
```

#### Part 1.3: Assign the product attributes defined above to the matrix desmat defined earlier

``` r
colnames(desmat) = attributesMatrix
```

#### Part 1.4: Map the respondentData to the conjointData based on the ID field

Both of these files are found on the desktop and were provided to the
analyst

``` r
conjointDataDetail = merge(conjointData, respondentData, 'ID')
```

#### Part 1.5: Create an array of the various respondants attributes

``` r
# Creates an array for the various attributes
ageD = conjointDataDetail[, 8]
genderD = conjointDataDetail[, 9]
ratings = conjointDataDetail[, 3]
ID = conjointDataDetail[, 1]
```

### Part 2: Part-Utility Analysis

This section includes an analysis that calculates the perceived benefits of each attribute, known as part-utilities, at the individual level. These values will be used in our post-hoc segmentation process and to predict the missing ratings in incomplete profiles, resulting in a complete set of profile ratings.

#### Part 2.1: Calculate sample size

This will be used to help fill out the partworths matrix that will be defined later

``` r
sampsize = nrow(respondentData)
```

#### Part 2.2: Add column for constant

``` r
desmatf = cbind(rep(1,nrow(desmat)),desmat); 
```

#### Part 2.3: Create an empty matrix

This matrix will be filled with partworths that will be found in the
next code chunk

``` r
partworths = matrix(nrow=sampsize,ncol=ncol(desmatf))
```

#### Part 2.4: Fill out the partworth matrix

This will be done running a regression and applying each profile to it

``` r
for(i in 1:sampsize){
  partworths[i,]=lm(ratings~desmat,subset=ID==i)$coef
}
```

#### Part 2.5: Add column names to the partworths matrix

``` r
colnames(partworths) = c("Intercept",attributesMatrix)
```

#### Part 2.6: Predict the missing cells

This will prepare the data for the market simulation

``` r
partworths.full = matrix(rep(partworths,each=16),ncol=5)
```

#### Part 2.7: Create an array of the ratings

``` r
pratings = rowSums(desmatf*partworths.full)
```

#### Part 2.8: Combine mthe actual and predicted ratings

``` r
finalratings = matrix(round(ifelse(is.na(ratings),pratings,ratings)),ncol = 16,byrow = T) 
```

### Part 3: Post-Hoc Segmentation

In this section, cluster analysis is used on the part-utilities,
including the constant defined in part 2, to identify the optimal
post-hoc segmentation scheme. Post-hoc segmentation involves analyzing
data to identify segments after it has been collected, as opposed to
a-priori segmentation where segments are identified before the analysis
process. This analysis will generate several visualizations to help us
determine the unique profiles and optimal number of clusters that make
up the segments.

#### Part 3.1: Create the clustTest function

This function will take the following arguments: 
- toClust - the data to do kmeans cluster analysis 
- maxClusts - the max number of clusters to consider 
- seed - random number used to initialize the clusters
- iter.max - the max iterations for clustering algorithms to use 
- nstart - the number of starting points to consider

The function is designed to yield a list of weighted sum of squares and
the pamk output including optimal number of clusters to create
visualizations need to print tmp

``` r
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
#### Part 3.2: Create the runClust function

runClust will run a set of clusters as kmeans and plot them


Arguments:
- toClust, data.frame with data to cluster
- nClusts, vector of number of clusters, each run as separate kmeans 
- ... some additional arguments to be passed to clusters


Return:
list of 
- kms - kmeans cluster output with length of nClusts
- ps - list of plots of the clusters against first 2 principle components
``` r
runClusts = function(toClust,nClusts,print=TRUE,maxClusts=15,seed=12345,nstart=20,iter.max=100){
  if(length(nClusts)>4){
    warning("Using only first 4 elements of nClusts.")
  }
  kms=list(); ps=list();
  for(i in 1:length(nClusts)){
    kms[[i]] = kmeans(toClust,nClusts[i],iter.max = iter.max, nstart=nstart)
    ps[[i]] = fviz_cluster(kms[[i]], geom = "point", data = toClust) + ggtitle(paste("k =",nClusts[i]))
   
  }
  library(gridExtra)
  if(print){
    tmp = marrangeGrob(ps, nrow = 2,ncol=2)
    print(tmp)
  }
  list(kms=kms,ps=ps)
}
```
#### Part 3.3: Create the plotClust function

The plotClust function will generate an in debth visual of the clusters.

Plots a kmeans cluster as three plot report
-  pie chart with membership percentages
-  plot that indicates cluster definitions against principle components
-  barplot of the cluster means, which by default standardizes the cluster means

``` r
plotClust = function(km,toClust,
                     discPlot=FALSE,standardize=TRUE,margins = c(7,4,4,2)){
  nc = length(km$size)
  percsize = paste(1:nc," = ",format(km$size/sum(km$size)*100,digits=2),"%",sep="")
  pie(km$size,labels=percsize,col=1:nc)
  
  gg = fviz_cluster(km, geom = "point", data = toClust) + ggtitle(paste("k =",nc))
  print(gg)
  
  if(discPlot){
    plotcluster(toClust, km$cluster,col=km$cluster); #plot against discriminant functions ()
  }
  if(!standardize){
    rng = range(km$centers)
    dist = rng[2]-rng[1]
    locs = km$centers+.05*dist*ifelse(km$centers>0,1,-1)
    bm = barplot(km$centers,beside=TRUE,col=1:nc,main="Cluster Means",ylim=rng+dist*c(-.1,.1))
    text(bm,locs,formatC(km$centers,format="f",digits=1))
  } else {
    kmc = (km$centers-rep(colMeans(toClust),each=nc))/rep(apply(toClust,2,sd),each=nc)
    rng = range(kmc)
    dist = rng[2]-rng[1]
    locs = kmc+.05*dist*ifelse(kmc>0,1,-1)
    par(mar=margins)
    bm = barplot(kmc,col=1:nc,beside=TRUE,las=2,main="Cluster Means",ylim=rng+dist*c(-.1,.1))
    text(bm,locs,formatC(kmc,format="f",digits=1))
  }
}
```
### Part 4: Visual Analysis

Apply the functions created in step 3 to analyze the output visualizations and determine the optimal cluster size.

#### Part 4.1: Elbow Chart Analysis
``` r
# set random number seed before doing cluster analysis so results are constant
set.seed(123) 

toClust = partworths
tmp = clustTest(toClust)
```

![](ConjointAnalysis_files/figure-gfm/postHocSegmentation%20Application-1.png)<!-- -->

These charts indicate that three clusters is the optimal amount of clusters. Both visuals have a kink in the line when the x-axis is 3, indicating that three is the optimal amount of clusters.

#### Part 4.2 runClust Analysis

Based on the results seen in Part 4.1, use the runClust function to test 2, 3, and 4 clusters.

``` r
clusts = runClusts(toClust,2:4)
```

![](ConjointAnalysis_files/figure-gfm/postHocSegmentation%20Application2-1.png)<!-- -->

This chart confirms that the data best conforms to being separated into
3 distinct clusters


#### Part 4.3: plotClust Analysis
With the information yielded in parts 4.1 & 4.2, divide the data into 3 clusters and run it through the plotClust function

``` r
plotClust(clusts$kms[[2]],toClust)
```

![](ConjointAnalysis_files/figure-gfm/postHocSegmentation%20Application3-1.png)<!-- -->![](ConjointAnalysis_files/figure-gfm/postHocSegmentation%20Application3-2.png)<!-- -->![](ConjointAnalysis_files/figure-gfm/postHocSegmentation%20Application3-3.png)<!-- -->

These results further the idea that 3 segments is the best.  Three segments best represents the idea of homogeneity within and heterogeneity between better than 2 segments or 4 segments.

The best products for each segment are as follows:

- Segment 1: Profile 4 - 119.99, 26 inch, bouncing, racing
- Segment 2: Profile 14 - 119.99, 18 inch, rocking, glamour
- Segment 3: Profile 16 - 119.99, 26 inch, rocking, glamour

### Step 5: A Priori Segmentation
In this section an a priori segmentation analysis will be performed using the variables gender and age in order to profile the attribute preferences based on these variables.


The output charts can be read as follows:

- Call - the function that generated the report
- Residuals - some summary statistics on the data
- Coefficients - the amount each variable affects the output (rating)

#### Part 5.1: Perform A Priori Segmentation on Age
``` r
summary(lm(ratings~desmat*ageD))
```

    ## 
    ## Call:
    ## lm(formula = ratings ~ desmat * ageD)
    ## 
    ## Residuals:
    ##     Min      1Q  Median      3Q     Max 
    ## -8.3750 -2.6250 -0.1463  2.5384  8.3100 
    ## 
    ## Coefficients:
    ##                      Estimate Std. Error t value Pr(>|t|)    
    ## (Intercept)           6.96149    0.22164  31.408  < 2e-16 ***
    ## desmatLow Price       2.95581    0.20769  14.232  < 2e-16 ***
    ## desmatTall Size       0.70770    0.19885   3.559  0.00038 ***
    ## desmatRocking         0.52125    0.19885   2.621  0.00881 ** 
    ## desmatGlamour         0.18455    0.19885   0.928  0.35344    
    ## ageD                 -0.33381    0.31190  -1.070  0.28461    
    ## desmatLow Price:ageD  0.33008    0.29226   1.129  0.25884    
    ## desmatTall Size:ageD  0.84036    0.27982   3.003  0.00270 ** 
    ## desmatRocking:ageD   -0.58747    0.27982  -2.099  0.03588 *  
    ## desmatGlamour:ageD   -0.05605    0.27982  -0.200  0.84126    
    ## ---
    ## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
    ## 
    ## Residual standard error: 3.375 on 2390 degrees of freedom
    ##   (800 observations deleted due to missingness)
    ## Multiple R-squared:  0.2086, Adjusted R-squared:  0.2056 
    ## F-statistic: 70.01 on 9 and 2390 DF,  p-value: < 2.2e-16

#### Step 5.2: Perform A Priori Segmentation on Gender
``` r
summary(lm(ratings~desmat*genderD))
```

    ## 
    ## Call:
    ## lm(formula = ratings ~ desmat * genderD)
    ## 
    ## Residuals:
    ##     Min      1Q  Median      3Q     Max 
    ## -8.8897 -2.2695 -0.0421  2.5534  8.5534 
    ## 
    ## Coefficients:
    ##                         Estimate Std. Error t value Pr(>|t|)    
    ## (Intercept)               6.2695     0.2195  28.561  < 2e-16 ***
    ## desmatLow Price           3.4973     0.2057  17.003  < 2e-16 ***
    ## desmatTall Size           0.7468     0.1969   3.792 0.000153 ***
    ## desmatRocking            -0.1526     0.1969  -0.775 0.438405    
    ## desmatGlamour            -0.4171     0.1969  -2.118 0.034271 *  
    ## genderD                   0.9693     0.2987   3.245 0.001191 ** 
    ## desmatLow Price:genderD  -0.6940     0.2799  -2.480 0.013224 *  
    ## desmatTall Size:genderD   0.7134     0.2680   2.662 0.007817 ** 
    ## desmatRocking:genderD     0.6985     0.2680   2.607 0.009202 ** 
    ## desmatGlamour:genderD     1.0618     0.2680   3.962 7.65e-05 ***
    ## ---
    ## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
    ## 
    ## Residual standard error: 3.222 on 2390 degrees of freedom
    ##   (800 observations deleted due to missingness)
    ## Multiple R-squared:  0.2787, Adjusted R-squared:  0.276 
    ## F-statistic: 102.6 on 9 and 2390 DF,  p-value: < 2.2e-16

#### Step 5.3: Perform A Priori Segmentation on older kids (aged 3-4)
``` r
summary(lm(ratings~desmat,subset=ageD==1))
```

    ## 
    ## Call:
    ## lm(formula = ratings ~ desmat, subset = ageD == 1)
    ## 
    ## Residuals:
    ##     Min      1Q  Median      3Q     Max 
    ## -7.2380 -2.7562 -0.4616  3.3723  8.3100 
    ## 
    ## Coefficients:
    ##                 Estimate Std. Error t value Pr(>|t|)    
    ## (Intercept)      6.62768    0.23235  28.524  < 2e-16 ***
    ## desmatLow Price  3.28589    0.21772  15.092  < 2e-16 ***
    ## desmatTall Size  1.54806    0.20845   7.426  2.1e-13 ***
    ## desmatRocking   -0.06621    0.20845  -0.318    0.751    
    ## desmatGlamour    0.12851    0.20845   0.616    0.538    
    ## ---
    ## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
    ## 
    ## Residual standard error: 3.573 on 1207 degrees of freedom
    ##   (404 observations deleted due to missingness)
    ## Multiple R-squared:  0.2224, Adjusted R-squared:  0.2198 
    ## F-statistic: 86.28 on 4 and 1207 DF,  p-value: < 2.2e-16

#### Step 5.4: Perform A Priori Segmentation on younger kids (aged 2 or less)
``` r
summary(lm(ratings~desmat,subset=ageD==0)) 
```

    ## 
    ## Call:
    ## lm(formula = ratings ~ desmat, subset = ageD == 0)
    ## 
    ## Residuals:
    ##     Min      1Q  Median      3Q     Max 
    ## -8.3750 -2.1463  0.0385  2.3327  7.1463 
    ## 
    ## Coefficients:
    ##                 Estimate Std. Error t value Pr(>|t|)    
    ## (Intercept)       6.9615     0.2075  33.551  < 2e-16 ***
    ## desmatLow Price   2.9558     0.1944  15.203  < 2e-16 ***
    ## desmatTall Size   0.7077     0.1862   3.802 0.000151 ***
    ## desmatRocking     0.5213     0.1862   2.800 0.005191 ** 
    ## desmatGlamour     0.1846     0.1862   0.991 0.321685    
    ## ---
    ## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
    ## 
    ## Residual standard error: 3.159 on 1183 degrees of freedom
    ##   (396 observations deleted due to missingness)
    ## Multiple R-squared:  0.1896, Adjusted R-squared:  0.1869 
    ## F-statistic: 69.21 on 4 and 1183 DF,  p-value: < 2.2e-16

#### Step 5.5: Perform A Priori Segmentation on Girls
``` r
summary(lm(ratings~desmat,subset=genderD==1))
```

    ## 
    ## Call:
    ## lm(formula = ratings ~ desmat, subset = genderD == 1)
    ## 
    ## Residuals:
    ##    Min     1Q Median     3Q    Max 
    ## -8.890 -2.884  0.755  2.767  6.571 
    ## 
    ## Coefficients:
    ##                 Estimate Std. Error t value Pr(>|t|)    
    ## (Intercept)       7.2388     0.2148  33.693  < 2e-16 ***
    ## desmatLow Price   2.8032     0.2013  13.924  < 2e-16 ***
    ## desmatTall Size   1.4603     0.1927   7.576 6.76e-14 ***
    ## desmatRocking     0.5459     0.1927   2.832 0.004694 ** 
    ## desmatGlamour     0.6447     0.1927   3.345 0.000847 ***
    ## ---
    ## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
    ## 
    ## Residual standard error: 3.416 on 1291 degrees of freedom
    ##   (432 observations deleted due to missingness)
    ## Multiple R-squared:  0.1876, Adjusted R-squared:  0.1851 
    ## F-statistic: 74.51 on 4 and 1291 DF,  p-value: < 2.2e-16

#### Step 5.6: Perform A Priori Segmentation on Boys
``` r
summary(lm(ratings~desmat,subset=genderD==0))
```

    ## 
    ## Call:
    ## lm(formula = ratings ~ desmat, subset = genderD == 0)
    ## 
    ## Residuals:
    ##     Min      1Q  Median      3Q     Max 
    ## -6.4466 -2.1168 -0.5136  1.7305  8.5534 
    ## 
    ## Coefficients:
    ##                 Estimate Std. Error t value Pr(>|t|)    
    ## (Intercept)       6.2695     0.2028  30.912  < 2e-16 ***
    ## desmatLow Price   3.4973     0.1900  18.402  < 2e-16 ***
    ## desmatTall Size   0.7468     0.1820   4.104 4.35e-05 ***
    ## desmatRocking    -0.1526     0.1820  -0.839   0.4018    
    ## desmatGlamour    -0.4171     0.1820  -2.292   0.0221 *  
    ## ---
    ## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
    ## 
    ## Residual standard error: 2.977 on 1099 degrees of freedom
    ##   (368 observations deleted due to missingness)
    ## Multiple R-squared:  0.2867, Adjusted R-squared:  0.2841 
    ## F-statistic: 110.4 on 4 and 1099 DF,  p-value: < 2.2e-16
This analysis suggests that gender plays a significant role in determining customer preferences for products. The data reveals that girls tend to prefer glamour horses while boys prefer racing horses. Additionally, older children tend to prefer the 26-inch horse over younger children. As expected, all consumers prefer lower prices. These findings are logical and will be taken into consideration when defining different product mixes later on in the analysis.

### Part 6: Disagergate Choice Model

This section will conduct a disaggregate choice analysis using a first choice rule to predict market shares for various scenarios. The objective is to forecast market shares for a relevant set of scenarios to the decision at hand. With these market share predictions and cost information, profitability for each product in the product line, as well as overall profitability for the firm and competition, will be calculated.

#### Part 6.1: Define Scenarios
The assumption that the competitor will not increase their prices will be made. It is crucial to evaluate a broad range of product mixes to identify the optimal option. However, it is not practical to evaluate every possible product combination. Therefore, the data collected in the A Priori Segmentation Analysis will be used to selectively define scenarios. It's worth noting that the first scenario defined (scens[[1]]) represents the current market view.
``` r
scens = list()
scens[[1]]=c(5,13,7)
scens[[2]]=c(15,7)        
scens[[3]]=c(16,7)        
scens[[4]]=c(8,7)         
scens[[5]]=c(4,7)         
scens[[6]]=c(3,7)     
scens[[7]]=c(14,7)  
scens[[8]]=c(13,7)   
scens[[9]]=c(15,16,7) 
scens[[10]]=c(15,8,7) 
scens[[11]]=c(15,4,7) 
scens[[12]]=c(15,3,7) 
scens[[13]]=c(15,14,7) 
scens[[14]]=c(15,13,7) 
scens[[15]]=c(16,8,7) 
scens[[16]]=c(16,4,7) 
scens[[17]]=c(16,3,7) 
scens[[18]]=c(16,14,7)
scens[[19]]=c(16,13,7) 
scens[[20]]=c(8,4,7)
scens[[21]]=c(8,3,7) 
scens[[20]]=c(8,14,7)
scens[[21]]=c(8,13,7) 
scens[[22]]=c(4,14,7)
scens[[23]]=c(4,13,7) 
scens[[24]]=c(3,14,7) 
scens[[25]]=c(3,13,7) 
scens[[26]]=c(15,16,8,7) 
scens[[27]]=c(15,16,4,7) 
scens[[28]]=c(15,16,3,7) 
scens[[29]]=c(15,16,14,7)
scens[[30]]=c(15,16,13,7) 
scens[[31]]=c(16,8,4,7) 
scens[[32]]=c(16,8,3,7) 
scens[[33]]=c(16,8,14,7) 
scens[[34]]=c(16,8,13,7) 
scens[[35]]=c(16,4,3,7)
scens[[36]]=c(16,4,14,7) 
scens[[37]]=c(16,3,14,7) 
scens[[38]]=c(16,3,13,7) 
scens[[39]]=c(16,14,13,7) 
scens[[40]]=c(16,8,13,7) 
scens[[41]]=c(15,8,4,7)
scens[[42]]=c(15,8,3,7) 
scens[[43]]=c(15,8,14,7) 
scens[[44]]=c(15,8,13,7) 
scens[[45]]=c(15,8,13,7) 
scens[[46]]=c(15,4,3,7)
scens[[47]]=c(15,4,14,7) 
scens[[48]]=c(15,4,13,7) 
scens[[49]]=c(15,3,14,7) 
scens[[50]]=c(15,3,13,7) 
scens[[51]]=c(15,14,13,7) 
scens[[52]]=c(8,4,3,7)
scens[[53]]=c(8,4,14,7) 
scens[[54]]=c(8,4,13,7) 
scens[[55]]=c(8,3,14,7) 
scens[[56]]=c(8,3,13,7) 
scens[[57]]=c(4,3,14,7) 
scens[[58]]=c(4,3,13,7) 
scens[[59]]=c(4,14,13,7) 
scens[[60]]=c(4,3,7)
scens[[61]]=c(14,13,7)
```
#### Part 6.2: Define Functions to Perform a Market Simulation 

##### Part 6.2.1: Define simSecnarios Function
This function will be used to simulate each scenario as defined in Part 6.1


simScenatios Arguments:
- scen - a list of scenarios, which are vectors that index into data
- data - a data.frame containing the rank order information
- ... - an argument that accepts anything else and just passes it along


Return:
- data.frame containing shares with columns corresponding to the columns in the data

This function works by looping over each scenario and:
1. Create a subsetted matrix.
2. For each product, make a decision by assigning a value of 1 or 0, with 1 indicating that the product was chosen by the consumer.
3. Assign the best option to the "choice" list.

After each scenario has been evaluated, the list "choice" will be returned.  Note that in the case of a tie the 1 will be split evenly among the tied products

``` r
simScenarios = function(scen,data,...){ 
  choice = list()
  for (k in 1: length(scen)){
    # constructs a subsetted matrix of options
    inmkt = data[, scen[[k]]]
    for (i in 1:nrow(inmkt)){
      bestOpts = max(inmkt[i,]) 
    for (j in 1:ncol(inmkt)){
      if(inmkt[i,j]==bestOpts){
       inmkt[i,j] = 1
      } else{
      inmkt[i,j] = 0  
      }
  }
      inmkt[i,] = inmkt[i,]/sum(inmkt[i,])}
    choice[[k]]=inmkt}
  return(choice)}
```
##### Part 6.2.2: Define shs Function
This function will be used in the simFCShares function to calculate market share
``` r
shs = function(decs){
  colSums(decs)/sum(decs)}
```
##### Part 6.2.3: Define simFCShares Function
Using the simScenarios function and the shs function, simFCShares returns the market share of each product in a given scenario.  This is done by applying the output of the simScenarios function to the shs function.
``` r
simFCShares = function(scen, data){
  #fill decisions to be 0 or 1 for all products
  decs = simScenarios(scen, data)
  #applies the decs input to the shs function and saves the output to share
  share = lapply(decs,shs)
  return(share)
}
```

–

##### Part 6.2.4: Define simProfit Function
The simProfit function utilizes the output from the simFCShares function to calculate the expected profit for the company and its competitor. The function returns two lists: the first list contains the profits of the company and the second list contains the profits of the competitor.

``` r
simProfit = function(inputmat,scens,mktshr){
   cm = list()
   result = c()
   competitor = c()
   for (i in 1:length(scens)){
     margin = profilesData[scens[[i]],'margin']
     cm[[i]] = unlist(mktshr[[i]])*margin* 4000
     result[i] = sum(cm[[i]][-length(cm[[i]])]) - 20000*(length(cm[[i]])-1) -7000*sum( !scens[[i]] %in% c(5,7,13))
     competitor[i] = cm[[i]][length(cm[[i]])] - 20000
       }
   return(list(result,competitor))
   
 }
```
#### Part 6.3: Apply the Functions
##### Part 6.3.1: Calculate Contribution Margin
``` r
for (i in (1:nrow(profilesData))){
  if (profilesData[i, 'size'] == 0 & profilesData[i,'motion']==1){
    vcosts = 33
  }else if(profilesData[i, 'size'] == 1 & profilesData[i,'motion']==1){
    vcosts = 41
  }else if(profilesData[i, 'size'] == 0 & profilesData[i,'motion']==0){
    vcosts = 21
  }else if(profilesData[i, 'size'] == 1 & profilesData[i,'motion']==0){
    vcosts = 29}
  
  if(profilesData[i, 'price'] ==0){
    prices = 111.99
  } else{
    prices =95.99
  }
  profilesData$margin[i] = prices - vcosts
}
profilesData
```

    ##    profile price size motion style priceLabel sizeLabel motionLabel styleLabel
    ## 1        1     0    0      0     0     139.99 18 inches    Bouncing     Racing
    ## 2        2     1    0      0     0     119.99 18 inches    Bouncing     Racing
    ## 3        3     0    1      0     0     139.99 26 inches    Bouncing     Racing
    ## 4        4     1    1      0     0     119.99 26 inches    Bouncing     Racing
    ## 5        5     0    0      1     0     139.99 18 inches     Rocking     Racing
    ## 6        6     1    0      1     0     119.99 18 inches     Rocking     Racing
    ## 7        7     0    1      1     0     139.99 26 inches     Rocking     Racing
    ## 8        8     1    1      1     0     119.99 26 inches     Rocking     Racing
    ## 9        9     0    0      0     1     139.99 18 inches    Bouncing    Glamour
    ## 10      10     1    0      0     1     119.99 18 inches    Bouncing    Glamour
    ## 11      11     0    1      0     1     139.99 26 inches    Bouncing    Glamour
    ## 12      12     1    1      0     1     119.99 26 inches    Bouncing    Glamour
    ## 13      13     0    0      1     1     139.99 18 inches     Rocking    Glamour
    ## 14      14     1    0      1     1     119.99 18 inches     Rocking    Glamour
    ## 15      15     0    1      1     1     139.99 26 inches     Rocking    Glamour
    ## 16      16     1    1      1     1     119.99 26 inches     Rocking    Glamour
    ##    margin
    ## 1   90.99
    ## 2   74.99
    ## 3   82.99
    ## 4   66.99
    ## 5   78.99
    ## 6   62.99
    ## 7   70.99
    ## 8   54.99
    ## 9   90.99
    ## 10  74.99
    ## 11  82.99
    ## 12  66.99
    ## 13  78.99
    ## 14  62.99
    ## 15  70.99
    ## 16  54.99

##### Part 6.3.2: Obtain the Market Share
Use simFCShares and assign the output to a variable which can be used as an argument in the simProfit function.
``` r
mktshr = simFCShares(scens,finalratings)
```
##### Part 6.3.3: Calculate Profit
The consumer survey results, scenarios, and market share will be used as arguments here.
```r
simProfit(finalratings,scens,mktshr)
```

    ## [[1]]
    ##  [1]  96389.4 118529.5 191310.3 184711.5 173300.1 112423.2 197244.4  81107.2
    ##  [9] 164630.3 173690.8 193883.9 199413.5 198457.9 135165.2 165226.8 184420.2
    ## [17] 170560.3 174606.7 176763.5 171474.1 171984.5 188992.7 191875.9 186707.3
    ## [25] 181031.9 138546.8 157740.2 143880.3 147686.7 149950.2 155186.8 138226.8
    ## [33] 145653.3 148685.0 157420.2 165520.0 153585.0 156013.5 154766.7 148685.0
    ## [41] 164117.4 146970.8 155370.4 160013.9 160013.9 166883.9 181200.7 196469.9
    ## [49] 186970.9 214152.8 178937.9 152544.5 162414.0 166194.2 144474.1 145264.5
    ## [57] 161992.7 164875.9 170292.6 146300.1 179944.2
    ## 
    ## [[2]]
    ##  [1] 141383.9333 118430.5000 -17870.3000  -9351.5000  51699.9000 144696.8000
    ##  [7]  11235.6000 173092.8000 -17870.3000 -14557.4333   7449.4667  28746.4667
    ## [13] -11244.5667  88141.4333 -19053.4667 -18580.2000 -18106.9333 -19526.7333
    ## [19] -18816.8333 -14794.0667 -11717.8333   -832.7000  21884.1000   8159.3667
    ## [25]  64714.7333 -19053.4667 -18580.2000 -18106.9333 -19526.7333 -18816.8333
    ## [31] -19053.4667 -19053.4667 -20000.0000 -19645.0500 -18580.2000 -20000.0000
    ## [37] -19645.0500 -19053.4667 -19526.7333 -19645.0500 -14557.4333 -14557.4333
    ## [43] -17397.0333 -16213.8667 -16213.8667   7449.4667 -15267.3333  -8996.5500
    ## [49] -13610.9000    113.8333 -11244.5667 -11717.8333 -15267.3333 -13847.5333
    ## [55] -14794.0667 -11717.8333   -832.7000  21884.1000  -1305.9667  51699.9000
    ## [61]   9815.8000


Reorganizing the data can be useful here but not necessary.  Here the scenario that has the maximum profit will be identified.
``` r
profit <- as.data.frame(simProfit(finalratings,scens,mktshr))
colnames(profit) <- c("profits","competiors")
which.max(profit$profits)
```

    ## [1] 50


Shifting to selling horse profiles 3 and 15 appears to be the best option based on this analysis. Although this strategy does not maximize profit, the potential negative external effects of reducing the competition's profit to nearly zero outweigh the extra profit gained by introducing three product lines. Additionally, introducing three different product lines may lead to unintended consequences, such as limited availability of the horses at retailers. The competitor's profit would drop from $141,000 to $28,000 which may prompt a change in their strategy. This analysis can be re-evaluated to account for the competitor's actions.
