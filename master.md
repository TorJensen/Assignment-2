---
title: "Assignment 2"
output: html_document
---

```{r, include = FALSE}
# load the required packages for the report and set options
require("rvest")
require("xml2")
require("readr")
require("stringr")
require("ggplot2")
require("ggthemes")
require("scales")
require("viridis")
require("raster")
require("sp")
require("splitstackshape")
require("dplyr")
options(scipen=999)
```

```{r, eval = FALSE}
# Create Dataset by scraping the I Paid a Bribe website, link to pre-scraped dataset below this block

#General considerations website layout:  
#Data about the bribe; time, place, amount, category, etc. is available from the main search page.  
#The full text describing the event is only available from the link itself. 
#Can get most of the data from the main "search", without having to go through every link that has been pulled

## Initialize dataframe with the desired data columns
df <- data_frame(title = character(), 
                 amount = character(), 
                 dep = character(), 
                 trans = character(), 
                 views = character(),
                 city = character(),
                 date = character(), 
                 time = character())

## initialize empty dataframe for 
dftemp <- data_frame()

## Scrape data from "I paid a bribe" website

for (i in 0:99) { #loop through first 100 pages, 10 results per page = 1000 
  link <- paste("http://www.ipaidabribe.com/reports/paid?page=",i*10, sep = "") #Create hyperlink based on loop function
  print(paste("processing", i, sep = " ")) #progress report
  main <- read_html(link, encoding = "UTF-8") #define the static part of link references
  
  title <- main %>% 
    html_nodes(".heading-3 a") %>% 
    html_text() 
  
  amount <- main %>% # feed `main.page` to the next step
    html_nodes(".paid-amount span") %>% # get the CSS nodes
    html_text() # extract the link text
  
  dep <- main %>% # feed `main.page` to the next step
    html_nodes(".name a") %>% # get the CSS nodes
    html_text() # extract the link text
  
  trans <- main %>% # feed `main.page` to the next step
    html_nodes(".transaction a") %>% # get the CSS nodes
    html_text() # extract the link text
  
  views <- main %>% # feed `main.page` to the next step
    html_nodes(".overview .views") %>% # get the CSS nodes
    html_text() # extract the link text
  
  city <- main %>% # feed `main.page` to the next step
    html_nodes(".location") %>% # get the CSS nodes
    html_text() # extract the link text
  
  date <- main %>% # feed `main.page` to the next step
    html_nodes(".date") %>% # get the CSS nodes
    html_text() # extract the link text
  
  time <- main %>% # feed `main.page` to the next step
    html_nodes(".time-span") %>% # get the CSS nodes
    html_text() # extract the link text
  
  dftemp <- cbind(title, amount, dep, trans, views, city, date, time) #bind the variables together into a 10 by n dataframe
  df <- rbind(df,dftemp) #rbind the temp dataframe for this page to the main dataframe
  
  Sys.sleep(1) #timer, wait 1 second
  cat(" done!\n") #progress report
}

#clean unused variables from workspace
rm("title", "amount", "dep", "trans", "views", "city", "date", "time", "dftemp", "i")

## Split the city column into a city and state column

df$states <- lapply(strsplit(as.character(df$city), "\\,"), "[", 2)
df$city <- lapply(strsplit(as.character(df$city), "\\,"), "[", 1)
```

```{r, eval = FALSE, echo = FALSE}
## Clean and order dataset, force correct data types, factors for faceting and checking number of levels
clean <- function(df) {
df$title <- df$title %>% #clean text
  str_replace_all(pattern = "\\n" , replacement = " ") %>%
  str_trim()

df$dep <- df$dep %>% 
  as.factor() #convert to factor 

df$trans <- df$trans %>% 
  as.factor() #convert to factor 

df$amount <- df$amount %>% #clean text from amount and convert to numeric
  str_replace_all(pattern = "Paid INR" , replacement = " ") %>% 
  str_replace_all(pattern = "," , replacement = "") %>% 
  str_trim() %>% 
  as.numeric() 

df$views <- df$views %>% #clean text from views and convert to numeric
  str_replace_all(pattern = "views" , replacement = " ") %>%
  str_trim() %>% 
  as.numeric()

df$city <- df$city %>% #clean text from city
  as.character() %>%
  str_trim() %>%
  as.factor() # conver tot factor

df$states <- df$states %>% #clean text from states
  as.character(df$states) %>%
  str_trim() %>% 
  as.factor() #convert to factor

df$time <- as.numeric(str_extract(df$time,"[0-9]*"))*!grepl("minutes|hours",df$time) #clean hours and minutes out of time stamp and change to whole number of days

df$date <- as.Date(df$date, format("%B %d, %Y")) #convert the date column to date format

df <- df[, c(1,2,3,4,5,6,9,7,8)] #rearrange columns

return (df)

}

df <- clean(df)

```

