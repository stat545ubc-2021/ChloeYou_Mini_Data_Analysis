Mini Data Analysis Milestone 2
================

# Housekeeping Code

``` r
sg_copy<-readRDS("m1_data.RData")
library(datateachr) # <- might contain the data you picked!
library(tidyverse)
library(ggrepel)
```

# Task 1: Process and summarize your data (15 points)

### 1.1 Research Questions (2.5 points)

1.  Which publisher/developer produces the most games and receives the most positive reviews? We can see how much of the market a publisher takes up in the market and also gauge player sentiment towards a developer from these metrics.

2.  Do some publishers get more positive reviews? Which ones? Some studios recieve better public sentiment. My hypothesis is that the bigger publishers tend to get more mixed reviews (ratio of reviews that are positive lean towards 50%); on the other hand gamers are more forgiving towards smaller developers/publishers and hence will have more positive reviews.

3.  What's the difference between genres in terms of the average and total volume of reviews and average proportion of positive reviews of each genre. The popular genres such as Action, Adventure, RPG, Massive Multipler. What's the difference between genres in terms of the average and total volume of reviews and average proportion of positive reviews of each genre.

4.  Use logistic regression with player sentiment as outcome variable and see if we can interpret the coefficients of the model. Interpreations of the coefficients can help us quantify how each factor contributes to player reviews.

### 1.2 Summarizing and Graphing (10 points)

#### Research Question 1

Which publishers produces the most games and receives the most positive reviews?

``` r
# Tidy data: remove repetitive publishers names for each observation. Create a long table that if a game has multiple publishers, there will be multiple observations each having one unique publisher name for the same game. 
publishers <- sg_copy%>%
  select(id, publisher, ratio_of_postive_user_reviews,original_price) %>%
  group_by(id) %>%
  mutate(publisher = strsplit(as.character(publisher), ",")) %>% 
  unnest(publisher) %>% 
  drop_na() %>% # remove new observations with any NAs, could be caused by data tidying
  filter(!publisher %in% c(" Inc.", " LLC", " ltd.", " Ltd.", " LTD." )) %>%
  distinct() # return unique observations
```

##### Summarizing

Compute the range, mean, median, and standard deviation of ratio of positive user reviews, along with the number of games released on steam across the groups of publishers.

**Comment:** The min, max, mean, median, and standard deviation of positive review ratios for all games of a publisher tells us the overall distribution of player reviews for the publisher. By counting how much unique games the publisher has on Steam we are able to roughly gauge which publishers produce more games.

``` r
publisher_summary <- publishers %>% 
  group_by(publisher) %>%
  summarise(min = min(ratio_of_postive_user_reviews), # rating of the title with the least positive reviews received
            max = max(ratio_of_postive_user_reviews),
            mean = mean(ratio_of_postive_user_reviews),
            median = median(ratio_of_postive_user_reviews),
            sd = sd(ratio_of_postive_user_reviews), # spread
            games_on_steam = length(unique(id))) %>%
  arrange(desc( games_on_steam)) # order from the publisher that has the most games on Steam to least
```

##### Graphing

Create a graph out of summarized variables that has at least two geom layers.

**Comment:** For visualization, the top 50 publishers with the most amount of games on Steam will be plotted here to gauge if there are any publishers that have published many games and have also maintained a streak of positive reviews.

``` r
ggplot(publisher_summary[1:50,], aes(x= games_on_steam, y= mean, label = publisher)) +
  geom_point() +
  geom_smooth() +
  geom_label_repel(
    arrow = arrow(length = unit(0.03, "npc"), type = "closed", ends = "first"),
    force = 10) +
  ylim(0,1) + 
  xlab("Number of Games on Steam") + 
  ylab("Average Ratio of Positive Reviews")
```

    ## `geom_smooth()` using method = 'loess' and formula 'y ~ x'

![](mini-project-2_files/figure-markdown_github/unnamed-chunk-4-1.png)

Based on the scatterplot, we see that the top 10 publishers with the most amount of games on Steam seem to tend toward a mean of 75% positive reviews from their games, while a smaller publisher have a wider spread in terms of the average positive rates of games. Sekai Project and SCS Software are medium-to-large sized publishers and seem to have overall pretty positive reviews from most of their games.

#### Research Question 2

