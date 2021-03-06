# Practical ML Course Project ML CV
Raja Kumar  
Tuesday, May 19, 2015  
This report will analyze data collected from [http://groupware.les.inf.puc-rio.br/har] (see the section on the Weight Lifting Exercise Dataset). 

The training data is available at
[https://d396qusza40orc.cloudfront.net/predmachlearn/pml-training.csv]
and the test data is available at
[https://d396qusza40orc.cloudfront.net/predmachlearn/pml-testing.csv]

The goal of this project is to predict the manner in which users in the datasets did the exercise. This is the "classe" variable in the training set. 

#Building the model#
There were 3 main steps required to build the model:
1.Selecting the predictors. First all the columns which are not available in the test set were removed.  

2.Variables containing the user name, timestap and num_window were removed because they are not good predictors. Any user can do any movement at any time and hence these variables are not used as predictors.  

3.Build a model using random forests with "classe" of activity as the output variable and all other remaining variables as predictors.  

#Loading and cleaning the data#
The code requires pml-training.csv and pml-testing.csv files in the currenty working directory





```r
set.seed(5000)
trainingSet<-read.table("pml-training.csv",header = 1,sep= ",")
testSet<-read.csv("pml-testing.csv")

# remove all columns which are NA in the testset.
coldropNA=c()
for(col in seq_along(colnames(testSet))){
	if(sum(is.na(testSet[,col]))>10){
		coldropNA<-append(coldropNA,col)
	}   
}
testSet<-testSet[,-coldropNA]
trainingSet<-trainingSet[,-coldropNA]


removeUnwantedCols<-function (dropmatch,data){
	coldrop<-grep(dropmatch,x = colnames(data),value = FALSE)
	data[,-coldrop]
}
# remove the time,user,x and num_window variables
trainingSet<-removeUnwantedCols("time|user|X|num_window",trainingSet)
testSet<-removeUnwantedCols("time|user|X|num_window",testSet)
```

#Expected out of sample error and rationale for choices#
Only 60% of the data was used to build the model (training set), the remaining 40% was kept for testing. Reason for using a small sample are related to computational cost and improved validity in out of sample testing.  
5-fold cross validation was used on the 60% of the data that was used as the training set.
A higher value of folds would result in lower bias but higher variance, i.e overfitting. Hence in order to both avoid overfitting and keep the computational time down, 5 fold cross validation was used.

```r
# keep 60% of the data for training and 40% for testing
inTrain<-createDataPartition(trainingSet$classe,p = 0.6,list = FALSE)
trainingSet1<-trainingSet[inTrain,]
testSet1<-trainingSet[-inTrain,]

#initializae seed
set.seed(5000)
```


```r
# setup required for parallel RF
cl <- makePSOCKcluster(7)
clusterEvalQ(cl, library(foreach))
registerDoParallel(cl)
```

#Cross Validation#
Cross validation is a procedure used to reduce overfitting of the model and reduce the variance in out of sample prediction error. Crossvalidation also makes efficient use of data by using the entire training data for building the model and testing it.

In this case we will do a simple 5 fold cross validation, which means that the training set gets divided into 5 sets and one is reserved for testing. The model is built on 4/5ths of the original training data set and tested on the 1/5th which was reserved for testing. This process is then repeated 5 times and the average errors are reported. 

```r
fitControl<- trainControl(method="repeatedcv",	number=5,repeats=5, allowParallel=1)
# fit the random forest using the parallelized version
modelfit1<-train(y=trainingSet1[,54],x=trainingSet1[,-54],method="parRF",trControl=fitControl)
```


```r
# Check the accuracy
# Check the fit on the training set
fit1<-modelfit1
trainPred<-predict(fit1,trainingSet1)
table(trainingSet1$classe,trainPred)
```

```
##    trainPred
##        A    B    C    D    E
##   A 3348    0    0    0    0
##   B    0 2279    0    0    0
##   C    0    0 2054    0    0
##   D    0    0    0 1930    0
##   E    0    0    0    0 2165
```

```r
# Check the fit on the test set
testPred<-predict(fit1,testSet1)
table(testSet1$classe,testPred)
```

```
##    testPred
##        A    B    C    D    E
##   A 2232    0    0    0    0
##   B   19 1493    6    0    0
##   C    0    8 1351    9    0
##   D    0    0   15 1268    3
##   E    0    0    4    4 1434
```

```r
acc<-1-sum(testPred!=testSet1$classe)/length(testPred)
print(paste0("Accuracy on test set =",round(acc*100,2),"%"))
```

```
## [1] "Accuracy on test set =99.13%"
```
#Description of the model and testing#
The model constructed is a randomforest. A random forest uses the classification tree approach but instead of building up one tree, many such trees are built and the final prediction is decided by a vote. 
Since the caret package is used it automatically optimizes the number of predictors used as 27
For a detailed explanation of how randomforests are constructed refer to: 
[http://www.stat.berkeley.edu/~breiman/RandomForests/cc_home.htm#ooberr](http://www.stat.berkeley.edu/~breiman/RandomForests/cc_home.htm#ooberr)


```r
plot(modelfit1)
```

![](PML_Cross_Validation_Course_project_files/figure-html/unnamed-chunk-6-1.png) 

```r
modelfit1
```

```
## Parallel Random Forest 
## 
## 11776 samples
##    53 predictor
##     5 classes: 'A', 'B', 'C', 'D', 'E' 
## 
## No pre-processing
## Resampling: Cross-Validated (5 fold, repeated 5 times) 
## 
## Summary of sample sizes: 9421, 9420, 9420, 9421, 9422, 9421, ... 
## 
## Resampling results across tuning parameters:
## 
##   mtry  Accuracy   Kappa      Accuracy SD  Kappa SD   
##    2    0.9882473  0.9851315  0.002170762  0.002747007
##   27    0.9887905  0.9858190  0.002531238  0.003203548
##   53    0.9825235  0.9778900  0.003177879  0.004021925
## 
## Accuracy was used to select the optimal model using  the largest value.
## The final value used for the model was mtry = 27.
```

```r
#confusionMatrix(trainingSet1$classe,trainPred)
confusionMatrix(testSet1$classe,testPred)
```

```
## Confusion Matrix and Statistics
## 
##           Reference
## Prediction    A    B    C    D    E
##          A 2232    0    0    0    0
##          B   19 1493    6    0    0
##          C    0    8 1351    9    0
##          D    0    0   15 1268    3
##          E    0    0    4    4 1434
## 
## Overall Statistics
##                                          
##                Accuracy : 0.9913         
##                  95% CI : (0.989, 0.9933)
##     No Information Rate : 0.2869         
##     P-Value [Acc > NIR] : < 2.2e-16      
##                                          
##                   Kappa : 0.989          
##  Mcnemar's Test P-Value : NA             
## 
## Statistics by Class:
## 
##                      Class: A Class: B Class: C Class: D Class: E
## Sensitivity            0.9916   0.9947   0.9818   0.9899   0.9979
## Specificity            1.0000   0.9961   0.9974   0.9973   0.9988
## Pos Pred Value         1.0000   0.9835   0.9876   0.9860   0.9945
## Neg Pred Value         0.9966   0.9987   0.9961   0.9980   0.9995
## Prevalence             0.2869   0.1913   0.1754   0.1633   0.1832
## Detection Rate         0.2845   0.1903   0.1722   0.1616   0.1828
## Detection Prevalence   0.2845   0.1935   0.1744   0.1639   0.1838
## Balanced Accuracy      0.9958   0.9954   0.9896   0.9936   0.9983
```

#Predict 20 different test cases, final submission#
Once we have the fitted model we calculate our predictions on the supplied testset for which 
the actual classe values had not been provided.

The results were submitted online and 20/20 results were correct.
This is in line with the high accuracy level seen when testing predictions on the 40% of the data reserved for testing

```r
pml_write_files = function(x){
  n = length(x)
  for(i in 1:n){
    filename = paste0("problem_id_",i,".txt")
    write.table(x[i],file=filename,quote=FALSE,row.names=FALSE,col.names=FALSE)
  }
}

# make sure that the testSet has the same factor levels as the trainingSet
levels(testSet$new_window) <- levels(trainingSet1$new_window)

answers=predict(modelfit1,testSet)
answers<-as.character(answers)
answers
```

```
##  [1] "B" "A" "B" "A" "A" "E" "D" "B" "A" "A" "B" "C" "B" "A" "E" "E" "A"
## [18] "B" "B" "B"
```

```r
length(answers)
```

```
## [1] 20
```

```r
#output the answers to file for submission
pml_write_files(answers)
```
