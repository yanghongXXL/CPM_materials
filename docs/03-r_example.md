

# 实操教学 {.unnumbered}

# R 实操

## **机器学习模型构建**

### **加载需要的r包**


```r
# rm(list = ls())
library(tidyverse) # 数据整理
library(dlookr) # 自动化EDA
library(plotROC) # 绘制ROC
library(pROC) # 计算AUC
library(e1071) # 支持向量机
library(caret) # 机器学习包
library(ggplot2)
library(rms) # 计算校准曲线\绘制nomo
library(glmnet) # lasso
```

### **加载数据**


```r
data <- read_csv("data/CardiovascularDataset.csv")
# dlookr 是一个自动输出一份数据诊断报告包，可以自行探索
# eda_report(data) # 输出诊断报告
# eda_report(data, target = 1) # 添加按照target的分组信息
```

-   查看数据情况


```r
# 查看前10行树
head(data,10)
```

```
## # A tibble: 10 x 14
##     age   sex    cp trest~1  chol   fbs restecg thalach
##   <dbl> <dbl> <dbl>   <dbl> <dbl> <dbl>   <dbl>   <dbl>
## 1    63     1     3     145   233     1       0     150
## 2    53     1     0     140   203     1       0     155
## 3    41     0     1     130   204     0       0     172
## 4    56     1     1     120   236     0       1     178
## 5    60     1     0     130   206     0       0     132
## 6    57     1     0     140   192     0       1     148
## # ... with 4 more rows, 6 more variables: exang <dbl>,
## #   oldpeak <dbl>, slope <dbl>, ca <dbl>, thal <dbl>,
## #   target <dbl>, and abbreviated variable name
## #   1: trestbps
```


```r
# 查看数据变量属性
str(data)
```

```
## spc_tbl_ [303 x 14] (S3: spec_tbl_df/tbl_df/tbl/data.frame)
##  $ age     : num [1:303] 63 53 41 56 60 57 56 44 52 57 ...
##  $ sex     : num [1:303] 1 1 0 1 1 1 0 1 1 1 ...
##  $ cp      : num [1:303] 3 0 1 1 0 0 1 1 2 2 ...
##  $ trestbps: num [1:303] 145 140 130 120 130 140 140 120 172 150 ...
##  $ chol    : num [1:303] 233 203 204 236 206 192 294 263 199 168 ...
##  $ fbs     : num [1:303] 1 1 0 0 0 0 0 0 1 0 ...
##  $ restecg : num [1:303] 0 0 0 1 0 1 0 1 1 1 ...
##  $ thalach : num [1:303] 150 155 172 178 132 148 153 173 162 174 ...
##  $ exang   : num [1:303] 0 1 0 0 1 0 0 0 0 0 ...
##  $ oldpeak : num [1:303] 2.3 3.1 1.4 0.8 2.4 0.4 1.3 0 0.5 1.6 ...
##  $ slope   : num [1:303] 0 0 2 2 1 1 1 2 2 2 ...
##  $ ca      : num [1:303] 0 0 0 0 2 0 0 0 0 0 ...
##  $ thal    : num [1:303] 1 3 2 2 3 1 2 3 3 2 ...
##  $ target  : num [1:303] 1 0 1 1 0 1 1 1 1 1 ...
##  - attr(*, "spec")=
##   .. cols(
##   ..   age = col_double(),
##   ..   sex = col_double(),
##   ..   cp = col_double(),
##   ..   trestbps = col_double(),
##   ..   chol = col_double(),
##   ..   fbs = col_double(),
##   ..   restecg = col_double(),
##   ..   thalach = col_double(),
##   ..   exang = col_double(),
##   ..   oldpeak = col_double(),
##   ..   slope = col_double(),
##   ..   ca = col_double(),
##   ..   thal = col_double(),
##   ..   target = col_double()
##   .. )
##  - attr(*, "problems")=<externalptr>
```

### 划分数据集

使用caret包来生成并比较不同的模型与性能。

使用sample()函数，将数据集按照8:2随机拆分为训练集（264例）和测试集（61例）


