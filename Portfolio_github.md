# portfolio.github
Catherine Pilarz  
February 6, 2017  






```r
# Portfolio Optimization

# Calculation Sharpe ratio for various ETFs
data.x = read.csv("asset_data.txt")

#Change date column from string to date
data.x$date <- as.Date(data.x$date, format="%Y-%m-%d")

#only include dates where federal interest rate is available
data.x <- data.x[!is.na(data.x$fed.rate),]
#rename rows
rownames(data.x) <- 1:nrow(data.x)

plot(data.x$date, data.x$fed.rate, type="l", main="Time Series Plot of Federal Interest Rate",
     xlab="Date", ylab="Federal Funds Interest Rate [%]")
```

![](Portfolio_github_files/figure-html/cars-1.png)<!-- -->

```r
#What is the start and end date for this dataset?
data.x$date[1]
```

```
## [1] "2003-01-08"
```

```r
data.x$date[nrow(data.x)]
```

```
## [1] "2014-10-29"
```

```r
#Make training and testing datasets
train <- data.x[data.x$date < "2014-01-01",]
test <- data.x[data.x$date >= "2014-01-01",]

#How many observations are in each?
nrow(train)
```

```
## [1] 570
```

```r
nrow(test)
```

```
## [1] 43
```

```r
#Convert federal interest rate to a percentage
perc <- function(x) x/100
data.x$fed.rate <- sapply(data.x$fed.rate, perc)

#federal interest rate to percent in training set
train$fed.p <- sapply(train$fed.rate, perc)

#function to calculate returns
returns <- function(x) {
  
  rt <- rep(NA, length(x)) #create empty vector of length x
  for (i in 2:length(x)){ #First value is NA, you can't calculate returns from single value
    rt[i] <- (x[i] - x[i-1]) / x[i-1]
  }
  return(rt)
}

#Calculate returns for training set
train$ret.spy <- returns(train$close.spy)
train$ret.tlt <- returns(train$close.tlt)

#Plot returns
par(mfrow=c(2,1))
plot(train$date, train$ret.spy, type="l", main="Time Series Plot of S&P Returns",
     xlab="Date", ylab="Returns [%]", ylim = c(-.15,.1))
abline(h=0, lty=2)
plot(train$date, train$ret.tlt, type="l", main="Time Series Plot of Treasury ETF Returns",
     xlab="Date", ylab="Returns [%]", ylim=c(-.15,.1))
abline(h=0, lty=2)
```

![](Portfolio_github_files/figure-html/cars-2.png)<!-- -->

```r
#Are the returns normally distributed?
qqnorm(train$ret.spy, las=TRUE, main="Normal Q-Q Plot for S&P Returns", pch=20)
qqline(train$ret.spy)

qqnorm(train$ret.tlt, las=TRUE, main="Normal Q-Q Plot for Treasury ETF Returns", pch=20)
qqline(train$ret.tlt)
```

![](Portfolio_github_files/figure-html/cars-3.png)<!-- -->

```r
#what is the correlation between the two ETFs?
cor(train$ret.spy, train$ret.tlt, use="complete.obs")
```

```
## [1] -0.3439013
```

```r
#calculate rolling returns over 24 week window
#first rolling window
returns(data.x[1:24,]$close.spy)
```

```
##  [1]           NA  0.011051537 -0.045779221 -0.019167517 -0.018848289
##  [6] -0.032410136  0.037515225 -0.022775299  0.002522826 -0.028639904
## [11]  0.085122132 -0.010004548  0.011139182 -0.011584327  0.014018155
## [16]  0.044532578 -0.002929052  0.016102709  0.011992719 -0.019680457
## [21]  0.032595791  0.036479565  0.011496571  0.012662014
```

