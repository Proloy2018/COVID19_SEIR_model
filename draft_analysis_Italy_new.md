COVID19 - Forecast and predictions using a time dependent SEIR model -
Italy
================
Paolo Girardi
28 Marzo, 2020

<a rel="license" href="http://creativecommons.org/licenses/by-nc/4.0/"><img alt="Creative Commons License" style="border-width:0" src="https://i.creativecommons.org/l/by-nc/4.0/88x31.png" /></a><br />This
work is licensed under a
<a rel="license" href="http://creativecommons.org/licenses/by-nc/4.0/">Creative
Commons Attribution-NonCommercial 4.0 International License</a>.

# Disclaimer

  - We want to investigate the evolution of the coronavirus pandemic in
    Italy from a statistical perspective using aggregated data.

  - Our point of view is that of surveillance with the goal of detecting
    important changes in the underlying (random) process as soon as
    possible after it has occured.

  - We use data provided by Italian Civil Protection Department

  - This document is in a draft mode, and it is continuously updated.

  - The layout of the draft must definitely be improved.

\*NB: set the file output format to

\#output:html\_document:  
df\_print: paged  
pdf\_document:  
toc: yes

which performs the same analysis enabling Javascript Pictures.

## The COVID dataset

The present analysis started from the dataset on COVID19 updated in
<https://github.com/pcm-dpc/COVID-19>, database provided by the Italian
Civil Protection.

# Software

Install packages `dygraphs`, `xts` and `EpiDynamics` if not available

``` r
checkpackage <- function(package) {
  if (!package %in% installed.packages()) install.packages(package)
}
checkpackage("dygraphs")
checkpackage("xts")
checkpackage("EpiDynamics")
checkpackage("webshot")
checkpackage("bsts")
checkpackage("ggplot2")
checkpackage("knitr")
checkpackage("splines")
```

and load them.

``` r
library(dygraphs)
library(xts)
```

    ## Loading required package: zoo

    ## 
    ## Attaching package: 'zoo'

    ## The following objects are masked from 'package:base':
    ## 
    ##     as.Date, as.Date.numeric

``` r
library(EpiDynamics)
library(webshot)
library(bsts)
```

    ## Loading required package: BoomSpikeSlab

    ## Loading required package: Boom

    ## Loading required package: MASS

    ## 
    ## Attaching package: 'Boom'

    ## The following object is masked from 'package:stats':
    ## 
    ##     rWishart

    ## 
    ## Attaching package: 'BoomSpikeSlab'

    ## The following object is masked from 'package:stats':
    ## 
    ##     knots

    ## 
    ## Attaching package: 'bsts'

    ## The following object is masked from 'package:BoomSpikeSlab':
    ## 
    ##     SuggestBurn

``` r
library(EpiDynamics)
library(ggplot2)
library(knitr)
library(splines)
```

# Source of the data

Download the data from

<https://github.com/pcm-dpc/COVID-19/>

# Results

## Load dataset

``` r
rm(list=ls())
###import italian updated dataset 
dat_csv<-read.csv("https://raw.githubusercontent.com/pcm-dpc/COVID-19/master/dati-andamento-nazionale/dpc-covid19-ita-andamento-nazionale.csv",header=T)
days<-dim(dat_csv)[1]
dat_csv$t<-1:days
# The total number of epidemic day is
days
```

    ## [1] 34

Several outcomes can be potentially monitored, that is

``` r
names(dat_csv[,-c(1:2,17:19)])
```

    ##  [1] "ricoverati_con_sintomi"      "terapia_intensiva"          
    ##  [3] "totale_ospedalizzati"        "isolamento_domiciliare"     
    ##  [5] "totale_attualmente_positivi" "nuovi_attualmente_positivi" 
    ##  [7] "dimessi_guariti"             "deceduti"                   
    ##  [9] "totale_casi"                 "tamponi"                    
    ## [11] "note_it"                     "note_en"                    
    ## [13] "t"

It is worth noting that some outcomes present negative counts in some
regions. It looks like some of these negative counts are redesignations.
Outcomes presenting negative values cannot be analyzed using the
proposed model.

