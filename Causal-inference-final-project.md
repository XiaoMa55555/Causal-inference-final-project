Average Treatment Effect(ATE) in multi-level treatment
================
Xiao Ma

``` r
library(Hmisc, quietly = TRUE)
library(tidyverse, quietly = TRUE)
library(nnet, quietly = TRUE)
library(rms, quietly = TRUE)
getHdata(plasma)
plasma<-plasma %>%
  mutate(Vituse=vituse,
         vituse=fct_recode(vituse,
                           "0"="No",
                           "1"="Yes, not often",
                           "2"="Yes, fairly often"))
```

``` r
#Unjusted method
Unjusted<-plasma %>%
  select(vituse,betaplasma) %>% 
  group_by(vituse) %>% 
  summarise(mean=mean(betaplasma), 
  sd=sd(betaplasma), 
  n=n()) 
#ATE
ATE_unadj_01<-Unjusted$mean[Unjusted$vituse==1]-Unjusted$mean[Unjusted$vituse==0]
ATE_unadj_02<-Unjusted$mean[Unjusted$vituse==2]-Unjusted$mean[Unjusted$vituse==0]

#SE
se_unadj_01<-sqrt(Unjusted$sd[Unjusted$vituse==1]^2/Unjusted$n[Unjusted$vituse==1]+Unjusted$sd[Unjusted$vituse==0]^2/Unjusted$n[Unjusted$vituse==0]) 
se_unadj_02<-sqrt(Unjusted$sd[Unjusted$vituse==2]^2/Unjusted$n[Unjusted$vituse==2]+Unjusted$sd[Unjusted$vituse==0]^2/Unjusted$n[Unjusted$vituse==0])
```

``` r
#Regression adjusted
#Fit regression model
m1 <- lm(betaplasma ~ age + sex + smokstat + quetelet + vituse + calories + fat + fiber + alcohol + cholesterol + betadiet + retdiet, data = plasma)
plasma0 <- plasma1 <- plasma2 <- plasma
plasma0$vituse = factor(0)
plasma1$vituse = factor(1)
plasma2$vituse = factor(2)
pred0 <- predict(m1, newdata = plasma0, type = "response")
pred1 <- predict(m1, newdata = plasma1, type = "response")
pred2 <- predict(m1, newdata = plasma2, type = "response")
ATE_adj_01 <- mean(pred1 - pred0)
ATE_adj_02 <- mean(pred2 - pred0)

#Bootstrap for calculating se
set.seed(7485)
B <- 100
ATE_adj_01.boot <- NULL
ATE_adj_02.boot <- NULL
n <- nrow(plasma)
for(i in 1:B) {
plasma.boot <- plasma[sample(1:n, n, replace = TRUE), ]
m1.boot <- lm(betaplasma ~ age + sex + smokstat + quetelet + vituse + calories + fat + fiber + alcohol + cholesterol + betadiet + retdiet, data = plasma.boot)
data0.boot <- data1.boot <- data2.boot <- plasma.boot
data0.boot$vituse = factor(0)
data1.boot$vituse = factor(1)
data2.boot$vituse = factor(2)
pred0.boot <- predict(m1.boot, newdata = data0.boot, type = "response")
pred1.boot <- predict(m1.boot, newdata = data1.boot, type = "response")
pred2.boot <- predict(m1.boot, newdata = data2.boot, type = "response")
ATE_adj_01.boot <- c(ATE_adj_01.boot, mean(pred1.boot - pred0.boot))
ATE_adj_02.boot <- c(ATE_adj_02.boot, mean(pred2.boot - pred0.boot))
}

#Result for Regression adjusted method
se_adj_01 <- sd(ATE_adj_01.boot)
se_adj_02 <- sd(ATE_adj_02.boot)
```