Do some publishers get more positive reviews? Which ones? \#\#\#\#\# Summarizing Create a categorical variable with 3 or more groups from an existing numerical variable. For the games that each publisher owns, based on positive reviews, we will bucket them into a categorical variable with groups: - Negative (0,25%\] - Neutral (25%, 50%\] - Positive (50%,75%\] - Very\_Positive (75%,100\]

**Comment:** It will be easier to visualize the differences between categorized player reviews and will be easier to use them as the outcome for building a logistic regression which is a type of model that is easier to interpret.

``` r
((publisher_categorical_review <- publishers %>% 
   mutate(category = case_when(ratio_of_postive_user_reviews < 0.25 ~ "negative",
                                 ratio_of_postive_user_reviews < 0.5~ "neutral",
                                ratio_of_postive_user_reviews < 0.75 ~ "positive",
                                 ratio_of_postive_user_reviews <= 1 ~ "very_positive"))%>%
  group_by(publisher, category) %>%
  summarise(n=n_distinct(id)) %>%
   filter(publisher %in% publisher_summary$publisher[1:20]) ## only look at the top 20 publishers on Steam
))
```

    ## `summarise()` has grouped output by 'publisher'. You can override using the `.groups` argument.

    ## # A tibble: 67 x 3
    ## # Groups:   publisher [20]
    ##    publisher        category          n
    ##    <chr>            <chr>         <int>
    ##  1 1C Entertainment neutral          10
    ##  2 1C Entertainment positive         38
    ##  3 1C Entertainment very_positive    37
    ##  4 2K               negative          1
    ##  5 2K               neutral          12
    ##  6 2K               positive         52
    ##  7 2K               very_positive    62
    ##  8 Activision       neutral          10
    ##  9 Activision       positive         25
    ## 10 Activision       very_positive    58
    ## # ... with 57 more rows

##### Graphing

Create a graph out of summarized variables that has at least two geom layers.

**Comment:** The summarized variables is the number of games for each publisher that falls into one of the categorized player review buckets. We use a stacked bar chart to show for the top 20 largest publishers on Steam, the proportion of titles that fall into each player review bucket.

``` r
ggplot(publisher_categorical_review, 
       aes(fill=category,y=n ,x=publisher)) +
   geom_bar(position="fill", stat="identity")+
  coord_flip() +
  ylab("proportion of games")
```

![](mini-project-2_files/figure-markdown_github/unnamed-chunk-6-1.png)

Sekai Project, Devolver Digital, Nightdive Studios have an overwhelming proportion of games that have very positive reviews.

#### Research Question 3

What's the difference between genres in terms of the average and total volume of reviews and average proportion of positive reviews of each genre.

##### Summarizing

Compute the number of observations within each genre. Based on prior knowledge, we choose the following 11 most popular genres and omit the other ones.

**Comment:** Computing the number of games that exist in each genre is helpful in understanding how big the genre is on Steam. Although this alone does not suffice to understand the reseach question we have in mind. Additional calculations of total reviews received in genre, average number of reviews received per title will be needed to answer the question. For now, I have sneaked in the calculation of average positive review ratio into the summary, along with the number of observations within each genre.

``` r
((genres_summary<- sg_copy %>%
  select(id, genre, ratio_of_postive_user_reviews, number_of_total_reviews) %>% 
  separate_rows(genre, sep= ",") %>% 
  filter(genre %in% c("Strategy","Sports","Simulation","RPG","Racing","Massively Multiplayer", "Indie", "Free to Play","Casual","Adventure","Action")) %>%
  group_by(genre) %>%
  summarise(number_of_titles = n_distinct(id),
            avg_ratio_of_postive_reviews = mean(ratio_of_postive_user_reviews, na.rm=T)
            ) %>%
    arrange(desc(number_of_titles))
))
```

    ## # A tibble: 11 x 3
    ##    genre                 number_of_titles avg_ratio_of_postive_reviews
    ##    <chr>                            <int>                        <dbl>
    ##  1 Indie                            22868                        0.757
    ##  2 Action                           15265                        0.743
    ##  3 Casual                           12010                        0.749
    ##  4 Adventure                        11871                        0.748
    ##  5 Simulation                        8711                        0.706
    ##  6 Strategy                          7975                        0.720
    ##  7 RPG                               6962                        0.746
    ##  8 Free to Play                      2620                        0.715
    ##  9 Sports                            1725                        0.718
    ## 10 Racing                            1384                        0.714
    ## 11 Massively Multiplayer             1277                        0.645

##### Graphing

Create a graph out of the summarized variable- number of observation by genre, that has at least two geom layers.

