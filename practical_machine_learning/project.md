---
title: "Practical Machine Learning Course Project"
author: "Mark Zhang"
date: "July 18, 2018"
output:
  pdf_document: default
  html_document: default
---

####Overview

This project concerns about classfying different type of human activities by using data collected from body activity tracking devices. Since the number of classes to be classified has more than 2 outcomes, so we will implement classification tree, random forest, and boosting classifiers to see which one performs the best, and then use the best performer to predict on the test set.

####Setups
```{r setup, message=FALSE}
knitr::opts_chunk$set(message = F, fig.align = 'center')
require(caret)
require(dplyr)
require(rattle)
```

####Read data
```{r}
trainSet <- read.csv(url('https://d396qusza40orc.cloudfront.net/predmachlearn/pml-training.csv'))
testSetFinal <- read.csv(url('https://d396qusza40orc.cloudfront.net/predmachlearn/pml-testing.csv'))
```

####Custom function
```{r}
getCfMTX <- function (model, data) {
    pred <- predict(model, data)
    confusionMatrix(pred, data$classe)
}
```

####Exploratory analysis

By looking at the structure of the dataset, we can see that there are a lot of columns with NA value or blank value. By examine more closely, we can see that all the columns with either NA or blank values, their number of NA or blank values are the same: 19216, which is accountable for about 97% of the data size, so we are going to drop these columns.
```{r}
str(trainSet)
colSums(is.na(trainSet) | trainSet == "")[colSums(is.na(trainSet) | trainSet == "") != 0]
19216/19622
```

####Data cleansing

By removing columns with NAs or blank values, and first 7 columns, which are not human activities tracking data, we have our dataset tidy. And then we subset 20% of data from the training set as our validation set to calculate the out of sample error rate.
```{r}
columnsRemove <- names(colSums(is.na(trainSet) | trainSet == "")[colSums(is.na(trainSet) | trainSet == "") != 0])
trainSet <- select(trainSet, -columnsRemove, -(X:num_window))

set.seed(0871)
sampledIdx <- createDataPartition(y = trainSet$classe, p = 0.8, list = F)
trainSubset <- trainSet[sampledIdx,]
valiSet <- trainSet[-sampledIdx,]
```

####Modeling - rpart

We first train the rpart model with 3-folds cross-validation (same cv technique also used for the subsequent two models).
```{r, cache = T}
trCtrl <- trainControl(method = 'cv', number = 3)

fit_rpart <- train(classe ~ ., 
                   method = 'rpart', 
                   data = trainSubset, 
                   trControl = trCtrl)
fancyRpartPlot(fit_rpart$finalModel)
getCfMTX(fit_rpart, valiSet)
```
As the result shows, the accuracy is not quite satisfying.

####Modeling - rf

Then let's take a look how random forest perform.
```{r, cache=T}
fit_rf <- train(classe ~ ., 
                method = 'rf', 
                data = trainSubset, 
                trControl = trCtrl)
getCfMTX(fit_rf, valiSet)
varImp(fit_rf$finalModel)
```
It looks like random forest is pretty good at handling this type of data!

####Modeling - gbm

Finally, let's examine the performance of boosting.
```{r, cache=T}
fit_gbm <- train(classe ~ ., 
                 method = 'gbm', 
                 data = trainSubset, 
                 trControl = trCtrl, 
                 verbose = F)
getCfMTX(fit_gbm, valiSet)
```
Not bad, but still underperforms the random forest

####Prediction

Since the random forest is the best performer, so we use it as our final model to make prediction on the test set.
```{r}
predict(fit_rf, testSetFinal)
```