```{r, eval = FALSE, include = FALSE}
# futher Data cleaning

## look at headers and footers for obvious outliers in bribe size

### sorted in descending value 
df %>% 
  arrange(desc(amount)) %>% 
  head(5) 
#one single value that is incredibly high, equivalent to 113 million euros for a birth certificate, this could be removed as an unlikely entry

### sorted in ascending order
df %>% 
  arrange(desc(desc(amount))) %>% 
  head(5)
#quite a few instances of Rs 1, not necessarily wrong, 1 Rs is 1.4 eurocent, very small bribe, but given relative poverty, could be reasonable? can't really argue for setting a minimum threshold for bribes. 

## look for missing values in the character variables (title, dep, trans, city, state)
dfmissing <- df %>% filter(title == "" | dep == "" | trans == "" | city == "" | states == "") %>% head(5)
#only a single entry with no data

## count number of rows with NA values
narows <- nrow(df[!complete.cases(df),])
#total number is 0, no need to filter out NA values

## test for misassigned values in either 
duplicate <- match(df$city,df$states)
duplicate2 <- match(df$states,df$city)
dupcheck <- df[!is.na(duplicate),]
dupcheck2 <- df[!is.na(duplicate2),]
#3 values come up, city name is capital name, chandigarh is a state and an area so no problem

## filter based on the above criteria
df <- df %>% 
  filter(dep != "", amount < max(amount))  #Remove entry with missing data and remove the

## remove unwanted variables again used
rm("dfmissing", "narows", "duplicate", "duplicate2", "dupcheck", "dupcheck2")

```

```{r}
#Import the common csv file from github to work on, at end of project remove this reference, lines to be removed marked with [X]

df_raw <- read_csv("https://raw.githubusercontent.com/TorJensen/Assignment-2/master/df.csv") #read csv file from github [X]
df_raw <- df_raw[,2:10] #remove the rownames column - [X]

df <- df_raw #define df
rm(df_raw) #remove df_raw from the environment
```

Basic summary shows large difference in mean and median amounts, third quartile and mean difference indicates large relatively small amount of large bribes pulling up the mean, while the vast majority of bribes reported are relatively small. Almost half of the bribes are reported from Bangalore, and about 2/3rds of the bribes are recorded within the top 5 departments.  The reporting period ends at 12th of october, so we only have 1 month worth of data, which limits the time/weekeday/seasonality analsysis options available. 

```{r, echo = FALSE}

df <- df %>% 
  filter(dep != "", amount < max(amount))

#exploratory data analysis 

summary(df)

```

Looking closer at the distribution of bribes submitted per day, we can see that the majority of our dataset was submitted on a single day last month. This is definitely an outlier compared to the other days that average about 10-30 reports per day. Having a look at the data for that specific day, it seems to be reasonably well-distributed across states and departments, so it can probably rules out that the reason is a barrage of spam / incorrect reports. 

```{r, echo = FALSE}
#plot showing amount of records per day
p <- ggplot(df %>% group_by(date) %>% summarise(count = n()),aes(date, count))
p + geom_bar(stat = "identity") + 
  labs(x="Date", y="Count", title="Records per day") +
  theme(plot.title = element_text(lineheight=.8, face="bold", vjust=1))
```

a quick google news search for october reveals that an article was posted in Times of India on the 11th of October, which could be a possible explanation for the sudden spike link: https://www.google.dk/search?q=ipaidabribe&num=100&espv=2&biw=1920&bih=1067&source=lnt&tbs=cdr%3A1%2Ccd_min%3A10%2F1%2F2015%2Ccd_max%3A10%2F31%2F2015&tbm=nws

```{r, echo = FALSE}
#narrow down the the dataset to entries made on october 12rd
df_lump <- df %>% 
  filter(date == min(date)) #filter the data based on the earliest day in the dataset

summary(df_lump) # show summary data
```