``` r
#PS Regression Adjustment
#Propensity Model, use multinomial logistic regression model
ps_predict<-multinom(factor(vituse) ~ age + sex + smokstat + quetelet + calories + fat + fiber + alcohol + cholesterol + betadiet + retdiet, data = plasma)

#Obtain Estimated Propensity Scores
ps<-data.frame(fitted(ps_predict))
ps0<-fitted(ps_predict)[,1]
ps1<-fitted(ps_predict)[,2]
ps2<-fitted(ps_predict)[,3]
names(ps)[1]<-paste("ps0")
names(ps)[2]<-paste("ps1")
names(ps)[3]<-paste("ps2")
plasma<-cbind(plasma,ps)

#Include in Outcome Model
m1.ps0 <- lm(betaplasma ~ vituse*rcs(ps0, 5), data = plasma)
m1.ps1 <- lm(betaplasma ~ vituse*rcs(ps1, 5), data = plasma)
m1.ps2 <- lm(betaplasma ~ vituse*rcs(ps2, 5), data = plasma)

#PS Regression Adjustment
data0 <- data1 <- data2 <- plasma
data0$vituse = factor(0)
data1$vituse = factor(1)
data2$vituse = factor(2)

#Adjusted outcome value
pred0.ps <- predict(m1.ps0, newdata = data0, type = "response")
pred1.ps <- predict(m1.ps1, newdata = data1, type = "response")
pred2.ps <- predict(m1.ps2, newdata = data2, type = "response")

#Adjusted outcome value
ATE_ps_01<-mean(pred1.ps-pred0.ps)
ATE_ps_02<-mean(pred2.ps-pred0.ps)

#Bootstrap for calculating se
set.seed(7485)
B <- 100
ATE_ps_01.boot <- NULL
ATE_ps_02.boot <- NULL
n <- nrow(plasma)
for(i in 1:B) {
plasma.boot <- plasma[sample(1:n, n, replace = TRUE), ]
m1.ps0.boot <- lm(betaplasma ~ vituse*rcs(ps0, 5), data = plasma.boot)
m1.ps1.boot <- lm(betaplasma ~ vituse*rcs(ps1, 5), data = plasma.boot)
m1.ps2.boot <- lm(betaplasma ~ vituse*rcs(ps2, 5), data = plasma.boot)
data0.boot <- data1.boot <- data2.boot <- plasma.boot
data0.boot$vituse = factor(0)
data1.boot$vituse = factor(1)
data2.boot$vituse = factor(2)
pred0.boot <- predict(m1.ps0.boot, newdata = data0.boot, type = "response")
pred1.boot <- predict(m1.ps1.boot, newdata = data1.boot, type = "response")
pred2.boot <- predict(m1.ps2.boot, newdata = data2.boot, type = "response")
ATE_ps_01.boot <- c(ATE_ps_01.boot, mean(pred1.boot - pred0.boot))
ATE_ps_02.boot <- c(ATE_ps_02.boot, mean(pred2.boot - pred0.boot))}

#Result for PS Regression adjusted method
se_ps_01 <- sd(ATE_ps_01.boot)
se_ps_02 <- sd(ATE_ps_02.boot)
```

``` r
#Propensity Scores Stratification
#Divide Propensity Scores Into Quintiles
ps0_quintile <- cut(plasma$ps0,
breaks = c(0, quantile(plasma$ps0, p = c(0.2, 0.4, 0.6, 0.8)), 1), labels = 1:5)
ps1_quintile <- cut(plasma$ps1,
breaks = c(0, quantile(plasma$ps1, p = c(0.2, 0.4, 0.6, 0.8)), 1), labels = 1:5)
ps2_quintile <- cut(plasma$ps2,
breaks = c(0, quantile(plasma$ps2, p = c(0.2, 0.4, 0.6, 0.8)), 1), labels = 1:5)

#Propensity Scores Stratification
n <- nrow(plasma)
nj <- table(ps0_quintile)
te_quintile01 <- tapply(plasma$betaplasma[plasma$vituse == 1], ps1_quintile[plasma$vituse == 1], mean) - tapply(plasma$betaplasma[plasma$vituse == 0], ps0_quintile[plasma$vituse == 0], mean)
te_quintile02 <- tapply(plasma$betaplasma[plasma$vituse == 2], ps2_quintile[plasma$vituse == 2], mean) - tapply(plasma$betaplasma[plasma$vituse == 0], ps0_quintile[plasma$vituse == 0], mean)
ATE_PSS_01 <- sum(te_quintile01 *nj/n)
ATE_PSS_02 <- sum(te_quintile02 *nj/n)

#Bootstrap for PS Stratification
set.seed(7485)
B <- 100
ATE_PSS_01.boot <- NULL
ATE_PSS_02.boot <- NULL
n <- nrow(plasma)
for(i in 1:B) {
plasma.boot <- plasma[sample(1:n, n, replace = TRUE), ]
ps_predict.boot <- multinom(vituse ~ age + sex + smokstat + quetelet + calories + fat + fiber + alcohol + cholesterol + betadiet + retdiet, data = plasma.boot)
ps0.boot <- fitted(ps_predict.boot)[,1]
ps1.boot <- fitted(ps_predict.boot)[,2]
ps2.boot <- fitted(ps_predict.boot)[,3]
ps0_quintile.boot <- cut(ps0.boot,
breaks = c(0, quantile(ps0.boot, p = c(0.2, 0.4, 0.6, 0.8)), 1), labels = 1:5)
ps1_quintile.boot <- cut(ps1.boot,
breaks = c(0, quantile(ps1.boot, p = c(0.2, 0.4, 0.6, 0.8)), 1), labels = 1:5)
ps2_quintile.boot <- cut(ps2.boot,
breaks = c(0, quantile(ps2.boot, p = c(0.2, 0.4, 0.6, 0.8)), 1), labels = 1:5)
nj0.boot <- table(ps0_quintile.boot)
nj1.boot <- table(ps1_quintile.boot)
nj2.boot <- table(ps2_quintile.boot)
te_quintile01.boot <- tapply(plasma.boot$betaplasma[plasma.boot$vituse == 1], ps1_quintile.boot[plasma.boot$vituse == 1], mean) - tapply(plasma.boot$betaplasma[plasma.boot$vituse == 0], ps0_quintile.boot[plasma.boot$vituse == 0], mean)
te_quintile02.boot <- tapply(plasma.boot$betaplasma[plasma.boot$vituse == 2], ps2_quintile.boot[plasma.boot$vituse == 2], mean) - tapply(plasma.boot$betaplasma[plasma.boot$vituse == 0], ps0_quintile.boot[plasma.boot$vituse == 0], mean)
ATE_PSS_01.boot <- (te_quintile01.boot *nj1.boot/n+te_quintile01.boot *nj0.boot/n)/2
ATE_PSS_02.boot <- (te_quintile02.boot *nj2.boot/n+te_quintile02.boot *nj0.boot/n)/2
}

#Result of PSS
se_PSS_01 <- sd(ATE_PSS_01.boot)
se_PSS_02 <- sd(ATE_PSS_02.boot)
```