**Comment:** Number of games in genre is plotted against the average positive review ratio as a scatterplot.

``` r
ggplot(genres_summary, aes(number_of_titles, avg_ratio_of_postive_reviews, label= genre))+
  geom_point() +
  geom_label_repel(
    arrow = arrow(length = unit(0.02, "npc"), type = "closed", ends = "first"),
    force = 2,
    size = 2) +
  ylab("Averge Ratio of Positive Reviews") +
  xlab("Number of Games in Genre") +
  ylim(0,1)
```

![](mini-project-2_files/figure-markdown_github/unnamed-chunk-8-1.png)

There are less Massively Multiplayer games on Steam and they seem to have a lower average positive review ratio. While there are many Indie games and average reviews is 17% higher than the Massively Multiplayer genre.

#### Research Question 4:

Use logistic regression with player sentiment as outcome variable and see if we can interpret the coefficients of the model.

We continue to try and find if there are other variables in the dataset that contribute to players' sentiment towards a game. We now use the categorized game review that we obtained in Reseach Question \#2. We want to see if there are any correlation between player sentiment and the original pricing of the game.

##### Summary:

Compute the range, mean, and median, sd of original price across the groups of categorical player review defined in Research Question 2.

**Comment:** If there are any patterns in how a game is rated vs the original price, we might be able to use price as one of the explanatory variable in the logistic regression to predict player review category. We can see if the mean is significantly different for example.

``` r
publishers <- publishers %>% 
   mutate(category = case_when(ratio_of_postive_user_reviews < 0.25 ~ "negative",
                                 ratio_of_postive_user_reviews < 0.5~ "neutral",
                                ratio_of_postive_user_reviews < 0.75 ~ "positive",
                                 ratio_of_postive_user_reviews <= 1 ~ "very_positive"))  %>%
  group_by(category) %>% 
  filter(id != 18640, !is.na(category))  # this game id has original price equal to 650560 which is likely a typo
  
  
publishers %>%  
summarise(min= min(original_price),
            max= max(original_price),
            IQR= IQR(original_price),
            mean = mean(original_price),
            median = median(original_price),
            sd = sd(original_price),
            n=n_distinct(id))
```

    ## # A tibble: 4 x 8
    ##   category        min   max   IQR  mean median    sd     n
    ##   <chr>         <dbl> <dbl> <dbl> <dbl>  <dbl> <dbl> <int>
    ## 1 negative          0  111.   8.5  9.96   4.99  14.3   254
    ## 2 neutral           0  625.  13.0 14.1    4.99  46.4  1428
    ## 3 positive          0  625.  13   17.5    6.99  51.4  5113
    ## 4 very_positive     0  625.  12   15.2    7.99  43.3  9517

##### Graphing

Make a graph where it makes sense to customize the alpha transparency.

**Comment:** The density distrbution of original price by review category can help us see if there are any correlation between pricing and how the game is rated. Since there might be some overlapping of the chart for each category, we will need to adjust the alpha transparency. Also since the tail of the pricing is quite long but not many games have prices above 50, we will cap the price limit on the graph to 50 so we can identify differences easier.

``` r
# ggplot(publishers, aes(x=original_price, y= category, fill = category)) +
#   ggridges::geom_density_ridges(alpha = .2) 

# ggplot(publishers, aes(x=original_price, y= category, fill = category)) +
#   ggridges::geom_density_ridges(alpha = .2) + xlim(0,200)
ggplot(publishers, aes(x=original_price, y= category, fill = category)) +
   ggridges::geom_density_ridges(alpha = .2) + xlim(0,50)
```

    ## Picking joint bandwidth of 1.49

    ## Warning: Removed 611 rows containing non-finite values (stat_density_ridges).

![](mini-project-2_files/figure-markdown_github/unnamed-chunk-10-1.png)

Games that are rated negatively are less likely to be in the 25+ price range compared to the other groups. Surprisingly the peak of the negatively rated games is pricier than the peak of those games that are rated positively or neutrally. It will be interesting to see if it was rated poorly because the actual quality of the game doesn't match the price or some other reason.

### 1.3 (2.5 points)

The initial reseach goal is to understand the association between player sentiments(game ratings) and other variables such as genres, publishers, pricing. After some data exploration, we have identified some interesting trends in the data that addressed the above research goal.

Additionally, the research questions can be refined to a limited pool of publishers and genres, since there are an overwhelming amount of publishers and genres. If we focus on the games produced by the 20 largest publishers and the 11 most popular genres, then we might be able to find stronger signals in the data to help build a prediction model.