doesn't seem to be any real relation, few outliers are for medium-sized amounts, but outlier status seems to be driven by other things than size in general.

a possibiliy could be a relation between the amount of text in each post - with a more descriptive "story" of the bribe than the standard, more views would be attracted, but this would require a further individual scrape of each posts' link to get the full comment text.

```{r, echo = FALSE}
# looking into relation between views and amount of data - do higher value bribe reports attract more attention 
p2 <- ggplot(df,aes(views, log(amount)))
p2 + geom_point(alpha = 0.4) + #lower alpha (transparency) to be able to identify clusters of data
  labs(x="Views", y="log(Amount)", title="Relationship Between Views and Amount of Data") +
  theme(plot.title = element_text(lineheight=.8, face="bold", vjust=1))
```

There are 27 states and 17 departments (with 40 transaction types) just in the 1000 records we have mined, so an analysis looking at transaction type level by state, for example, would need more than 1000 data points to be able to get a reasonable analysis (assuming some degree of normality.) Instead focus on higher level summaries, specific departments or specific states with a particular  

## get an idea of the sizes of the factors in our dataset: 

```{r, echo = FALSE}
levelcount <- lapply(df,levels) %>% summary
levelcount
```

In the gathered data we can see that the most bribes are paid in the department of Municipal Services, especially for issuing a birth certification. Facing this situation, we wanted to find out if there are any reasons for the high rate of corruption in this sector. Apart from the fact that the Indian citizens mostly have to pay bribes for every service they ask for, the high birth rate in India is probably the main reason for this circumstance. The more urgent the need of the documents, the higher the amount the people have to pay for the bribe. Furthermore, we have aggregated the amounts of bribes paid per department and this shows us that there is not just a wide selection of corruption in the sector of Municipal Services, but these public authorities also gain the highest amount of money by far. Relating to the high birth rate in India, as mentioned above, it is probably the easiest and most common way to be bribed. Due to the complicated taxes and licensing systems and corrupt police system, the bribe payments in the departments of “Income Tax” and “Police” belong to the highest in India.

The below tables dispays the top 5 departments by the number of bribes reported, transactions with the highest average amounts.

```{r, echo = FALSE}
# Number of bribes and their sizes, by variables in various combinations, these can be called throughout the text:

### below are some tables looking at the 

#department
df_dep <- df %>% 
  group_by(dep) %>% 
  summarise(count = n(), amount = sum(amount))%>% 
  mutate(avgamount = round(amount/count)) %>% 
  arrange(desc(avgamount))

#state
df_state <- df %>% 
  group_by(states) %>% 
  summarise(count = n(), amount = sum(amount))%>% 
  mutate(avgamount = round(amount/count)) %>% 
  arrange(desc(count))

#department and transaction type
df_deptrans <- df %>% 
  group_by(dep,trans) %>% 
  summarise(count = n(), sum = sum(amount))%>% 
  mutate(avgamount = round(sum/count)) %>% 
  select(dep, trans, count, avgamount)


df_deptrans[with(df_deptrans, order(desc(count))),]
#state and city
df_statecity <- df %>% 
  group_by(states, city) %>% 
  summarise(count = n(), sum = sum(amount)) %>% 
  mutate(avgamount = round(sum/count))

#state and department, the thougth was that some departments could be more corrupt in some areas compared to others, hard to tell with so few data for so many sates. 
df_statedep <- df %>% 
  group_by(states, dep) %>% 
  summarise(count = n(), sum = sum(amount)) %>% 
  mutate(avgamount = round(sum/count)) %>% 
  arrange(desc(count))

#number of views by state, are some states viewed more frequently, do states with lots of reports have more views, do states with a higher value of bribes have more views
df_stateviews <- df %>% 
  group_by(states) %>% 
  summarise(count = n(), views = sum(views), amount = sum(amount)) %>% 
  mutate(avgviews = round(views/count), avgamount = round(amount/count))

#number of views by department, are some departments viewed more frequently, do departments with lots of reports have more views, do departments with higher bribe value have more views.  
df_depviews <- df %>% 
  group_by(dep) %>% 
  summarise(count = n(), views = sum(views), amount = sum(amount)) %>% 
  mutate(avgviews = round(views/count), avgamount = round(amount/count))

#does number of views level out with time? 
df_viewsdate <- df %>% 
  group_by(date) %>% 
  summarise(count = n(), views = sum(views)) %>% 
  mutate(avgviews = round(views/count))

head(df_deptrans[with(df_deptrans, order(desc(count))),],5) 
head(df_deptrans[with(df_deptrans, order(desc(avgamount))),],5) 
```

