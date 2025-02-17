---
title: "Practical Machine Learning - Barbell lifts"
author: "Tau-Sigma"
date: "21. September 2015"
output: html_document
---

### Management summary
This analysis will show how one can predict whether a barbell lift was performed correctly or incorrectly. To achieve this, we will first do some exploratory data analysis to see if we can already get some hints about which machine learning techniques might yield the best results for the task at hand. We will then use information gathered this way to try various methods in order to find the one, that gives sufficiently satisfying results to the question posed. The data used for this analysis was generously made available by the folks of http://groupware.les.inf.puc-rio.br/har. 

### Exploratory data analysis
##Data loading
The data is loaded as follows:


```r
setwd("/home/sascha/Dokumente/Coursera/Practical Machine Learning/Assignment/barbell_lifts")
barbel_train<- read.csv("./data/pml-training.csv")
barbel_test<- read.csv("./data/pml-testing.csv")
```

## The data
The original data was aquired from 6 participants performing a weight lifting excersise. They were instructed to perform the excersise in 5 different ways, each time with 10 repetitions. The data was gathered with 4 sensors, one attached to the hand, the arm, the waist and the weight. Each sensor collected three-axes
acceleration,  gyroscope  and  magnetometer  data. The dataset also includes the Euler angles (roll, pitch and yaw) for each sensor. The following table shows how many observations our original training data set has for each participant and excersise type:


```r
table(barbel_train$user_name, barbel_train$classe)
```

```
##           
##               A    B    C    D    E
##   adelmo   1165  776  750  515  686
##   carlitos  834  690  493  486  609
##   charles   899  745  539  642  711
##   eurico    865  592  489  582  542
##   jeremy   1177  489  652  522  562
##   pedro     640  505  499  469  497
```

## Missing values
Looking at the training data it becomes clear that many of the available variables have a lot of missing values (19216 of 19622 observations in total). The reason for this is that the researchers used a sliding window approach to calculate features like mean, standard deviation, etc. for the collected data over a defined timeframe. These calculated features have only been added to observations which represent the end of a window (timeframe). 

The performance of an excersise consists of a continous flow of movement. When judging how an excersise was performed it makes sense to try to find features that represent the execution in its totality or certain parts (time slices) of it. However, to validate our model we are to use a test set, which does not include the data of a full execution of the excersise. The calculated features are missing in the test set. Instead it only includes a very limited number of measurements. This basically means that in order to achieve good results on the test set we will have to optimize our model with regard to single measurements instead of a movement over a certain timeframe.

We will therefore drop all variables with calculated features using the number of NAs in the test data set as an identifier. We will therefore also drop the variables raw_timestamp_part_1 and 2, cvtd_timestamp, new_window and num_window as they are all timeframe related (they define the order of measurements and to which timeframe it belongs). We drop X as it is just unique identifier for observations and we drop the user_name as we want our model to generalize well instead of being able to predict how a specific user performed. The test data table also includes a variable called problem_id which is not part of the training set and therefore needs to be removed, while we have to add a placeholder for the classe variable, which we want to predict.

After applying these changes to the test data, we use this table as a template for our training data table, dropping all variables that are not included in the test data table. This leaves us with 52 potential numeric predictor variables for our model. 

The training data set is then split into a training and a cross validation data set (at the ratio of 60 to 40). All of the above is done using the following code:


```r
#remove X, user_name, raw_timestamp_part_1 and 2, cvtd_timestamp, new_window and num_window
barbel_test <- barbel_test[,-c(1:7)]
#remove na variables and problem_id from test data
barbel_test <- barbel_test[,colSums(is.na(barbel_test))<20]
barbel_test <- barbel_test[,!(names(barbel_test)=="problem_id")]
#add placeholder variable for predictions to test data set
barbel_test$classe<-as.character("")
#only keep variables that occur in both reduced tables
barbel_train<-barbel_train[,intersect(names(barbel_train), names(barbel_test))]
#Split training data into training and cross-validation dataset
library(caret);
set.seed(32617)
inTrain <- createDataPartition(y=barbel_train$classe,p=0.6, list=FALSE)
training <- barbel_train[inTrain,]
cross_valid <- barbel_train[-inTrain,]
```

### Modelling
Instead of spending a lot of time on thinking about how to transform the 52 features we have, we decided to try different algorithms, basically with their default settings, and see how they would perform on our prediction problem. As the challenge we face is to predict a factor variable with 5 possible outcomes various approaches come to mind 
The first approach that yielded any results, was a simple decision tree. We also tried more sophisticated approaches like logistic regression, boosting and random forrests, but they required some additional research, before we saw any results.

## The simple approach
Modeling the problem with a decision tree quickly yields results. Running the following code gives us a first model just after a few seconds.


