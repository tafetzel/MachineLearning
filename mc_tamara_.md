# Predicting quality of exercise
Tamara Fetzel  
19. Jan. 2016  


# Executive summary

Exercising provides substantial benefits for health and many people regularly work out and take records using new devices like the Jawbone Up, Fitbit or the Nike FuelBand. Despite measuring the quantity of a specific exercise the recorded data could also be used to analyse the quality of the exercise. This is important because how well an exercise is done essentially influences health benefits and doing certain exercises in a wrong manner can cause injury. This analysis uses data on weight lifting exercises performed correctly and incorrectlty in 5 different ways to predict correct or incorrect execution. The data are trained using a random forest algorithm implementing boosting (25 bootstrap replications), cross-validation (10fold) and Principal Components (PCA) pre-processing. The out of sample error based on the cross-validation technique is 1.45% and the algorithm is able to predict all test-samples correctly. This analysis aims to help athlets to perform exercise in a correct 
manner to maximise health benefits and minimize injuries

# Methods 

The data used to analyse excercise quality are taken from Velloso et al. (2013) weight lifting data-set. The authors collect data on weight excercises in 6 different manner, one where the excercise is performed correctly and 5 others where common mistakes are simulated. The whole process was supervised by a professional trainer.


```r
setwd("../MachineLearning")
testing <- read.csv("pml-testing.csv", na.strings=c("#DIV/0","","NA"), stringsAsFactors=TRUE)
training <- read.csv("pml-training.csv",na.strings=c("#DIV/0","","NA"), stringsAsFactors=TRUE)
```

##Preprocessing
The original data-set was split into a training and testing data-set. The data-set consists of 160 variables which is quite large. The first step is to reduce the number of variables by excluding variables dominated by NA values. Based on a threshold of >50% NA per column all irrelevant columns are removed. Testing other tresholds (e.g. 20%, 80%) and counting the total NA's per column reveals that almost 100 out of the 160 columns consist completely of NA's. Removing these variables helps to reduce the original data-set from 160 to 60 columns. This is applied to both, the testing and training sets. 


```r
norow <- nrow(training)
norowt <- nrow(testing)
thresh <- norow*0.5 
thresht <- norowt*0.5

training<-training[,colSums(is.na(training)) < thresh]
testing <- testing[,colSums(is.na(testing))<thresht]
```

The data also contains mixed variable types (e.g. integer and numeric variables) which complicates the model-fitting process and causes regular error in the train() function. To avoid this, all integer columns are converted to numeric. 


```r
indx <- sapply(training, is.integer)
training[indx] <- lapply(training[indx],function(x)as.numeric(x))
testing[indx] <- lapply(testing[indx],function(x)as.numeric(x))

require(caret)
```

```
## Loading required package: caret
```

```
## Warning: package 'caret' was built under R version 3.2.2
```

```
## Loading required package: lattice
```

```
## Loading required package: ggplot2
```

```r
trainpc <- training[,7:60]
testpc <- testing[,7:59]
```

The resulting data-set consists of 60 variables and contains 19622 observations in the training set and 20 in the testing set. Of the 60 variables, the first 7 refer to user name, time-stamps and other complementary information which is not essential for the prediction and therefore removed. The time-stamps are irrelevant in this context because the aim is simply to predict whether or not the exercise is conducted correctly.  Whether or not an exercise is performed correctly in a real world example can of course be linked to a time-component (e.g. wrong execution due to exhaustion) however, this is not relevant for training the prediction algorithm because the data at hand have been undertaken in a controlled environment. For further information on this see the paper by Velloso et al. (2013).

## Exploratory Data Analysis


```r
require(corrplot)
```

```
## Loading required package: corrplot
```

```
## Warning: package 'corrplot' was built under R version 3.2.3
```

```r
correl <- cor(training[,7:59])
par(oma=c(2,2,2,2))
corrplot(correl,method="square", tl.pos="n")
```

![](mc_tamara__files/figure-html/correlation plot-1.png)\

A correlation plot of the remaining 53 columns indicates that the observed correlation between most variables is relatively low. However, exceptions like in the upper left part of the correlation plot indicate substantial correlation between some of the variables. To prevent overfitting by including to many (correlated) variables it is necessary to select the most important variables. Because this data-set is relatively large, Principal Component Analysis is applied to reduce complexity by extracting principal components that can explain the bulk of the total variation. 


## Create prediction model

To predict the correct or incorrect manner of exercise a random forest algorithm is applied to train a model using R's train() function from the caret package. The random forst algorithm is known to provide highly accurate results but being computationally expensive (e.g. because the algorithm work per default with 25 bootstrap replications). The size of the underlying data-set is acceptably small and using parallel processing allows to essentially speed things up, hence the trade-off between computation time and accuracy is deemed to be worth the effort. 

The random forest algorithm is known to be sensible to overfitting. Cross-validation can essentially help avoiding this problem. The original training set is further splitted into testing and training sets mulitiple times (10 fold in this case), e.g. the model is trained over and over again on the 10 new training and testing sets based on the original training set. The final model is then tested on the testing set. Cross-validation is specified using trainControl() function: 


