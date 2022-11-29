---
title: "临床预测模型"
author: "杨 弘"
date: "2022-11-30"
output:
  html_document: default
  pdf_document: default
documentclass: book
# bibliography: [book.bib, packages.bib]
description: "了解临床预测模型相关概念与构建流程"
site: bookdown::bookdown_site
biblio-style: apalike
csl: chicago-fullnote-bibliography.csl
---



# 关于课程 {.unnumbered}

## **课程目标** {.unnumbered}

-   了解临床预测模型相关概念与构建流程

-   了解Python与R语言基本使用与案例实操

-   了解MIMIC与NHANES公共数据库

## **课程大纲** {.unnumbered}

### **案例教学** {.unnumbered}

1.  **临床预测模型概述**
    -   应用场景

    -   模型构建流程

    -   模型报告

    -   传统统计与机器学习
2.  **案例介绍**
    -   案例背景

    -   数据描述

    -   分析思路

    -   代码预览

### **实操教学** {.unnumbered}

1.  **Anaconda安装与包的安装**

    -   如何打开.ipynb文件

    -   Anaconda Prompt 基本命令

    -   使用**conda**或**pip**安装包

2.  **Python入门基**础

    -   数据读取

    -   模型构建等基础语法

    -   案例实操：代码与对应结果详细解释

        -   数据描述

        -   数据预处理

        -   模型构建

3.  **R安装与入门**

    -   数据读取

    -   模型构建等基础语法

    -   案例实操：代码与对应结果详细解释

4.  **NHANES**数据库简介、数据提取

5.  **MIMIC IV 数据库**简介、申请、安装、提取

## **上课要求** {.unnumbered}

-   自带电脑（课程以windows系统为例）

-   软件安装、代码运行实操

## **如何提问** {.unnumbered}

-   鼓励先自己动手查询（[baidu](www.baidu.com)、[biying](https://cn.bing.com/)、[github](https://github.com/)、[CSDN](https://www.csdn.net/)）

-   想获得快速帮助，请描述以下内容（**问题截全图**）

    -   想解决的问题是什么？

    -   代码是什么？

    -   报错信息是什么？

## **课程用到的软件** {.unnumbered}

> Anaconda
>
> R、Rstudio
>
> Postgresql、Navicat、7z

## **需要安装的包** {.unnumbered}

-   Python


```python
pip install scikit-learn
pip install pandas_profiling
pip install matplotlib
```

**或者缺少什么包安装什么包**


```python
conda install 包名
```

-   R


```r
my_packages <- 
   c("tidyverse", "dlookr", "plotROC", "pROC", "e1071", "caret",
     "ggplot2", "rms", "regplot","nhanesA", "haven","glmnet")
```


```r
install.packages(my_packages, dependencies = T)

for (pkg in my_packages)
{
  if (!require(pkg, character.only = TRUE)) {
    install.packages(pkg)
  }
}
```

**或者缺少什么包安装什么包**


```r
install.packages("包名")
```