```r
if (file.exists("./cache/modelFit1.rda")) {
  load("./cache/modelFit1.rda")
} else {
  modelFit1<-train(training$classe~., method="rpart", data=training)
  save(modelFit1, file= "./cache/modelFit1.rda")}  
quality1<-confusionMatrix(cross_valid$classe, predict(modelFit1, cross_valid))
quality1$overall
```

```
##       Accuracy          Kappa  AccuracyLower  AccuracyUpper   AccuracyNull 
##      0.4946470      0.3404334      0.4835240      0.5057738      0.5081570 
## AccuracyPValue  McnemarPValue 
##      0.9919097            NaN
```

However, applying the model to our cross validation data shows us that the accuracy of our predictions is below 50%, a very unsatisfactory result.

## An alternative approach - Logistic regression
When using logistic regression to classify observations into more than 2 potential groups one can use a one vs. all approach. This means that one takes the estimand and brakes it into n (in our case 5) new binary variables, where for each new variable another factor level is set to 1 while all others are set to 0. Then for each of these new variables a model is build. Afterwards all (in our case 5) models are used to calculate a prediction for the respective ne variable. These values represent the probability that the observative is of the classe, which the respective new variable represents, making the variable with the highest calculated probability our predicted classee. 

Trying to implement this approach by simply choosing "glm" as method gives warning messages, which hint at the necessity of using regularization in order to penalize letting the model coefficients getting to big. We therefore use the method "glmnet" instead. However, it soon becomes clear that this algorithm is computationally much more expensive than the algorithm for the previously modelled tree, as the computation just wtakes ages on the dual core Celeron we are using, when writing our code. Luckily we can also run the algorithm on a Core i7, but this also does not speed things up very much. Some additional research shows tat there are libraries available, which can parallize the computations by makeing use of more than just one core of the processor, which allowes us to calculate the model in approx. 10 minutes. This is the code we use:


```r
library(doMC); registerDoMC(cores=12);library(glmnet); #to make use of a multicore processor and the glmnet library
if (file.exists("./cache/modelFit2.rda")) {load("./cache/modelFit2.rda")
} else {
  modelFit2<-train(classe~., data = training, method="glmnet")
  save(modelFit2, file= "./cache/modelFit2.rda")}
quality2<-confusionMatrix(cross_valid$classe,predict(modelFit2,cross_valid))
quality2$overall
```

```
##       Accuracy          Kappa  AccuracyLower  AccuracyUpper   AccuracyNull 
##   7.286515e-01   6.555312e-01   7.186652e-01   7.384678e-01   3.140454e-01 
## AccuracyPValue  McnemarPValue 
##   0.000000e+00   1.743184e-29
```

As can be seen from the above overview, the accuracy of this model is much higher. It is capable of classifying approx. 72% of our observations in the cross validation set correctly.

## Boosting
The boosting algorithm prooves to be computationally also expensive, yet much cheaper than logistic regression (about 3-5 Minutes). We create a model as follows:


```r
if (file.exists("./cache/modelFit3.rda")) {load("./cache/modelFit3.rda")
} else {
  modelFit3<-train(classe~., data = training, method="gbm")
  save(modelFit3, file= "./cache/modelFit3.rda")}
quality3<-confusionMatrix(cross_valid$classe,predict(modelFit3,cross_valid))
quality3$overall
```

```
##       Accuracy          Kappa  AccuracyLower  AccuracyUpper   AccuracyNull 
##   9.580678e-01   9.469378e-01   9.533957e-01   9.623956e-01   2.884272e-01 
## AccuracyPValue  McnemarPValue 
##   0.000000e+00   2.740742e-10
```

Although the calculations are performed much faster, the results on our cross validation data are much better. Accuracy is now at approx. 96%.

## Random forests
The last algorithm we are trying is the random forrest, which again is computationaly very expensive. We use the following code:


```r
if (file.exists("./cache/modelFit4.rda")) {load("./cache/modelFit4.rda")
} else {
  modelFit4<-train(classe~., data = training, method="rf")
  save(modelFit4, file= "./cache/modelFit4.rda")}  
quality4<-confusionMatrix(cross_valid$classe,predict(modelFit4,cross_valid))
quality4$overall
```

```
##       Accuracy          Kappa  AccuracyLower  AccuracyUpper   AccuracyNull 
##      0.9899312      0.9872614      0.9874668      0.9920205      0.2862605 
## AccuracyPValue  McnemarPValue 
##      0.0000000            NaN
```

The accuracy we can achieve on our cross validation set with our random forest model reaches nearly 99%.

With such a high accuracy it does not seem necessary to try building additional models with alternative features. 

###Predicting classes of the test data set
Having build a random forrest which achieved a 99% accuracy on our cross validation data, we will use this model to predict the classes of our 20 test data sets as follows:


```r
predictions<-predict(modelFit4,barbel_test)
predictions
```

```
##  [1] B A B A A E D B A A B C B A E E A B B B
## Levels: A B C D E
```