We can see that the most pervasive bribes aren't necessarily the largest ones, this isn't suprising, but the two tables emphasize how you risk encountering corruption everywhere you go, with every interaction you have with the state. Ration cards are especially important as they are necessary to benefit from the fuel subsidies the indian government gives, and to access the public food distribution systems. For the less frequent but higher value amounts, it is striking that the issuing of [new PAN cards](http://www.indiacgny.org/pdf/pan_faq.pdf), which are not just used to pay taxes, but also as proofs of identity and are necessary to enter most financial transactions, are either bought on false from corrupt officials, or those officials extort people who need it. 

```{r, echo = FALSE}

### Bribes per department

bribe_dep <- aggregate(amount ~ dep, df, sum)
bribe_dep$dep <- as.character(bribe_dep$dep)

bribe_dep[which(with(bribe_dep, dep == "Municipal Services")), c(1)] <- "Mun. Services"
bribe_dep[which(with(bribe_dep, dep == "Food, Civil Supplies and Consumer Affairs")), c(1)] <- "Food/Civil Supplies"
bribe_dep[which(with(bribe_dep, dep == "Stamps and Registration")), c(1)] <- "Stamps and Reg."
bribe_dep[which(with(bribe_dep, dep == "Commercial Tax, Sales Tax, VAT")), c(1)] <- "Com./Sales/VAT"
bribe_dep[which(with(bribe_dep, dep == "Customs, Excise and Service Tax")), c(1)] <- "Cust./Ex./Serv"

bribe_dep <- bribe_dep %>% filter(dep != "") %>% arrange(desc(amount))

bribe_dep$dep <- as.factor(bribe_dep$dep)

bribe_dep$dep <- factor(bribe_dep$dep,levels(bribe_dep$dep)[c(8, 6, 11, 15, 2, 9, 16, 5, 3, 10, 14, 13, 4, 7, 17, 1, 12)])

p3 = ggplot(bribe_dep, aes(x = dep, y = amount, fill = dep))
p3 + geom_bar(stat="identity") + 
  theme_tufte() + 
  scale_y_continuous(labels=comma) +
  labs(x="", y="Amount", title="Bribe Amount per Department") +
  theme(axis.text.x = element_blank(),
        legend.position = "none", 
        plot.title = element_text(lineheight=.8, face="bold", vjust=1)) + 
  facet_wrap(~ dep, scales = "free_x") +
  scale_fill_viridis(discrete = TRUE)
```

Text for Janette's Part

```{r, echo = FALSE}
#Bribe amounts
bribe_sta <- aggregate(amount ~ states, df, sum)
bribe_n <- df %>% group_by(states) %>% count(states) # create column with number of bribes per state 
bribe_sta$n <- bribe_n$n # add column with number of bribes per state to amount of bribes per state
bribe_sta <- bribe_sta[-c(9),] # remove Hardoi
bribe_sta[which(with(bribe_sta, states == "Orissa")), c(1)] = "Odisha" #rename columns

# Download India GDP data

ind.gdp = read_html("https://en.wikipedia.org/wiki/List_of_Indian_states_by_GDP") %>%
  html_nodes(xpath = '//*[@id="mw-content-text"]/table[2]') %>% 
  html_table() %>% data.frame # then convert the HTML table into a data frame

names(ind.gdp) <- c("states", "co_ru")

ind.gdp$co_ru = as.numeric(gsub("," , "" , ind.gdp$co_ru)) #remove commas from indian number notation
ind.gdp = ind.gdp %>% mutate(gdp = co_ru * 10000000) %>% as.data.frame() %>% select(states, gdp)#1 crore is 10 million, convert to unit measure and stick it in a new dataframe
ind.gdp[which(with(ind.gdp, states == "Chattisgarh")), c(1)] = "Chhattisgarh" 
ind.gdp[which(with(ind.gdp, states == "Jammu & Kashmir")), c(1)] = "Jammu and Kashmir" 
ind.gdp[which(with(ind.gdp, states == "Andaman & Nicobar Islands")), c(1)] = "Andaman and Nicobar Islands" 

# Download India population data

ind.pop = read_html("https://en.wikipedia.org/wiki/List_of_states_and_union_territories_of_India_by_population") %>%
  html_nodes(xpath = '//*[@id="mw-content-text"]/table[2]') %>% 
  html_table() %>% data.frame %>% select(State.or.union.territory, Population..2011.Census..12.....of.Population.of.India..13.)

names(ind.pop) <- c("states", "population")

ind.pop <- ind.pop[-c(37), ] #remove rows 37, India, from population
ind.pop[which(with(ind.pop, states == "Manipurβ")), c(1)] = "Manipur"
ind.pop$population <-  str_extract(ind.pop$population, "(?<=\\d{19}...)[0-9,]*") %>% str_replace_all(",","") %>% as.numeric()

# Combine bribe_sta, ind.gdp and ind.pop

ind.df <- rbind(bribe_sta, data.frame(states = setdiff(ind.pop$states, bribe_sta$states), amount = rep(0,10), n = rep(0, 10)))

ind.gdp <- rbind(ind.gdp, data.frame(states = setdiff(ind.df$states, ind.gdp$states), gdp = rep(0,3)))

ind.df <- left_join(ind.df, ind.gdp, by = c("states" = "states"))
ind.df <- left_join(ind.df, ind.pop, by = c("states" = "states"))
ind.df <- mutate(ind.df, gdp_cap = gdp/as.numeric(population)) #create new colun with gdp per capita
ind.df <- mutate(ind.df, m_per_b = amount/n)

# Remove NaN values from dataset
          
rm.nan <- function(x)
do.call(cbind, lapply(x, is.nan))

ind.df[rm.nan(ind.df)] <- 0
```

Visualisation of GDP per capita and log(money per bribe), trying to see if there is a correlation between gdp per capita and the amount of money paid per bribe, e.g. do richer states have a higher bribe level than poorer states. GDP in $ or rupees?

```{r, echo = FALSE}
p4 <- ggplot(ind.df, aes(gdp_cap, log(m_per_b)))
p4 + geom_point() + 
  labs(x="GDP per Capita", y="log(Amount per Bribe)", title="GDP/Capita and Amount of Bribes Paid") +
  theme(plot.title = element_text(lineheight=.8, face="bold", vjust=1))
```

We created a map of India showing the total amount of bribes by each state. Here we can see, that the highest rate of bribes are made in the south of India, particularly in Kerala, Tamil Nadu, Karnataka. Jammu and Kashmir has also an high amount of bribes, so in conclusion the bribes are more concentrated in the north and south of India, near the borders to Pakistan and the Indian Ocean. Here we have total amounts of bribes paid about 2,000,000.  In the inside of the country the level of bribes paid decreases in comparison to the external states. One reason for this circumstance could be the high number of citizens in these regions compared to the other states in India.


```{r, message = FALSE, echo = FALSE}

## Visualize bribe/state

ind.map <- getData("GADM", country="IND", level=1)

bribe_s <- select(bribe_sta, states, amount)

am.count <- rbind(bribe_s, data.frame(states = setdiff(ind.map$NAME_1, bribe_s$states), amount = rep(0, 12)))

am.count <- am.count[-c(38),]
am.count <- am.count[- grep("Andaman and Nicobar", am.count$states),]

am.count[which(with(am.count, states == "Uttarakhand")), c(1)] <- "Uttaranchal"

am.count <- am.count[order(am.count$states, decreasing = FALSE),]

ind.map <- fortify(ind.map)

am.count$index <- as.numeric(tapply(ind.map$id, ind.map$id, length))
am.count <- data.frame(amount = am.count$amount, index = am.count$index)

am.count <- expandRows(am.count, "index")

ind.map$count <- am.count$amount

p5 <- ggplot() + 
  geom_polygon(data = ind.map, aes(x=long, y=lat, group=group, fill = count), colour = "black") +
  scale_fill_gradient("Bribes paid \nby state ", low="yellow",high="red", breaks = c(0, 1e+06, 2e+06, 3e+06, 4e+06),
                      labels=c("0", "1M", "2M", "3M", "4M")) +
  labs(x="", y="", title="Total Amount of Bribes by State") + 
  theme(axis.ticks.y = element_blank(),axis.text.y = element_blank(), 
        axis.ticks.x = element_blank(),axis.text.x = element_blank(), 
        plot.title = element_text(lineheight=.8, face="bold", vjust=1))
p5
```