``` r
#Inverse Probability Weighting (IPW2)
#Create dummy indicators for each of the treatment
plasma$dummy0<-ifelse(plasma$vituse==0,1,0)
plasma$dummy1<-ifelse(plasma$vituse==1,1,0)
plasma$dummy2<-ifelse(plasma$vituse==2,1,0)

#Obtain the estimated propensity score
ps_0<-glm(dummy0 ~ age + sex + smokstat + quetelet + calories + fat + fiber + alcohol + cholesterol + betadiet + retdiet, data = plasma)
plasma$ps_dummy0<-predict(ps_0, type = "response", family = "binomial")
ps_1<-glm(dummy1 ~ age + sex + smokstat + quetelet + calories + fat + fiber + alcohol + cholesterol + betadiet + retdiet, data = plasma, family = "binomial")
plasma$ps_dummy1<-predict(ps_1, type = "response", family = "binomial")
ps_2<-glm(dummy2 ~ age + sex + smokstat + quetelet + calories + fat + fiber + alcohol + cholesterol + betadiet + retdiet, data = plasma)
plasma$ps_dummy2<-predict(ps_2, type = "response", family = "binomial")

#Use the estimated propensity scores to compute the ATE weights
plasma$w0 <- plasma$dummy0/plasma$ps_dummy0
plasma$w1 <- plasma$dummy1/plasma$ps_dummy1
plasma$w2 <- plasma$dummy2/plasma$ps_dummy2

#IPW2
ATE_IPW2_01 <- weighted.mean(plasma$betaplasma, plasma$w1) - weighted.mean(plasma$betaplasma, plasma$w0)
ATE_IPW2_02 <- weighted.mean(plasma$betaplasma, plasma$w2) - weighted.mean(plasma$betaplasma, plasma$w0)

#Bootstrap for IPW2 se
set.seed(7485)
B <- 100
ATE_IPW2_01.boot <- NULL
ATE_IPW2_02.boot <- NULL
n <- nrow(plasma)
for(i in 1:B) {
plasma.boot <- plasma[sample(1:n, n, replace = TRUE), ]
ps_0.boot<-glm(dummy0 ~ age + sex + smokstat + quetelet + calories + fat + fiber + alcohol + cholesterol + betadiet + retdiet, data = plasma.boot)
ps_1.boot<-glm(dummy1 ~ age + sex + smokstat + quetelet + calories + fat + fiber + alcohol + cholesterol + betadiet + retdiet, data = plasma.boot)
ps_2.boot<-glm(dummy2 ~ age + sex + smokstat + quetelet + calories + fat + fiber + alcohol + cholesterol + betadiet + retdiet, data = plasma.boot)
ps_dummy0.boot <- predict(ps_0.boot, type = "response", family = "binomial")
ps_dummy1.boot <- predict(ps_1.boot, type = "response", family = "binomial")
ps_dummy2.boot <- predict(ps_2.boot, type = "response", family = "binomial")
w0.boot<-plasma.boot$dummy0/ps_dummy0.boot
w1.boot<-plasma.boot$dummy1/ps_dummy1.boot
w2.boot<-plasma.boot$dummy2/ps_dummy2.boot
ATE_IPW2_01.boot <- c(ATE_IPW2_01.boot, weighted.mean(plasma.boot$betaplasma, plasma.boot$w1) - weighted.mean(plasma.boot$betaplasma, plasma.boot$w0))
ATE_IPW2_02.boot <- c(ATE_IPW2_02.boot, weighted.mean(plasma.boot$betaplasma, plasma.boot$w2) - weighted.mean(plasma.boot$betaplasma, plasma.boot$w0))
}

#Result of IPW2
se_IPW2_01 <- sd(ATE_IPW2_01.boot)
se_IPW2_02 <- sd(ATE_IPW2_02.boot)
```