Since there are so many publishers in the dataset, it is probably a good idea to keep publisher out of the prediction model and simply look at data that is published by them for simplicity sake.

Research Question 3 & 4(Genre & Pricing of the game) is yielding interesting results. We saw that there are less Massively Multiplayer games on Steam and they seem to have a lower average positive review ratio. While there are many Indie games and average reviews is 17% higher than the Massively Multiplayer genre. We can continue to see if genres play a role in determining the average number of ratings received per title and also use the categorized player review instead of the numerical values. We also saw some differences in terms of the prices of the games that fall into each review buckets.

# Task 2: Tidy your data (12.5 points)

### 2.1 Explain Data Tidiness (2.5 points)

Select the columns that have been used or will be useful for the analysis.

``` r
# select 8 columns from dataset
games<- sg_copy %>%
  select(id, name, genre, publisher, original_price, release_date,ratio_of_postive_user_reviews,number_of_total_reviews)
```

The subsetted data is indeed tidy since it meets the following critera: 1. Each variable has its own column. 2. Each observation(game) has its own row. 3. Each value has its own cell.

### 2.2 Untidy and Tidy Data (5 points)

Now, if your data is tidy, untidy it! Then, tidy it back to it's original state.

If your data is untidy, then tidy it! Then, untidy it back to it's original state.

Be sure to explain your reasoning for this task. Show us the "before" and "after".

We will untidy the data by having the categories of Genres as variables and the rating as the value in the cells of that column, which will lead to a wide table with many NAs.

``` r
head(games) # Before
```

    ## # A tibble: 6 x 8
    ##      id name    genre    publisher  original_price release_date ratio_of_postiv~
    ##   <dbl> <chr>   <chr>    <chr>               <dbl> <chr>                   <dbl>
    ## 1     1 DOOM    Action   Bethesda ~           20.0 May 12, 2016             0.92
    ## 2     2 PLAYER~ Action,~ PUBG Corp~           30.0 Dec 21, 2017             0.49
    ## 3     3 BATTLE~ Action,~ Paradox I~           40.0 Apr 24, 2018             0.71
    ## 4     4 DayZ    Action,~ Bohemia I~           45.0 Dec 13, 2018             0.61
    ## 5     5 EVE On~ Action,~ CCP,CCP               0   May 6, 2003              0.74
    ## 6     7 Devil ~ Action   CAPCOM Co~           60.0 Mar 7, 2019              0.92
    ## # ... with 1 more variable: number_of_total_reviews <dbl>

``` r
untidy <- games %>% pivot_wider(id_cols = c(-genre, -ratio_of_postive_user_reviews),
                names_from = genre,
                values_from = ratio_of_postive_user_reviews)

head(untidy) # After
```

    ## # A tibble: 6 x 1,192
    ##      id name    publisher    original_price release_date number_of_total~ Action
    ##   <dbl> <chr>   <chr>                 <dbl> <chr>                   <dbl>  <dbl>
    ## 1     1 DOOM    Bethesda So~           20.0 May 12, 2016            42550   0.92
    ## 2     2 PLAYER~ PUBG Corpor~           30.0 Dec 21, 2017           836608  NA   
    ## 3     3 BATTLE~ Paradox Int~           40.0 Apr 24, 2018             7030  NA   
    ## 4     4 DayZ    Bohemia Int~           45.0 Dec 13, 2018           167115  NA   
    ## 5     5 EVE On~ CCP,CCP                 0   May 6, 2003             11481  NA   
    ## 6     7 Devil ~ CAPCOM Co.,~           60.0 Mar 7, 2019              9645   0.92
    ## # ... with 1,185 more variables: Action,Adventure,Massively Multiplayer <dbl>,
    ## #   Action,Adventure,Strategy <dbl>,
    ## #   Action,Free to Play,Massively Multiplayer,RPG,Strategy <dbl>,
    ## #   Adventure,Indie <dbl>, Strategy,Early Access <dbl>,
    ## #   Action,Adventure,RPG <dbl>, Adventure,Indie,RPG,Strategy <dbl>,
    ## #   Adventure <dbl>,
    ## #   Action,Adventure,Free to Play,Massively Multiplayer,RPG <dbl>, ...

Next we will tidy it back to it's original state which is having a variable called genre that holds the names of genre and a variable to store the ratio of postive user reviews.