```r
set.seed(1002)
trainset <- sample(nrow(data), 0.8*nrow(data))
testset <- data[-trainset,]
trainset <- data[trainset, ]
```

-   为方便后续建模，将target转为factor


```r
trainset$target <- ifelse(trainset$target == 1, "yes", "no")
testset$target <- ifelse(testset$target == 1, "yes", "no")
trainset$target <- as.factor(trainset$target)
testset$target <- as.factor(testset$target)
```


```r
str(trainset$target)
```

```
##  Factor w/ 2 levels "no","yes": 1 2 1 2 2 1 1 1 1 1 ...
```

```r
# str(testset)
```

### lasso变量筛选

在统计学习中存在一个重要理论：**方差权衡**。一般常理认为模型建立得越复杂，分析和预测效果应该越好。而方差权衡恰恰指出了其中的弊端。复杂的模型一般对已知数据（training sample）的拟合（fitting）大过于简单模型，但是**复杂模型很容易对数据出现过度拟合（over-fitting）**。因为所有实际数据都会有各种形式的误差，过度拟合相当于把误差也当做有用的信息进行学习。**所以在未知数据（test sample）上的分析和预测效果会大大下降。**

下图说明了方差权衡的结果。模型复杂度在**最低的时候（比如线性回归）预测的偏差比较大，但是方差很小**。随着模型复杂度的增大，对已知数据的预测误差会一直下降（因为拟合度增大），而对未知数据却出现拐点，一旦过于复杂，预测方差会变大，模型变得非常不稳定。

![](images/%E6%96%B9%E5%B7%AE%E6%9D%83%E8%A1%A1-%E8%BF%87%E6%8B%9F%E5%90%88%E4%B8%8E%E6%8B%9F%E5%90%88%E4%B8%8D%E8%B6%B3.jpg){width="100%"}

因此在很多实际生活应用中，线性模型因为其预测方差小，参数估计稳定可靠，仍然起着相当大的作用。正如上面的方差权衡所述，**建立线性模型中一个重要的问题就是变量选择（或者叫模型选择），指的是选择建立线性模型所用到的独立变量的选择。**在实际问题例如疾病风险控制中，独立变量一般会有200 \~ 300个之多。如果使用所有的变量，很可能会出现模型的过度拟合。所以对**变量的选择显得尤为重要**。

传统的变量选择是采用逐步回归法（stepwise selection），其中又分为向前（forward）和向后（backward）的逐步回归。向前逐步是从0个变量开始逐步加入变量，而向后逐步是从所有变量的集合开始逐次去掉变量。加入或去掉变量一般按照标准的统计信息量来决定。**这种传统的变量选择的弊端是模型的方差一般会比较高，而且灵活性较差**。

近年来回归分析中的一个重大突破是引入了正则化回归（regularized regression）的概念, 而最受关注和广泛应用的**正则化回归**是1996年由现任斯坦福教授的Robert Tibshirani提出的**LASSO回归**。**LASSO回归最突出的优势在于通过对所有变量系数进行回归惩罚（penalized regression）, 使得相对不重要的独立变量系数变为0，从而排除在建模之外。**

LASSO方法不同于传统的逐步回归的最大之处是**它可以对所有独立变量同时进行处理，而不是逐步处理**。这一改进使得建模的**稳定性大大增加**。除此以外，LASSO还具有**计算速度快**，模型容易解释等很多优点。而模型发明者Tibshirani教授也因此获得当年的有统计学诺贝尔奖之称的考普斯总统奖（COPSS award）。

#### 训练集中变量筛选

1.  首先告诉软件，哪些是你的分类变量。分类变量转为因子


```r
data1 <- trainset

data1[,c(2,3,6,7,9,11:13)] <- lapply(data1[, c(2,3,6,7,9,11:13)], factor)

str(data1)
```