```r
#compute all rolling returns
roll.spy = rep(NA, 24)
roll.tlt = rep(NA, 24)
roll.cor = rep(NA, 590)
for (i in 1:(nrow(data.x)-23)){

  roll.spy <- returns(data.x[i:(i+24),]$close.spy)
  roll.tlt <- returns(data.x[i:(i+24),]$close.tlt)
  roll.cor[i] <- cor(roll.spy, roll.tlt, use="complete.obs")
}
  
#plot rolling returns
plot(data.x[24:nrow(data.x),]$date, roll.cor, xlab="Date", ylab="Correlation", type="l",
     main="Rolling 24 week correlation of \n S&P and Treasury ETFs")
abline(h=0, lty=2, col="gray")
  
#calculating sharpe ratios
#excess returns for each week
excess.spy <- rep(NA, nrow(train))
excess.tlt <- rep(NA, nrow(train))

for (i in 2:nrow(train)){
  excess.spy[i] <- train$ret.spy[i] - train$fed.p[i-1] / 52 
  excess.tlt[i] <- train$ret.tlt[i] - train$fed.p[i-1] / 52
}

#excess returns index
eindex.spy <- rep(NA, nrow(train))
eindex.spy[1] <- 100
eindex.tlt <- rep(NA, nrow(train))
eindex.tlt[1] <- 100

for (i in 2:nrow(train)){
  eindex.spy[i] <- eindex.spy[i-1] * (1+excess.spy[i])
  eindex.tlt[i] <- eindex.tlt[i-1] * (1+excess.tlt[i])
}
  
#number of years of data:
nyear <- (nrow(train)-1) / 52
#compounded annual growth rate
cagr.spy <- (eindex.spy[length(eindex.spy)] / eindex.spy[1]) ^ (1/nyear) -1
cagr.tlt <- (eindex.tlt[length(eindex.tlt)] / eindex.tlt[1]) ^ (1/nyear) -1

#calculate annual volatility
vol.spy <- sd(excess.spy, na.rm=TRUE) * sqrt(52)
vol.tlt <- sd(excess.tlt, na.rm=TRUE) * sqrt(52)

#Calculate sharpe ratio
sr.spy <- cagr.spy / vol.spy
sr.tlt <- cagr.tlt / vol.tlt
  
#for each WEIGHT in x, compute sharpe ratio of that portfolio.  
sharpe.ratio <- function(x, a = train$ret.spy,b = train$ret.tlt,c = train$fed.p){

  s.ratio <- rep(NA, length(x))
                  
  for (i in 1:length(x)){
    port.ret <- rep(NA, length(a))
    ex.ret <- rep(NA, length(a))
  
    #excess returns for each week
    for (j in 1:length(a)){
      port.ret[j] <- x[i]*a[j] + (1-x[i])*b[j]
    }
    
    #now i have a vector of returns.
    #need to calculate excess returns
    for (j in 2:length(a)){
      
      ex.ret[j] <- port.ret[j] - c[j-1] / 52
    }
    
    #need to calculate excess returns index
    eindex <- rep(NA, length(a))
    eindex[1] <- 100
    for (j in 2:nrow(train)){
      eindex[j] <- eindex[j-1] * (1+ex.ret[j])
    }
    
    #number of years of data:
    n <- (length(a)-1) / 52
    #compounded annual growth rate:
    cagr <- (eindex[length(eindex)] / eindex[1])^(1/n) - 1
    #annualized volatility:
    v <- sd(ex.ret, na.rm=TRUE) * sqrt(52)
    
    #sharpe ratio:
    s.ratio[i] <- cagr/v
  }
  return(s.ratio)
    
}

#create curve for sharpe ratio function    
curve(sharpe.ratio,from=0,to=1, main="Sharpe Ratio for Various Weights in Portfolio
      (S&P is Default Weight)",xlab="Portfolio Weight of S&P ETF",
      ylab="Sharpe Ratio of Construced Portfolio")
```

![](Portfolio_github_files/figure-html/cars-4.png)<!-- -->

```r
#Find the highest point of the function
optimize(sharpe.ratio,lower=0,upper=1,maximum=TRUE)
```

```
## $maximum
## [1] 0.5958502
## 
## $objective
## [1] 0.3634139
```

```r
#compute returns series for each of three assets
test$fed.p <- sapply(test$fed.rate, perc)   

#calculate returns of s&p and treasury 
test$ret.spy <- returns(test$close.spy)
test$ret.tlt <- returns(test$close.tlt)   

#calculate returns of optimized portfolio
test$ret.port <- returns(.5958*test$close.spy +(1-.5958)*test$close.tlt)  
 
#calculate excess returns   
ex.spy <- rep(NA, nrow(test))
ex.tlt <- rep(NA, nrow(test))
ex.port <- rep(NA, nrow(test))

for (i in 2:nrow(test)){
  ex.spy[i] <- test$ret.spy[i] - test$fed.p[i-1] / 52   
  ex.tlt[i] <- test$ret.tlt[i] - test$fed.p[i-1] / 52
  ex.port[i] <- test$ret.port[i] - test$fed.p[i-1] / 52
}

#calculate excess returns index
#excess returns index
index.spy <- rep(NA, nrow(test))
index.spy[1] <- 100
index.tlt <- rep(NA, nrow(test))
index.tlt[1] <- 100
index.port <- rep(NA, nrow(test))
index.port[1] <- 100

for (i in 2:nrow(test)){
  index.spy[i] <- index.spy[i-1] * (1+ex.spy[i])
  index.tlt[i] <- index.tlt[i-1] * (1+ex.tlt[i])
  index.port[i] <- index.port[i-1] * (1+ex.port[i])
}

#plot excess returns index of each set on the same plot
plot(test$date,index.port, type='l',
     xlab="Month in 2014",ylab="Excess Return Index",
     main="Excess Returns of Various Portfolio Constructions",
     ylim=c(95,120),col="firebrick")
lines(test$date,index.tlt,type='l',col="cadetblue")
lines(test$date, index.spy,type='l')
legend('topleft',legend=c("Optimized Portfolio",'Long Term Treasury ETF','S&P ETF'),
       col=c('firebrick','cadetblue','black'),
       lty=1, cex=0.8)
abline(h=100, lty=2, col="gray")

#How much did each asset earn by the end of the test set period?
index.spy[length(index.spy)]
```

```
## [1] 107.8763
```

```r
index.tlt[length(index.tlt)]
```

```
## [1] 116.376
```

```r
index.port[length(index.port)]
```

```
## [1] 110.2133
```

```r
#plot the long term treasury closing price over entire dataset
plot(data.x$date, data.x$close.tlt, main="Long Term Treasury ETF Time Series",
     xlab='Date',ylab='Closing Price [$]', type='l')
```

![](Portfolio_github_files/figure-html/cars-5.png)<!-- -->