``` r
tidy <- untidy %>% 
  pivot_longer(cols = c(-id, -name, -publisher, -original_price, -release_date,-number_of_total_reviews), 
               names_to  = "genre", 
               values_to = "ratio_of_postive_user_reviews",
              values_drop_na = TRUE)
head(tidy)
```

    ## # A tibble: 6 x 8
    ##      id name    publisher   original_price release_date number_of_total~ genre  
    ##   <dbl> <chr>   <chr>                <dbl> <chr>                   <dbl> <chr>  
    ## 1     1 DOOM    Bethesda S~           20.0 May 12, 2016            42550 Action 
    ## 2     2 PLAYER~ PUBG Corpo~           30.0 Dec 21, 2017           836608 Action~
    ## 3     3 BATTLE~ Paradox In~           40.0 Apr 24, 2018             7030 Action~
    ## 4     4 DayZ    Bohemia In~           45.0 Dec 13, 2018           167115 Action~
    ## 5     5 EVE On~ CCP,CCP                0   May 6, 2003             11481 Action~
    ## 6     7 Devil ~ CAPCOM Co.~           60.0 Mar 7, 2019              9645 Action 
    ## # ... with 1 more variable: ratio_of_postive_user_reviews <dbl>

### 2.3 Pick 2 Research Questions and Finalize Dataset(5 points)

I will continue to work on Research Questions 3 & 4. We can continue to see if genres play a role in determining the average number of ratings received per title and also use the categorized player review instead of the numerical values. We also saw some differences in terms of the prices of the games that fall into each review buckets. These variables can be helpful in creating the logistic regression mentioned in Research Question 4.

We will filter and clean the dataset to only contain games from the 11 most popular genres and from Steam's top 20 publishers (in terms of number of games published on Steam). We end up with a dataset with 3,069 observations and 9 variables.

``` r
## finalize dataset 
genre_list= 'Strategy|Sports|Simulation|RPG|Racing|Massively Multiplayer|Indie|Free to Play|Casual|Adventure|Action'
top20_publishers= 'Paradox Interactive|SEGA|Dovetail Games - Trains|Feral Interactive (Mac)|Square Enix|Ubisoft|2K|BANDAI NAMCO Entertainment|THQ Nordic|Feral Interactive|Activision|Degica|Devolver Digital|1C Entertainment|Deep Silver|Aspyr (Mac)|Nightdive Studios|Slitherine Ltd.|Focus Home Interactive|Sekai Project'
  

m2_end <- sg_copy %>%
  select(id, name, genre, publisher, original_price, release_date,ratio_of_postive_user_reviews,number_of_total_reviews) %>%  
  mutate(category = case_when(ratio_of_postive_user_reviews < 0.25 ~ "negative",
                                ratio_of_postive_user_reviews < 0.5~ "neutral",
                                ratio_of_postive_user_reviews < 0.75 ~ "positive",
                                 ratio_of_postive_user_reviews <= 1 ~ "very_positive")) %>%
  filter(grepl(top20_publishers, publisher)) %>% ## only look at the top 20 publishers on Steam
  filter(grepl(genre_list, genre)) %>% # restrict to games genre in this list
  filter(id != 18640)  # this game id has original price equal to 650560 which is likely a typo

glimpse(m2_end) # 3069 x 9
```

    ## Rows: 3,069
    ## Columns: 9
    ## $ id                            <dbl> 3, 14, 18, 21, 23, 24, 30, 34, 38, 41, 5~
    ## $ name                          <chr> "BATTLETECH", "Call of Duty®: Modern War~
    ## $ genre                         <chr> "Action,Adventure,Strategy", "Action", "~
    ## $ publisher                     <chr> "Paradox Interactive,Paradox Interactive~
    ## $ original_price                <dbl> 39.99, 1.02, 59.99, 1.02, 39.99, 19.99, ~
    ## $ release_date                  <chr> "Apr 24, 2018", "Jul 27, 2017", "Feb 7, ~
    ## $ ratio_of_postive_user_reviews <dbl> 0.71, 0.51, 0.77, 0.84, 0.70, 0.90, 0.81~
    ## $ number_of_total_reviews       <dbl> 7030, 1118, 1945, 4190, 487, 9757, 2701,~
    ## $ category                      <chr> "positive", "positive", "very_positive",~

``` r
saveRDS(m2_end, "m2_end.RData")
```

### Attribution

Thanks to Victor Yuan for mostly putting this together.