Then we extract the
timeseries.

``` r
myDateTimeStr1 <- paste(substr(dat_csv$data,1,10),substr(dat_csv$data,12,19))
myPOSIXct1 <- as.POSIXct(myDateTimeStr1, format="%Y-%m-%d %H:%M:%S")
days_dy<-as.Date(myPOSIXct1)
dat_csv_dy<-xts(dat_csv[,-c(1:2,13:15)], order.by = days_dy, frequency = 7)
```

``` r
p <- dygraph(dat_csv_dy,main=paste("Italy",sep =""),xlab="Day",height=400,width=800) 
p
```

![](draft_analysis_Italy_new_files/figure-gfm/unnamed-chunk-6-1.png)<!-- -->

### The S(E)IR model (to be revised)

With the aim of predicting the future number of COVID19 cases on the
basis of the actual data, we used a SEIR model applied to the COVID19
epidemic to Italy

We will consider the classical [SIR
model](https://en.wikipedia.org/wiki/Compartmental_models_in_epidemiology)
\[@Kermack1927\].

The model divides a population of hosts into three classes: susceptible,
infected, recovered. The model describes how the portion of the
population in each of these classes changes with time. Births are
modeled as flows from “elsewhere” into the susceptible class; deaths are
modeled as flows from the \(S\), \(I\), or \(R\) compartment into
“elsewhere”. If \(S\), \(I\), and \(R\) refer to the numbers of
individuals in each compartment, then these **state variables** change
according to the following system of differential equations:
\[\begin{aligned}
\frac{d}{dt}S(t) &= B(t)-\lambda\,S(t)-\mu\,S(t)\\
\frac{d}{dt}I(t) &= \lambda\,S(t)-\gamma\,I(t)-\mu\,I(t)\\
\frac{d}{dt}R(t) &= \gamma\,I(t)-\mu\,R(t).\\
\end{aligned}\] Here, \(B\) is the crude birth rate (births per unit
time), \(\mu\) is the death rate and \(\gamma\) is the recovery rate.
We’ll assume that the force of infection, \(\beta\), for a constant
population \(N\) \[\lambda = \beta\,\frac{I}{N},\] so that the risk of
infection a susceptible faces is proportional to the *prevalence* (the
fraction of the population that is infected). This is known as the
assumption of frequency-dependent transmission.

# The reproduction number of COVID19.

The number of infected individuals \(I\) at time \(t\) is approximately
\[I(t)\;\approx\;I_0\,e^{(R_0-1)\,(\gamma+\mu)\,t}\] where \(I_0\) is
the (small) number of infectives at time \(0\), \(\frac{1}{\gamma}\) is
the infectious period, and \(\frac{1}{\mu}\) is the host lifespan.

\(R_0\) is the reproduction number
(<https://en.wikipedia.org/wiki/Basic_reproduction_number>) and
indicates how contagious an infectious disease is.

Taking logs of both sides, we get

\[\log{I}(t)\;\approx\;\log{I_0}+(R_0-1)\,(\gamma+\mu)\,t,\] which
implies that a semi-log plot of \(I\) vs \(t\) should be approximately
linear with a slope proportional to \(R_0\) and the recovery
rate.

``` r
dat_csv_dy$log_totale_attualmente_positivi<-log(dat_csv_dy$totale_attualmente_positivi)
p <- dygraph(dat_csv_dy$log_totale_attualmente_positivi,main=paste("Italy"),ylab="Log Infected case",xlab="Day",height=400,width=800) 
p
```

![](draft_analysis_Italy_new_files/figure-gfm/unnamed-chunk-7-1.png)<!-- -->

We estimate the \(R_0\) parameter in the linear model.

\[
\log(I(t))= \beta_0 + \beta_1  t +e_t.
\]

The estimated slope coefficient \(\hat\beta_1\) is used to estimate
\(R_0\) as in the following formula:

\[
\widehat\beta_1=(\widehat{R_0}-1)\,(\gamma+\mu)
\] The parameter \(\mu\)\<\<\(\gamma\) and it can not be considered. As
consequence, R0 can be estimated as follows \[
\hat{R_0}=1+\frac{\hat{\beta_1}}{\gamma}
\] Respect to the SIR model, \(R_0\) can be estimated as follows: \[
\hat{R_0}=\frac{\hat{\beta}}{\gamma}
\] And this was we can retrive the value of \(\beta\) in the SEIR model
by means of \[
\hat{R_0}=\frac{\hat{\beta}}{\gamma}=1+\frac{\hat{\beta_1}}{\gamma}\\
\hat{\beta}=(1+\frac{\hat{\beta_1}}{\gamma})*{\gamma}={\gamma}+\hat{\beta}_1
\] where \(\beta_1\) is the slope coefficient. \\

The incubation period for the coronavirus is in mean 5.1 days with a
range from 2-14 days. Please see
<https://www.worldometers.info/coronavirus/coronavirus-incubation-period/>.
However, the incubation period is used for epidemic diseases that causes
the immediate home isolation of infected subjects. The duration of the
diseases is about 2 weeks.

However, in the calculation of R0 we considered an infectious period of
18 days (<https://www.nejm.org/doi/10.1056/NEJMoa2001316>).

We calculate several R0 values, each one based on a mobile window of 5
days, that can be sufficient to estimate a local trend, in order to
assess if the R0 trend is decreasing (how is expected to be). In this
way, the R0 for the first and the last two days of observation is
impossibile to estimate.

``` r
#calculate r0 based with a mobile window of 5 days
#vector for beta and standard deviation
duration<-18
beta_vec<-NULL
sd_vec<-NULL
#for cycle for R0 estimates from days-2 to days+2
for (i in 3:(days-2)){
fit <- lm(log(totale_attualmente_positivi)~t,data=dat_csv[(i-2):(i+2),])
beta_vec<-c(beta_vec,coef(fit)[2])
sd_vec<-c(sd_vec,coef(summary(fit))[2,2])
}

label<-as.Date(substr(dat_csv$data,1,10))[3:(days-2)]


mean  <- 1+(beta_vec*duration)
lower <- 1+((beta_vec-1.96*sd_vec)*duration)
upper <- 1+((beta_vec+1.96*sd_vec)*duration)

df <- data.frame(label, mean, lower, upper)


fp <- ggplot(data=df, aes(x=label, y=mean, ymin=lower, ymax=upper)) +
  geom_pointrange() +
  geom_hline(yintercept=1, lty=2) +  # add a dotted line at x=1 after flip
  xlab("Date") + ylab("R0 Mean (95% CI)") +
  theme_bw() 
print(fp)
```

![](draft_analysis_Italy_new_files/figure-gfm/R0%20trend-1.png)<!-- -->

The R0 shows a decreasing trend in the last period. We use the estimated
trend between R0 and time to calculate the future R0 value for the next
14 days. We predict beta (and R0) for the next 14 days by means of a
linear regressione model, assuming a Normal distribution for the beta
(the slope).

``` r
time<-3:(days-2)
weekend<-rep(c(1,2,3,4,5,6,7),ceiling(days/7))[3:(days-2)]
data=data.frame(time,weekend)
beta.model<-glm(beta_vec~time+I(cos(time*2*pi/7))+I(sin(time*2*pi/7)),weights = 1/sd_vec,family=gaussian,data=data)
summary(beta.model)
```

    ## 
    ## Call:
    ## glm(formula = beta_vec ~ time + I(cos(time * 2 * pi/7)) + I(sin(time * 
    ##     2 * pi/7)), family = gaussian, data = data, weights = 1/sd_vec)
    ## 
    ## Deviance Residuals: 
    ##      Min        1Q    Median        3Q       Max  
    ## -0.44871  -0.08318   0.02367   0.10828   0.51895  
    ## 
    ## Coefficients:
    ##                           Estimate Std. Error t value Pr(>|t|)    
    ## (Intercept)              0.2928181  0.0085424  34.278  < 2e-16 ***
    ## time                    -0.0071215  0.0003555 -20.033  < 2e-16 ***
    ## I(cos(time * 2 * pi/7)) -0.0008654  0.0043605  -0.198 0.844219    
    ## I(sin(time * 2 * pi/7)) -0.0159716  0.0043029  -3.712 0.000987 ***
    ## ---
    ## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
    ## 
    ## (Dispersion parameter for gaussian family taken to be 0.04938084)
    ## 
    ##     Null deviance: 24.5448  on 29  degrees of freedom
    ## Residual deviance:  1.2839  on 26  degrees of freedom
    ## AIC: -147.98
    ## 
    ## Number of Fisher Scoring iterations: 2

``` r
anova(beta.model,test="Chisq")
```

    ## Analysis of Deviance Table
    ## 
    ## Model: gaussian, link: identity
    ## 
    ## Response: beta_vec
    ## 
    ## Terms added sequentially (first to last)
    ## 
    ## 
    ##                         Df Deviance Resid. Df Resid. Dev  Pr(>Chi)    
    ## NULL                                       29    24.5448              
    ## time                     1  22.5769        28     1.9679 < 2.2e-16 ***
    ## I(cos(time * 2 * pi/7))  1   0.0036        27     1.9643 0.7861065    
    ## I(sin(time * 2 * pi/7))  1   0.6804        26     1.2839 0.0002058 ***
    ## ---
    ## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1

``` r
#there is an effect of the week, as supposed 
forecast=14
# add 'fit', 'lwr', and 'upr' columns to dataframe (generated by predict)
weekend_pre<-rep(c(1,2,3,4,5,6,7),ceiling((days+forecast)/7))[1:(days+forecast)]
datanew<-data.frame(time=1:(days+forecast),weekend=weekend_pre)
pre<-predict(beta.model,type='response',newdata=datanew,se.fit=TRUE)
date<-seq(as.Date("2020-02-24"),as.Date("2020-02-24")+forecast-1+dim(dat_csv)[1],1)
predict <- data.frame(beta_vec=c(rep(NA,2),beta_vec,rep(NA,forecast+2)),time=date,fit=pre$fit,lwr=pre$fit-1*1.96*pre$se.fit,upr=pre$fit+1*1.96*pre$se.fit)
beta.predict<-predict 
r0.predict<-beta.predict
r0.predict[,c(1,3:5)]<-r0.predict[,c(1,3:5)]*duration+1
# plot the points (actual observations), regression line, and confidence interval
p <- ggplot(r0.predict, aes(date,beta_vec))
p <- p + geom_point() +labs(x="Date",y="R0 value") 
p <- p + geom_line(aes(date,fit))
p <- p + geom_ribbon(aes(ymin=lwr,ymax=upr), alpha=0.3)
p
```

    ## Warning: Removed 18 rows containing missing values (geom_point).

![](draft_analysis_Italy_new_files/figure-gfm/R0%20forecast-1.png)<!-- -->

R0 passes from a value of NA in the initial phase to an estimated value
of 0.33 at the ending of the 14-days forecast.  
We use the library(EpiDynamics) and the function SEIR() to implement a
SEIR model:

\[\begin{aligned}
\frac{d}{dt}S(t) &= \mu (N-S)-\beta \frac{SI}{N}-\nu S\\  
\frac{d}{dt}E(t) &= \beta \frac{SI}{N}-(\mu+\sigma) E    \\
\frac{d}{dt}I(t) &=\sigma E- (\mu+\gamma)I \\
\frac{d}{dt}R(t) &= \gamma\,I(t)-\mu\,R(t)+\nu S.\\
\end{aligned}\]

<img src="https://upload.wikimedia.org/wikipedia/commons/3/3d/SEIR.PNG"/>

Respect to the previous formulation, the compartment of exposed people
(E) was inserted between the susceptible and infected compartments.  
(<https://en.wikipedia.org/wiki/Bayesian_structural_time_series>). The
parameter \(\nu\) is the rate of vaccination.

We want to make a short term forecast (14 days).

We made a forecast by means of a SEIR model fixing a series of initial
status:  
*S:N, the size of Italian population  
*E: The number of exposed people, but supposed to be at least 3 times
the infected (3 x I\_start)  
*I\_start: initial number of COVID-19 cases  
*R\_start: initial number of recovered

and parameters:  
\- beta: gamma + slope  
\- gamma= 1/duration of diseases (duration=21 days)  
\- sigma: the coronavirus transmission rate  
\- mu0: the overall mortality rate

``` r
# initial number of infectus
I_start<-dat_csv$totale_attualmente_positivi[dim(dat_csv)[1]]; I_start
```

    ## [1] 70065

``` r
# initial number of recovered, based on the proportion of discharged from the health services
prop<-dat_csv$dimessi_guariti[dim(dat_csv)[1]]/dat_csv$totale_ospedalizzati[dim(dat_csv)[1]]
R_start<-prop*dat_csv$totale_casi[dim(dat_csv)[1]]; R_start
```

    ## [1] 37507.31

``` r
# Italian population
N=60480000
# duration of COVID19 as the sum of incubation (5-14 days) and the duration of diseases (about 18 days)
duration<-18
#mortality rate 
mu0<-1/(82*365.25) # 1/lifespan
```

We try to estimates sigma (from exposed to infected) and the number of
exposed population on the basis of the last 5 days of observed number of
total infected people.

s

``` r
# number of the last considered days for calibration
last<-10
#I_start
I_1<-dat_csv$totale_attualmente_positivi[(days-last)]
#R_start
R_1<-prop*dat_csv$totale_casi[(days-last)]
#loglikelihood function
LogLikfun <- function (initials,parameters,obs) {
n<-length(obs)
N=60480000
seir_fit <- SEIR(pars = parameters, init = initials, time = 0:last)
#Poisson
#sum(dpois(x=obs[-1],lambda = seir_fit$results$I[-1]*N,log=TRUE))
#Neg Binomial
sum(dnbinom(x=obs[-1],mu=seir_fit$results$I[-1]*N,size=N,log=TRUE))
#SSE
#-sum((obs[-1]-seir_fit$results$I[-1]*N)^2)
}
### logit and its inverse for sigma 
logit <- function (p) log(p/(1-p))    # the logit transform
expit <- function (x) 1/(1+exp(-x))   # inverse logit

f1<-function(par){
sigma<-expit(par)
f_exp<-3*I_1/N
parameters <- c(mu = mu0, beta =(mean(beta_vec[1:(days-last-2)])+1/duration), sigma = sigma, gamma =1/duration)
initials <- c(S = (1-(f_exp+I_1/N-R_start/N)), E = f_exp, I = I_1/N, R = R_1/N)
-LogLikfun(initials,parameters,dat_csv$totale_attualmente_positivi[c((days-last):days)])
}
est<-optim(fn=f1,par=logit(0.1))
```

    ## Warning in optim(fn = f1, par = logit(0.1)): one-dimensional optimization by Nelder-Mead is unreliable:
    ## use "Brent" or optimize() directly

``` r
#estimated par
expit(est$par)
```

    ## [1] 0.06602374

``` r
sigma<-expit(est$par)
f_exp<-3*I_1/N
parameters <- c(mu = mu0, beta =(mean(beta_vec[1:(days-last-2)])+1/duration), sigma = sigma, gamma = 1/duration)
initials <- c(S = (1-(f_exp+I_1/N-R_1/N)), E = f_exp, I = I_1/N, R = R_1/N)
pro<-SEIR(pars = parameters, init = initials, time = 0:last)
#predicted vs observed
cbind(pro$results$I*N,dat_csv$totale_attualmente_positivi[c((days-last):days)])
```

    ##           [,1]  [,2]
    ##  [1,] 28710.00 28710
    ##  [2,] 32760.35 33190
    ##  [3,] 36760.15 37860
    ##  [4,] 40766.01 42681
    ##  [5,] 44828.09 46638
    ##  [6,] 48991.28 50418
    ##  [7,] 53296.55 54030
    ##  [8,] 57781.91 57521
    ##  [9,] 62483.27 62013
    ## [10,] 67435.04 66414
    ## [11,] 72670.81 70065

The values of initial parameters are

``` r
#mu0
mu0;
```

    ## [1] 3.338842e-05

``` r
#sigma
sigma
```

    ## [1] 0.06602374

``` r
#gamma
1/duration
```

    ## [1] 0.05555556

``` r
#the unknowrn fraction of exposed people is
pro$results$E[last+1];
```

    ## [1] 0.002360544

``` r
#that is 
pro$results$E[last+1]/pro$results$I[last+1]
```

    ## [1] 1.964554

``` r
#times the infected
```

For the beta parameter, we perfom a simulation on its trend by means of
a Bayesian Structural Time Series using the library bsts of R.  
We estimate a BSTS model specifing a local linear trend ans 1000
simulation.

We made a forecast forward of 14 days dropping the first 100 simulation
as burn-in.

``` r
# Bayesian Structural Time Series
ss <- AddLocalLinearTrend(list(), beta_vec)
ss <- AddSeasonal(ss, beta_vec, nseasons = 7)
model1 <- bsts(beta_vec,
               state.specification = ss,
               niter = 1000,seed=123)
```

    ## =-=-=-=-= Iteration 0 Sat Mar 28 19:53:34 2020 =-=-=-=-=
    ## =-=-=-=-= Iteration 100 Sat Mar 28 19:53:34 2020 =-=-=-=-=
    ## =-=-=-=-= Iteration 200 Sat Mar 28 19:53:34 2020 =-=-=-=-=
    ## =-=-=-=-= Iteration 300 Sat Mar 28 19:53:34 2020 =-=-=-=-=
    ## =-=-=-=-= Iteration 400 Sat Mar 28 19:53:34 2020 =-=-=-=-=
    ## =-=-=-=-= Iteration 500 Sat Mar 28 19:53:34 2020 =-=-=-=-=
    ## =-=-=-=-= Iteration 600 Sat Mar 28 19:53:34 2020 =-=-=-=-=
    ## =-=-=-=-= Iteration 700 Sat Mar 28 19:53:34 2020 =-=-=-=-=
    ## =-=-=-=-= Iteration 800 Sat Mar 28 19:53:34 2020 =-=-=-=-=
    ## =-=-=-=-= Iteration 900 Sat Mar 28 19:53:34 2020 =-=-=-=-=

``` r
par(mfrow = c(1,1))
plot(model1, "components")
```

![](draft_analysis_Italy_new_files/figure-gfm/bsts%20model%20fit%20and%20prediction-1.png)<!-- -->

``` r
#previsioni
pred1 <- predict(model1, horizon = 16, burn = 100)
par(mfrow = c(1,1))
plot(pred1 , ylab="Beta1 coefficient",main="Data and predictions",ylim=c(-1,1)*IQR(pred1$distribution)+median(pred1$distribution))
```

![](draft_analysis_Italy_new_files/figure-gfm/bsts%20model%20fit%20and%20prediction-2.png)<!-- -->

``` r
# matrix of beta coefficients
coef<-(pred1$distribution[,3:16])
par(mfrow = c(1,1))
```

For each vector of simulated beta coefficients we perform a SEIR model.
We save the results and plot a credible interval at 50% (25-75%
percentile).

``` r
seir1_sim<-NULL
for(s in 1:dim(coef)[1]){
  # average number of single connections of an infected person
  # less contacts, less probability of new infections
  # we keep constant the other parameters
  forecast<-14
  seir1<-NULL
  for(i in 1:forecast){
    parameters <- c(mu = mu0, beta = (matrix(coef[s,i])+1/duration), sigma = sigma, gamma = 1/duration)
    if( i==1) initials <- c(S = 1-(pro$results$E[last+1]+I_start/N+R_start/N), E =pro$results$E[last+1] , I = I_start/N, R = R_start/N)
    if( i>1) initials <- c(S = seir1_temp$results$S[2], E = seir1_temp$results$E[2], I =seir1_temp$results$I[2], R = seir1_temp$results$R[2])
    seir1_temp <- SEIR(pars = parameters, init = initials, time = 0:1)
    seir1 <- rbind(seir1,SEIR(pars = parameters, init = initials, time = 0:1)$results[2,])
  }
  seir1_sim<-rbind(seir1_sim,cbind(rep(s,forecast),seir1))

}
seir1_sim[,2]<-rep(1:forecast,dim(coef)[1])
colnames(seir1_sim)[1]<-"sim"

### confidence limits
I_seir_med<-tapply(seir1_sim$I,seir1_sim$time,median)
I_seir_lwr<-tapply(seir1_sim$I,seir1_sim$time,quantile,p=0.25)
I_seir_upr<-tapply(seir1_sim$I,seir1_sim$time,quantile,p=0.75)

days.before<-date[1:days]
days.ahead<-date[(days+1):(days+forecast)]
step.ahead<-forecast+1
mu.lower<-c(dat_csv$totale_attualmente_positivi,I_seir_lwr*N)
mu.upper<-c(dat_csv$totale_attualmente_positivi,I_seir_upr*N)
mu.med<-xts(c(dat_csv$totale_attualmente_positivi,I_seir_med*N),order.by = c(days.before,days.ahead),frequency = 7)
counts<-mu.med
mu<-xts(x = as.matrix(cbind(counts,mu.lower,mu.upper)) , order.by = c(days.before,days.ahead))
p <- dygraph(mu,main=paste("Italy: Scenario  (Credible Interval ",100*0.50,"%)",sep = ""),ylab=" Infected",xlab="Day",height=400,width=800) %>%  dySeries(c("mu.lower", "counts", "mu.upper"),label="counts")
p<-p %>% dyLegend(show = "always", hideOnMouseOut = FALSE) %>%  dyShading(from = days.ahead[1], to = days.ahead[step.ahead], color = "#CCEBD6")%>% dyEvent(days.ahead[1], "Prediction", labelLoc = "bottom")
p
```

![](draft_analysis_Italy_new_files/figure-gfm/scenario%20plot%20-1.png)<!-- -->

At the end of the 2 weeks (2020-04-11): \*the number of infected is
(1.045641510^{5}).

\*the total number of COVID19 cases is expected to be (2.153227610^{5}).

The estimated (median) numbers of the current scenario by date are:

``` r
S_seir_med<-tapply(seir1_sim$S,seir1_sim$time,median)
E_seir_med<-tapply(seir1_sim$E,seir1_sim$time,median)
R_seir_med<-tapply(seir1_sim$R,seir1_sim$time,median)
forecast<-data.frame(S_seir_med,E_seir_med,I_seir_med,R_seir_med)*N
colnames(forecast)<-c("Susceptible","Exposed","Infected","Removed")
rownames(forecast)<-days.ahead
kable(forecast)
```

|            | Susceptible |   Exposed |  Infected |   Removed |
| ---------- | ----------: | --------: | --------: | --------: |
| 2020-03-29 |    60223101 | 139995.78 |  75357.46 |  41545.54 |
| 2020-03-30 |    60217467 | 136504.92 |  80159.98 |  45864.07 |
| 2020-03-31 |    60212734 | 132386.94 |  84464.58 |  50435.30 |
| 2020-04-01 |    60207787 | 128693.76 |  88284.95 |  55232.99 |
| 2020-04-02 |    60202066 | 126099.60 |  91692.83 |  60230.30 |
| 2020-04-03 |    60194951 | 124890.46 |  94751.81 |  65408.55 |
| 2020-04-04 |    60189845 | 121873.14 |  97524.23 |  70746.38 |
| 2020-04-05 |    60186101 | 117448.50 |  99956.81 |  76227.96 |
| 2020-04-06 |    60185090 | 110877.72 | 101973.08 |  81813.07 |
| 2020-04-07 |    60184807 | 103968.52 | 103334.71 |  87514.33 |
| 2020-04-08 |    60184781 |  96695.47 | 104176.58 |  93280.85 |
| 2020-04-09 |    60185038 |  91172.92 | 104628.53 |  99097.43 |
| 2020-04-10 |    60184223 |  86606.41 | 104812.27 | 104920.39 |
| 2020-04-11 |    60184657 |  80505.19 | 104564.15 | 110758.61 |
