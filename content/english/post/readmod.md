---
author: Makayla Whitney
date: "2022-07-03"
description: model development process
tags:
- markdown
- text
series:
- series-setup
tags:
- readability
- modelproduction
- linearregression
title: Readability Model
---

# Readability
The term "readability" describes the level of difficulty a written passage is. Factors of word choice, sentence length, and grammatical attributes contribute to the overall difficulty of a passage. In traditional school settings, leveled books exist for children to expand their reading skills within their designated grade level. These books are written within guidelines that fall within state- and nation-wide English Language Arts standards. What if there is a passage of text that does not have a designated grade level? A teacher could manually read through and check the passage for indicators. This takes significant time and an expansive knowledge on grammar. This project explores the use of linear regression and text features to model readability scores. 

## Significance
The readability of a passage determines how well a reader interacts with the overall content of a passage. If they are struggling to understand the basic words and structure, they will not be able to comprehend and absorb the overall story line. All readers are unique in the way that they approach a text. The impact that a text's readability and features has varies from student to student (1). It is important for teachers and literacy practitioners to have not only an understanding of their students' reading level, but the readability score of passages being presented. This score assists teachers and practitioners in providing reading assistance and resources calibrated to the individual student. 

# Model Description
The readability model featured in this series was developed in EDUC 654: Machine Learning taught by Dr. Cengiz Zopluoglu at the University of Oregon. Text features as well as the model were created with text data from the [CommonLit Readability Kaggle Competition](https://www.kaggle.com/c/commonlitreadabilityprize/ "CommonLit Readability Kaggle Competition"). In the previous post, I displayed how to gather text features to assist with predicting readability of a passage. In the following post, I will demonstrate how the readability prediction model was constructed. 

## Ridge Line Penalty
Performance evaluation metrics were used to evaluate which logistic regression model produced the best results. Models analyzed included logistic regression, logistic regression with ridge penalty, logistic regression with lasso penalty, and logistic regression with elastic net. Information collected from each model tested included the R-squared measurement, the Mean Absolute Error (MAE), and the Root Mean Squared Error (RMSE). Out of these four models, the model with logistic regression with ridge penalty performed the best with the following statistics:

| R-squared |     MAE   |     RMSE  |
|-----------|-----------|-----------|
| 0.7274006 | 0.4347177 | 0.5355444 |

Ridge penalty terms were added to the loss function to avoid large coefficients. By including a ridge penalty, we reduced model variance in exchange of adding bias [Lecture 5a](https://ml-21.netlify.app/notes/lecture-5a.html#21_Ridge_Penalty "Lecture 5a").

## Building the Model
To build the model, we began by downloading the necessary packages. 
```
require(caret)
require(recipes)
require(finalfit)
require(glmnet)
require(finalfit)
```
The data set is then imported. Readability is the data set obtained from the CommonLit Readability Kaggle Competition. We set the seed to allow for reproducibility.
```
readability <- read.csv('https://raw.githubusercontent.com/uo-datasci-specialization/c4-ml-fall-2021/main/data/readability_features.csv',header=TRUE)

set.seed(10152021)
```
### Training and Test Sets
A crucial set in building the model is training part of the available data as the model and then testing the accuracy of the model with the other part of the data. We begin with creating a list of random numbers ranging from 1 to number of rows from the data set and put 90% of the data into the training data. The other 10% serves as our test data.  
```
loc      <- sample(1:nrow(readability), round(nrow(readability) * 0.9))
read_tr  <- readability[loc, ]
read_te  <- readability[-loc, ]
```

### Blueprint
We use the [`recipe` package](https://www.rdocumentation.org/packages/recipes/versions/0.2.0/topics/recipe "`recipe` package") in R to develop a blueprint for our model. In this package, a _recipe_ consists of a description of the steps applied to a data set to assist with preparing it for data analysis. This function requires a data set, vars, and roles. In this case, _vars_ is a character string of column names corresponding to the variables within the data set and _roles_ is a character string, the same length as vars, that describes the roles each variable will take within the model.

Once these parameters were established, steps were added to the blueprint. These are similar to steps in a cooking recipe. For example, when making banana bread, you gather the ingredients and add them to a mixing bowl in a particular order. Shown below, we added our steps in an order that was ideal for our model.

```
  blueprint <- recipe(x     = readability,
                      vars  = colnames(readability),
                      roles = c(rep('predictor',990),'outcome')) %>%
    step_zv(all_numeric()) %>%
    step_nzv(all_numeric()) %>%
    step_impute_mean(all_numeric()) %>%
    step_normalize(all_numeric_predictors()) %>%
    step_corr(all_numeric(),threshold=0.9)
```
The table below gives a basic description of each of the steps used within this blueprint. For more information and step possibilities, please visit the `recipe` package link above. Click on the links in the table to find out more about those steps. . 

|         Step        |                                       Description                                         |
|---------------------|-------------------------------------------------------------------------------------------|
| [step_zv](https://www.rdocumentation.org/packages/recipes/versions/0.2.0/topics/step_zv "step_zv")             |   removes variables containing only a single value                                        |
| [step_nzv](https://www.rdocumentation.org/packages/recipes/versions/0.2.0/topics/step_nzv "step_nzv")            |   removes highly sparse and unbalanced variables                                          |
| [step_impute_mean](https://www.rdocumentation.org/packages/recipes/versions/0.2.0/topics/step_impute_mean "step_impute_mean")    |   substitutes missing values of numeric variables with the mean of training set variables |
| [step_normalize](https://www.rdocumentation.org/packages/recipes/versions/0.2.0/topics/step_normalize "step_normalize")      |   normalizes data to have an sd of 1 and a mean of 0                                      |
| [step_corr](https://www.rdocumentation.org/packages/recipes/versions/0.2.0/topics/step_corr "step_corr")           |   removes variables that have large absolute correlations with other variables            |

### Cross-Validation and Grid Tuning
Cross-validation was conducted to judge the overall performance and accuracy of the model. 

```
# Cross validation settings
  
# Randomly shuffle the data

read_tr = read_tr[sample(nrow(read_tr)),]

# Create 10 folds with equal size

folds = cut(seq(1,nrow(read_tr)),breaks=10,labels=FALSE)
  
# Create the list for each fold 
      
my.indices <- vector('list',10)
for(i in 1:10){
  my.indices[[i]] <- which(folds!=i)
}
      
cv <- trainControl(method = "cv",
                  index  = my.indices)
```
By optimizing the degree of ridge penalty via tuning, we can typically get models with better performance than a logistic regression with no regularization. In our case, the optimal lambda, after being tested, was determined as 0.57. This proved to tune the best grid for the model. 

```
grid <- data.frame(alpha = 0, lambda = 0.57) 
grid

```
The final step in building the model is to train it.

```
ridge <- caret::train(blueprint, 
                      data      = read_tr, 
                      method    = "glmnet", 
                      trControl = cv,
                      tuneGrid  = grid)
```
Please move on to the next post in this series **Model Validation with easyCBM** to read about validating the model developed here with a set of text and accompanying student data.For more information regarding logistic regression, refer to Dr. Zopluoglu's lectures [Regularization in Linear Regression](https://ml-21.netlify.app/notes/lecture-4b.html#Regularization "Regularization in Linear Regression") and [Logistic Regression and Regularization](https://ml-21.netlify.app/notes/lecture-5a.html#1_Overview_of_the_Logistic_Regression "Logistic Regression and Regularization").

Resources
1. Francis, D. J., Kulesz, P. A., & Benoit, J. S. (2018). Extending the simple view of reading to account for variation within readers and across texts: The complete view of reading (CVRi). Remedial and Special Education, 39(5), 274-288. doi:/doi.org/10.1177/07419325187729