```
## tibble [242 x 14] (S3: tbl_df/tbl/data.frame)
##  $ age     : num [1:242] 57 41 43 37 66 46 60 57 61 44 ...
##  $ sex     : Factor w/ 2 levels "0","1": 2 2 2 2 1 2 2 2 2 2 ...
##  $ cp      : Factor w/ 4 levels "0","1","2","3": 1 3 1 3 3 3 1 1 4 1 ...
##  $ trestbps: num [1:242] 165 130 120 130 146 150 145 110 134 110 ...
##  $ chol    : num [1:242] 289 214 177 250 278 231 282 335 234 197 ...
##  $ fbs     : Factor w/ 2 levels "0","1": 2 1 1 1 1 1 1 1 1 1 ...
##  $ restecg : Factor w/ 3 levels "0","1","2": 1 1 1 2 1 2 1 2 2 1 ...
##  $ thalach : num [1:242] 124 168 120 187 152 147 142 143 145 177 ...
##  $ exang   : Factor w/ 2 levels "0","1": 1 1 2 1 1 1 2 2 1 1 ...
##  $ oldpeak : num [1:242] 1 2 2.5 3.5 0 3.6 2.8 3 2.6 0 ...
##  $ slope   : Factor w/ 3 levels "0","1","2": 2 2 2 1 2 2 2 2 2 3 ...
##  $ ca      : Factor w/ 5 levels "0","1","2","3",..: 4 1 1 1 2 1 3 2 3 2 ...
##  $ thal    : Factor w/ 4 levels "0","1","2","3": 4 3 4 3 3 3 4 4 3 3 ...
##  $ target  : Factor w/ 2 levels "no","yes": 1 2 1 2 2 1 1 1 1 1 ...
```

2.  因子的处理,分类变量处理 onehot encodeing（**自由选择**）


```r
# 为了做lasso把y定义为numeric
data1$target <- as.numeric(data1$target)
# 分类变量转为factor
x.factors <- model.matrix(data1$target ~ data1$sex + data1$cp+data1$fbs+data1$restecg+data1$exang+data1$slope+data1$ca+data1$thal)[,-1]
```


```r
#生成自变量和因变量矩阵
x=as.matrix(data.frame(x.factors,data1[,c(1,4:5,8,10)]))
```


```r
#定义y
y=data1$target
```

-   进行拟合，默认为L1也就是Lasso


```r
fit1 <- glmnet(x,y,family="binomial")
#解释偏差百分比以及相应的λ值
print(fit1)
```

```
## 
## Call:  glmnet(x = x, y = y, family = "binomial") 
## 
##    Df %Dev Lambda
## 1   0  0.0 0.2410
## 2   1  2.9 0.2200
## 3   1  5.3 0.2000
## 4   3  8.3 0.1830
## 5   4 11.5 0.1660
## 6   4 14.5 0.1520
## 7   5 17.1 0.1380
## 8   5 19.4 0.1260
## 9   7 21.5 0.1150
## 10  7 23.5 0.1050
## 11  8 25.6 0.0952
## 12 10 28.0 0.0868
## 13 11 30.3 0.0791
## 14 11 32.3 0.0720
## 15 12 34.2 0.0656
## 16 12 36.1 0.0598
## 17 12 37.7 0.0545
## 18 12 39.2 0.0496
## 19 12 40.5 0.0452
## 20 14 41.6 0.0412
## 21 14 42.8 0.0376
## 22 14 43.8 0.0342
## 23 14 44.7 0.0312
## 24 14 45.6 0.0284
## 25 14 46.4 0.0259
## 26 14 47.1 0.0236
## 27 15 47.7 0.0215
## 28 16 48.3 0.0196
## 29 16 48.8 0.0178
## 30 16 49.3 0.0163
## 31 16 49.7 0.0148
## 32 17 50.0 0.0135
## 33 17 50.4 0.0123
## 34 18 50.7 0.0112
## 35 18 51.0 0.0102
## 36 19 51.2 0.0093
## 37 19 51.5 0.0085
## 38 19 51.8 0.0077
## 39 20 52.0 0.0070
## 40 20 52.2 0.0064
## 41 20 52.4 0.0058
## 42 20 52.5 0.0053
## 43 20 52.6 0.0049
## 44 20 52.8 0.0044
## 45 21 52.9 0.0040
## 46 21 52.9 0.0037
## 47 21 53.0 0.0033
## 48 21 53.1 0.0031
## 49 21 53.1 0.0028
## 50 21 53.2 0.0025
## 51 21 53.2 0.0023
## 52 21 53.2 0.0021
## 53 21 53.2 0.0019
## 54 21 53.3 0.0017
## 55 21 53.3 0.0016
## 56 21 53.3 0.0014
## 57 21 53.3 0.0013
## 58 21 53.3 0.0012
## 59 21 53.3 0.0011
## 60 21 53.3 0.0010
## 61 21 53.4 0.0009
## 62 21 53.4 0.0008
## 63 21 53.4 0.0008
## 64 21 53.4 0.0007
## 65 21 53.4 0.0006
## 66 21 53.4 0.0006
## 67 21 53.4 0.0005
## 68 21 53.4 0.0005
## 69 21 53.4 0.0004
## 70 21 53.4 0.0004
## 71 21 53.4 0.0004
## 72 21 53.4 0.0003
```