``` r
#Final results for ATE using different methods
data.1<-data.frame(Method=c("Unadjusted","Regression adjusted",
                            "Propensity Scores regression adjusted",
                            "Propensity Scores Stratification",
                            "Inverse Probability Weighting"),
                   "ATE between not often and never"=c(
                     paste(round(ATE_unadj_01,digits = 2),"(",round(ATE_unadj_01-qnorm(0.975)*se_unadj_01,digits = 2),",",round(ATE_unadj_01+qnorm(0.975)*se_unadj_01,digits = 2),")"),
                     paste(round(ATE_adj_01,digits = 2),"(",round(ATE_adj_01-qnorm(0.975)*se_adj_01,digits = 2),",",round(ATE_adj_01+qnorm(0.975)*se_adj_01,digits = 2),")"),
                     paste(round(ATE_ps_01,digits = 2),"(",round(ATE_ps_01-qnorm(0.975)*se_ps_01,digits = 2),",",round(ATE_ps_01+qnorm(0.975)*se_ps_01,digits = 2),")"),
                     paste(round(ATE_PSS_01,digits = 2),"(",round(ATE_PSS_01-qnorm(0.975)*se_PSS_01,digits = 2),",",round(ATE_PSS_01+qnorm(0.975)*se_PSS_01,digits = 2),")"),
                     paste(round(ATE_IPW2_01,digits = 2),"(",round(ATE_IPW2_01-qnorm(0.975)*se_IPW2_01,digits = 2),",",round(ATE_IPW2_01+qnorm(0.975)*se_IPW2_01,digits = 2),")")),
                   
                   "ATE between fairly often and never"=c(
                     paste(round(ATE_unadj_02,digits = 2),"(",round(ATE_unadj_02-qnorm(0.975)*se_unadj_02,digits = 2),",",round(ATE_unadj_02+qnorm(0.975)*se_unadj_02,digits = 2),")"),
                     paste(round(ATE_adj_02,digits = 2),"(",round(ATE_adj_02-qnorm(0.975)*se_adj_02,digits = 2),",",round(ATE_adj_02+qnorm(0.975)*se_adj_02,digits = 2),")"),
                     paste(round(ATE_ps_02,digits = 2),"(",round(ATE_ps_02-qnorm(0.975)*se_ps_02,digits = 2),",",round(ATE_ps_02+qnorm(0.975)*se_ps_02,digits = 2),")"),
                     paste(round(ATE_PSS_02,digits = 2),"(",round(ATE_PSS_02-qnorm(0.975)*se_PSS_02,digits = 2),",",round(ATE_PSS_02+qnorm(0.975)*se_PSS_02,digits = 2),")"),
                     paste(round(ATE_IPW2_02,digits = 2),"(",round(ATE_IPW2_02-qnorm(0.975)*se_IPW2_02,digits = 2),",",round(ATE_IPW2_02+qnorm(0.975)*se_IPW2_02,digits = 2),")")))

knitr::kable(data.1)
```

| Method                                | ATE.between.not.often.and.never | ATE.between.fairly.often.and.never |
|:--------------------------------------|:--------------------------------|:-----------------------------------|
| Unadjusted                            | 48.77 ( 13.17 , 84.36 )         | 104.07 ( 57.32 , 150.81 )          |
| Regression adjusted                   | 44 ( 6.49 , 81.51 )             | 78.28 ( 38.27 , 118.29 )           |
| Propensity Scores regression adjusted | 50.03 ( -3.04 , 103.09 )        | 76.82 ( 36.3 , 117.35 )            |
| Propensity Scores Stratification      | 52.14 ( 20.78 , 83.49 )         | 77.73 ( 34.96 , 120.5 )            |
| Inverse Probability Weighting         | 55.94 ( 10.87 , 101.02 )        | 44.54 ( -16.97 , 106.05 )          |