```r
controlvar <- trainControl(method = "cv",number = 10, allowParallel = TRUE)
set.seed(123456)
```

Using the trainControl() function also allows to specify parallel processing. Thank's to Len Greski (https://github.com/lgreski) for the valuable input on how to speed up random forest calculations using the train function from R's caret package.


```r
library(parallel)
library(doParallel)
```

```
## Warning: package 'doParallel' was built under R version 3.2.3
```

```
## Loading required package: foreach
```

```
## Warning: package 'foreach' was built under R version 3.2.3
```

```
## Loading required package: iterators
```

```
## Warning: package 'iterators' was built under R version 3.2.2
```

```r
cluster <- makeCluster(detectCores() - 1) 
registerDoParallel(cluster)
```

To reduce complexity of the data, Principal Component Analysis (PCA; see above) is used for pre-processing and is directly implemented in the train() function. Data transformation, e.g. centering  and scaling of the variables is a precondition for perfoming PCA and is done automatically by the train function.


```r
fitpca <- train(trainpc$classe~.,method="rf",preProcess=c("pca"),trControl = controlvar,data=trainpc)
```

```
## Loading required package: randomForest
```

```
## Warning: package 'randomForest' was built under R version 3.2.2
```

```
## randomForest 4.6-12
```

```
## Type rfNews() to see new features/changes/bug fixes.
```

```r
stopCluster(cluster) # stop do.parallel process
```

# Results


```r
fitpca
```

```
## Random Forest 
## 
## 19622 samples
##    53 predictor
##     5 classes: 'A', 'B', 'C', 'D', 'E' 
## 
## Pre-processing: principal component signal extraction (53), centered
##  (53), scaled (53) 
## Resampling: Cross-Validated (10 fold) 
## Summary of sample sizes: 17661, 17660, 17659, 17661, 17659, 17660, ... 
## Resampling results across tuning parameters:
## 
##   mtry  Accuracy   Kappa      Accuracy SD  Kappa SD   
##    2    0.9832843  0.9788534  0.003246539  0.004108077
##   27    0.9761492  0.9698302  0.001874305  0.002372535
##   53    0.9765059  0.9702827  0.001565754  0.001978952
## 
## Accuracy was used to select the optimal model using  the largest value.
## The final value used for the model was mtry = 2.
```

```r
fitpca$finalModel
```

```
## 
## Call:
##  randomForest(x = x, y = y, mtry = param$mtry) 
##                Type of random forest: classification
##                      Number of trees: 500
## No. of variables tried at each split: 2
## 
##         OOB estimate of  error rate: 1.46%
## Confusion matrix:
##      A    B    C    D    E class.error
## A 5555    8   11    4    2 0.004480287
## B   39 3727   27    0    4 0.018435607
## C    4   31 3366   19    2 0.016364699
## D    3    2   97 3109    5 0.033271144
## E    0    7    7   15 3578 0.008039922
```

The out of sample error indicates how well the trained model is able to predict the quality of exercise undertaken. Because the model was trained using cross-validation (e.g. training and testing the model on subsets of the training data), the error rate (OOB) as indicated by the final Model fit can be interpreted as the out of sample error. The error rate of 1.46% indicates a realatively high accuracy of the fitted 
model. This is also confirmed by the high sample accuracy of >98%. Using the alogorithm to predict the manner for the testing set reveals that all 20 cases of the testing set have been predicted correctly. 


```r
testpca<- predict(fitpca,testpc)
testpca
```

```
##  [1] B A B A A E D B A A B C B A E E A B B B
## Levels: A B C D E
```

# Conclusions

This analysis presents a model based on random-forest machine learning algorithm using Principal Components Analysis and cross-validation to detect if weight lifting exercises are perfomed well. Results indicate that the algorithm is able to predict mistakes in exercise with an accuracy > 98% going along with a low error rate of 1.46%. Such information is important because performing exercises in a correct manner can provide substantial health benefits and help avoiding injuries. 

# References
Max Kuhn. Contributions from Jed Wing, Steve Weston, Andre Williams, Chris Keefer, Allan Engelhardt, Tony Cooper,   Zachary Mayer, Brenton Kenkel, the R Core Team, Michael Benesty, Reynald Lescarbeau, Andrew Ziem, Luca Scrucca,   Yuan Tang and Can Candan. (2015). caret: Classification and Regression Training. R package version 6.0-58. http://CRAN.R-project.org/package=caret

R Core Team (2015). R: A language and environment for statistical computing. R Foundation for Statistical           Computing, Vienna, Austria. URL http://www.R-project.org/.

Revolution Analytics and Steve Weston (2015). doParallel: Foreach Parallel Adaptor for the 'parallel' Package. R    package version 1.0.10. http://CRAN.R-project.org/package=doParallel

Taiyun Wei (2013). corrplot: Visualization of a correlation matrix. R package version 0.73.
  http://CRAN.R-project.org/package=corrplot

Velloso, E.; Bulling, A.; Gellersen, H.; Ugulino, W.; Fuks, H. Qualitative Activity Recognition of Weight Lifting   Exercises. Proceedings of 4th International Conference in Cooperation with SIGCHI (Augmented Human '13) .         Stuttgart, Germany: ACM SIGCHI, 2013.