```r
#解释偏差不再随着λ值的增加而减小，因此而停止
```


```r
#画出收缩曲线图
plot(fit1,label = FALSE)
```

![](03-r_example_files/figure-latex/unnamed-chunk-13-1.pdf)<!-- --> 

```r
plot(fit1,label = TRUE,xvar = "lambda")#系数值如何随着λ的变化而变化
```

![](03-r_example_files/figure-latex/unnamed-chunk-13-2.pdf)<!-- --> 

```r
plot(fit1,label = TRUE,xvar = "dev")#偏差与系数之间的关系图
```

![](03-r_example_files/figure-latex/unnamed-chunk-13-3.pdf)<!-- --> 


```r
#指定lamda给出相应参数
lasso.coef <- predict(fit1, type = "coefficients",s = 0.040 )
lasso.coef
```

```
## 23 x 1 sparse Matrix of class "dgCMatrix"
##                        s1
## (Intercept)    -0.5051894
## data1.sex1     -0.4472455
## data1.cp1       .        
## data1.cp2       0.6233575
## data1.cp3       .        
## data1.fbs1      .        
## data1.restecg1  0.0148096
## data1.restecg2  .        
## data1.exang1   -0.6849315
## data1.slope1   -0.2724767
## data1.slope2    0.1970313
## data1.ca1      -0.9084992
## data1.ca2      -0.9740841
## data1.ca3      -0.6171345
## data1.ca4       .        
## data1.thal1     .        
## data1.thal2     0.1821188
## data1.thal3    -0.7726354
## age             .        
## trestbps       -0.0004619
## chol            .        
## thalach         0.0129917
## oldpeak        -0.2474481
```

glmnet包在使用cv.glmnet()估计λ值时，默认使用10折交叉验证。在K折交叉验证中，数据被划分成k个相同的子集（折），#每次使用k-1个子集拟合模型，然后使用剩下的那个子集做测试集，最后将k次拟合的结果综合起来（一般取平均数），确定最后的参数。

在这个方法中，每个子集只有一次用作测试集。在glmnet包中使用K折交叉验证非常容易，结果包括每次拟合的λ值和响应的MSE。默认设置为α=1。

-   进行交叉验证，选择最优的惩罚系数lambada


```r
set.seed(317)
cv.fit <- cv.glmnet(x,y,family="binomial")
#cv.fit <- cv.glmnet(x,y,family="binomial",type.measure = "auc")#AUC和λ的关系
#cv.fit <- cv.glmnet(x,y,family="binomial")#不要反复运行
plot(cv.fit)
```

![](03-r_example_files/figure-latex/unnamed-chunk-15-1.pdf)<!-- --> 


```r
#显示一个标准误的系数
coef(cv.fit, s = "lambda.1se")
```

```
## 23 x 1 sparse Matrix of class "dgCMatrix"
##                       s1
## (Intercept)    -0.570950
## data1.sex1     -0.429630
## data1.cp1       .       
## data1.cp2       0.610299
## data1.cp3       .       
## data1.fbs1      .       
## data1.restecg1  0.007149
## data1.restecg2  .       
## data1.exang1   -0.678984
## data1.slope1   -0.261868
## data1.slope2    0.194103
## data1.ca1      -0.883563
## data1.ca2      -0.944469
## data1.ca3      -0.586744
## data1.ca4       .       
## data1.thal1     .       
## data1.thal2     0.200315
## data1.thal3    -0.752414
## age             .       
## trestbps       -0.000139
## chol            .       
## thalach         0.012869
## oldpeak        -0.245773
```


