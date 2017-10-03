---
layout: post
title: Combining Github repositories traffic plots
tags: [Rselenium, webscraping]
---

This post will show how to use the [RSelenium](https://cran.r-project.org/web/packages/RSelenium/vignettes/RSelenium-basics.html) package to scrape your own github account to retrieve all that fun traffic data and create a single traffic plot for your account. 

![](img/gh_traffic.jpeg)


## Packages

```r
library(RSelenium)
library(XML)
library(ggplot2)
library(reshape2)
library(plyr)
library(dplyr)
```

Fill in the relevant information for your account. The team is usually your username, but it can be different. The repos can be a vector and since we are going in the front door of the site we can access the private repositories too! 

## Setup
```r
gh_user <- '<your github login name>'
gh_pass <- '<your github login password>'

gh_team <- '<team associated with account>'
repos <- '<repositories in team>'
```

## The function
```r
github_traffic <- function(gh_user,gh_pass,gh_team,repos){

#open the connection

rD <- RSelenium::rsDriver(verbose = FALSE)
remDr <- rD[["client"]]

#going to the first repo to invoke the login

remDr$navigate(sprintf('https://github.com/%s/%s/graphs/traffic',gh_team,repos[1]))

#entering the login information in the form and clicking the button. 

webElem <- remDr$findElement(using = 'id', value = "login_field")
webElem$setElementAttribute(attributeName = 'value',value = gh_user)
webElem <- remDr$findElement(using = 'id', value = "password")
webElem$setElementAttribute(attributeName = 'value',value = gh_pass)
webElem=remDr$findElement(using = 'xpath','//*[@id="login"]/form/div[4]/input[3]')
webElem$clickElement()
Sys.sleep(1)

# Retrieve the plots into an html
out <- plyr::llply(repos,function(repo){
  remDr$navigate(sprintf('https://github.com/%s/%s/graphs/traffic',gh_team,repo))
  Sys.sleep(1)
  out <- XML::htmlParse(remDr$getPageSource(),asText = TRUE)
  sapply(c('clones','visitors'),function(type){
  XML::getNodeSet(out,sprintf(sprintf('//*[@id="js-%s-graph"]/div/div[1]/svg/g/g',type)))
},simplify = FALSE,USE.NAMES = TRUE)
},.progress = 'text')

# set the names (llply doesnt)
names(out) <- repos

# that's it we dont need the connection anymore
remDr$close()
rD[["server"]]$stop()

# sort the data from html into a data.frame

plot_data <- plyr::ldply(out,function(repo){
  plyr::mdply(names(repo),function(type){
    
    dat <- repo[[type]]
  
    if(is.null(dat)) return(NULL)
    
    yticks_total <- as.numeric(sapply(getNodeSet(dat[[2]],'g'),XML::xmlValue))
    yticks_unique <- as.numeric(sapply(getNodeSet(dat[[5]],'g'),XML::xmlValue))
    
    x <- data.frame(type=type,
                    date = as.Date(sapply(getNodeSet(dat[[1]],'g'),XML::xmlValue),format = '%m/%d'),
                    total = as.numeric(sapply(getNodeSet(dat[[3]],'circle'),XML::xmlGetAttr,name = 'cy')),
                    unique = as.numeric(sapply(getNodeSet(dat[[4]],'circle'),XML::xmlGetAttr,name = 'cy')))
    
    x$total <- rescale(x$total,rev(range(yticks_total)))
    x$unique <- rescale(x$unique,rev(range(yticks_unique)))
    
    x%>%reshape2::melt(.,c('type','date'),variable.name=c('metric'))
  })
},.id='repo')%>%select(-X1)


#Create the plot

ggplot(plot_data,aes(x=date,y=value,colour=repo))+
  geom_point()+geom_line()+
  facet_grid(type~metric,scales='free_y')+
  scale_x_date(date_breaks = "1 day",date_labels = "%m/%d")+
  theme_bw()+
  theme(axis.text.x = element_text(angle=90),legend.position = 'top')+
  labs(title=sprintf('Github Team: %s',gh_team))
}

# run the function

traffic_plot <- github_traffic(gh_user=gh_user,
                               gh_pass=gh_pass,
                               gh_team=gh_team,
                               repos=repos)

# print the plot                               
traffic_plot
                               
```

If the function fails for some reason this will release the port Rselenium is holding ransom.
```r
rD <- rsDriver(verbose = FALSE,port=4444L)
remDr <- rD$client
remDr$close()
```