```r
#选择交叉验证误差最小的lambda
cv.fit$lambda.min
```

```
## [1] 0.0123
```

```r
cv.fit$lambda.1se
```

```
## [1] 0.04122
```

```r
coef(cv.fit, s = "lambda.min")
```

```
## 23 x 1 sparse Matrix of class "dgCMatrix"
##                       s1
## (Intercept)     1.718543
## data1.sex1     -1.058682
## data1.cp1       0.171712
## data1.cp2       1.200687
## data1.cp3       0.727069
## data1.fbs1      .       
## data1.restecg1  0.204794
## data1.restecg2  .       
## data1.exang1   -0.776428
## data1.slope1   -0.609627
## data1.slope2    0.276892
## data1.ca1      -1.635310
## data1.ca2      -1.885126
## data1.ca3      -1.390527
## data1.ca4       0.131830
## data1.thal1     .       
## data1.thal2     .       
## data1.thal3    -1.084248
## age             .       
## trestbps       -0.011442
## chol           -0.001332
## thalach         0.015735
## oldpeak        -0.324021
```


```r
#画出收缩系数图，及最小的lambda曲线
plot(cv.fit$glmnet.fit,xvar="lambda")
abline(v=log(c(cv.fit$lambda.min,cv.fit$lambda.1se)),lty=2)
```

![](03-r_example_files/figure-latex/unnamed-chunk-18-1.pdf)<!-- --> 


```r
#系数不为0的结果为Lasso筛选后的值
predict(cv.fit,type='coefficient',s=cv.fit$lambda.min)
```

```
## 23 x 1 sparse Matrix of class "dgCMatrix"
##                       s1
## (Intercept)     1.718543
## data1.sex1     -1.058682
## data1.cp1       0.171712
## data1.cp2       1.200687
## data1.cp3       0.727069
## data1.fbs1      .       
## data1.restecg1  0.204794
## data1.restecg2  .       
## data1.exang1   -0.776428
## data1.slope1   -0.609627
## data1.slope2    0.276892
## data1.ca1      -1.635310
## data1.ca2      -1.885126
## data1.ca3      -1.390527
## data1.ca4       0.131830
## data1.thal1     .       
## data1.thal2     .       
## data1.thal3    -1.084248
## age             .       
## trestbps       -0.011442
## chol           -0.001332
## thalach         0.015735
## oldpeak        -0.324021
```

```r
predict(cv.fit,type='coefficient',s=cv.fit$lambda.1se)
```

```
## 23 x 1 sparse Matrix of class "dgCMatrix"
##                       s1
## (Intercept)    -0.570950
## data1.sex1     -0.429630
## data1.cp1       .       
## data1.cp2       0.610299
## data1.cp3       .       
## data1.fbs1      .       
## data1.restecg1  0.007149
## data1.restecg2  .       
## data1.exang1   -0.678984
## data1.slope1   -0.261868
## data1.slope2    0.194103
## data1.ca1      -0.883563
## data1.ca2      -0.944469
## data1.ca3      -0.586744
## data1.ca4       .       
## data1.thal1     .       
## data1.thal2     0.200315
## data1.thal3    -0.752414
## age             .       
## trestbps       -0.000139
## chol            .       
## thalach         0.012869
## oldpeak        -0.245773
```

```r
predict(cv.fit,type='coefficient',s=0.1)
```

```
## 23 x 1 sparse Matrix of class "dgCMatrix"
##                       s1
## (Intercept)    -1.129256
## data1.sex1      .       
## data1.cp1       .       
## data1.cp2       0.135400
## data1.cp3       .       
## data1.fbs1      .       
## data1.restecg1  .       
## data1.restecg2  .       
## data1.exang1   -0.454297
## data1.slope1    .       
## data1.slope2    0.052755
## data1.ca1      -0.036494
## data1.ca2       .       
## data1.ca3       .       
## data1.ca4       .       
## data1.thal1     .       
## data1.thal2     0.614504
## data1.thal3    -0.220414
## age             .       
## trestbps        .       
## chol            .       
## thalach         0.008586
## oldpeak        -0.146561
```

#### 测试集\\验证集中 验证


```r
library(rms)
#导入验证集
test <- testset
str(test)
```

```
## tibble [61 x 14] (S3: tbl_df/tbl/data.frame)
##  $ age     : num [1:61] 63 56 44 64 58 58 57 61 60 54 ...
##  $ sex     : num [1:61] 1 0 1 1 0 1 0 0 1 1 ...
##  $ cp      : num [1:61] 3 1 1 3 2 2 0 0 0 0 ...
##  $ trestbps: num [1:61] 145 140 120 110 120 132 120 130 130 124 ...
##  $ chol    : num [1:61] 233 294 263 211 340 224 354 330 253 266 ...
##  $ fbs     : num [1:61] 1 0 0 0 0 0 0 0 0 0 ...
##  $ restecg : num [1:61] 0 0 1 0 1 0 1 0 1 0 ...
##  $ thalach : num [1:61] 150 153 173 144 172 173 163 169 144 109 ...
##  $ exang   : num [1:61] 0 0 0 1 0 0 1 0 1 1 ...
##  $ oldpeak : num [1:61] 2.3 1.3 0 1.8 0 3.2 0.6 0 1.4 2.2 ...
##  $ slope   : num [1:61] 0 1 2 1 2 2 2 2 2 1 ...
##  $ ca      : num [1:61] 0 0 0 0 0 2 0 0 1 1 ...
##  $ thal    : num [1:61] 1 2 3 2 2 3 2 2 3 3 ...
##  $ target  : Factor w/ 2 levels "no","yes": 2 2 2 2 2 1 2 1 1 1 ...
```

```r
test$target <- as.numeric(test$target)
#制作x矩阵
test[,c(2,3,6,7,9,11:13)] <- lapply(test[, c(2,3,6,7,9,11:13)], factor)

x.factors_test <- model.matrix(test$target ~ test$sex + test$cp+test$fbs+test$restecg+test$exang+test$slope+test$ca+test$thal)[,-1]

newx=as.matrix(data.frame(x.factors_test,test[,c(1,4:5,8,10)]))
```

-   利用lasso在测试集中保存预测值，验证集当中的预测值


```r
test.y <- predict(cv.fit, newx = newx, type = "response", s=cv.fit$lambda.1se)
library(rms)
#在R当中做calibration plot
test <- data.frame(test,test.y)
# write.csv(test,file="test0.csv")
# str(test)
##########val.prob#####
val.prob(test$s1,test$target)
```

![](03-r_example_files/figure-latex/unnamed-chunk-21-1.pdf)<!-- --> 

```
##        Dxy    C (ROC)         R2          D   D:Chi-sq 
##  9.143e-01  9.571e-01  6.671e-01  6.701e-01  4.187e+01 
##        D:p          U   U:Chi-sq        U:p          Q 
##         NA -3.602e-01 -1.997e+01  1.000e+00  1.030e+00 
##      Brier  Intercept      Slope       Emax        E90 
##  1.134e+00 -6.258e-02  2.432e+00  1.180e+00  1.168e+00 
##       Eavg        S:z        S:p 
##  1.016e+00 -6.885e+00  5.791e-12
```

### 训练集建立模型

-   训练Logistic


```r
fitControl = trainControl(method = "none", classProbs = TRUE)

set.seed(10101) # 设置种子数，复现结果
LR_model <- train(target~. ,
                  data = trainset,
                  method = "glm",
                  trControl = fitControl,
                  metric = "ROC")
```

-   训练随机森林


```r
set.seed(10101) # 设置种子数，复现结果
RF_model <- train(target~. ,
                  data = trainset,
                  method = "parRF",
                  trControl = fitControl,
                  metric = "ROC")
```

-   训练SVM


```r
set.seed(10101) # 设置种子数，复现结果
SVM_model <- train(target~. ,
                  data = trainset,
                  method = "svmRadial",
                  trControl = fitControl,
                  metric = "ROC")
```

### 测试集评价模型

-   获得模型测试集风险概率


```r
LR_pro <- predict (LR_model, newdata =testset, type= "prob") 
RF_pro <- predict (RF_model, newdata =testset, type= "prob") 
SVM_pro <-  predict (SVM_model, newdata =testset, type= "prob")

testset$LR <- LR_pro$yes
testset$RF <- RF_pro$yes
testset$SVM <- SVM_pro$yes
```

-   使用plotROC包和ggplot2绘制ROC曲线


```r
ROC <- melt_roc(testset, "target", c("LR", "RF", "SVM"))
ROC_plot <- ggplot(ROC, aes(d = D.target, m = M, color = name)) +
    geom_roc(n.cuts = 0) +
    labs(title = "三种模型测试集ROC曲线") + 
    theme(plot.title = element_text(hjust = 0.5))+
    geom_abline()
# theme(text = element_text(size = 50)) 设置所有字体大小
```


```r
ROC_plot
```

![](03-r_example_files/figure-latex/unnamed-chunk-27-1.pdf)<!-- --> 

-   pROC包的roc()和auc()函数，计算AUC


```r
roc_LR <- roc(testset$target, testset$LR)
auc_LR <- auc(roc_LR)
auc_LR   # Area under the curve: 0.956
```

```
## Area under the curve: 0.956
```

```r
roc_RF <- roc(testset$target, testset$RF)
auc_RF <- auc(roc_RF)
auc_RF   # Area under the curve: 0.935
```

```
## Area under the curve: 0.935
```

```r
roc_SVM <- roc(testset$target, testset$SVM)
auc_SVM <- auc(roc_SVM)
auc_SVM  # Area under the curve: 0.969
```

```
## Area under the curve: 0.969
```

### Logistic DCA曲线

<div>

> DCA曲线横坐标是判断恶性/良性的风险阈值（0\~1），纵坐标为不同阈值对应的临床净获益（net benifit）。主要比较了根据四种模型划分恶性/良性患者（针对性干预），相比于把所有患者都看作恶性实施干预（ALL曲线）和所有患者都不干预（None曲线），是否有临床净获益。
>
> 006年首次介绍了DCA曲线，并提供了使用R语言绘制DCA曲线的dca()函代码下载链接：<https://www.mskcc.org/departments/epidemiology-biostatistics/biostatistics/decision-curve-analysis>

</div>


```r
testset$target <- ifelse(testset$target == "yes", 1, 0)
# write_csv(testset, file = "data/testset.csv")
```


```r
source("dca/dca.r")
testset1 <- read.csv("data/testset.csv")
DCA <- dca(data = testset1, outcome = "target",
           predictors = c("LR"))
```

![](03-r_example_files/figure-latex/unnamed-chunk-30-1.pdf)<!-- --> 

能够看到，当风险阈值范围在0\~1 时，模型的临床净获益均高于ALL曲线和None曲线，能够取得临床净获益的阈值范围还是比较大的，但应该注意的是，随着阈值增大，模型的临床净获益也在减小。

### Logistic的Nomogram绘制


```r
# 解决"variable 变量名 does not have limits defined by datadist"
dd <- datadist(testset)
options(datadist="dd")

# 设置变量标签
testset$sex <- factor(testset$sex,levels=c(0,1),labels=c("Female","Male"))
# 构建LR模型
f1 <- lrm(target~ age+sex+chol+oldpeak, data= testset, x = T, y = T)
```


```r
# 绘制Nomogram
nom <- nomogram(f1,
                fun=function(x)1/(1+exp(-x)),
                #也可fun=plogis,fun.at概率坐标范围
                fun.at=c(.001,.01,.05,seq(.1,.9,by=.1),.95,.99,.999),
                funlabel="Risk of target",
                conf.int=F,
                abbrev=F,
                minlength=1,
                #lp线性预测值
                lp=F)
plot(nom)
```

![](03-r_example_files/figure-latex/unnamed-chunk-32-1.pdf)<!-- --> 

-   另外一种


```r
library(regplot)
regplot(f1)
```

```
## [1] "note: points tables not constructed unless points=TRUE "
```


\includegraphics[width=1\linewidth]{images/0301} 
