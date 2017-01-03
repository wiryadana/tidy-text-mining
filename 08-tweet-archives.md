# Case study: comparing Twitter archives {#twitter}



One type of text that has risen to attention in recent years is text shared online via Twitter. In fact, several of the sentiment lexicons used in this book (and commonly used in general) were designed for use with and validated on tweets. Both of the authors of this book are on Twitter and are fairly regular users of it so in this case study, let's compare the entire Twitter archives of [Julia](https://twitter.com/juliasilge) and [David](https://twitter.com/drob).

TODO: download Twitter archives again

## Getting the data and distribution of tweets

An individual can download their own Twitter archive by following [directions available on Twitter's website](https://support.twitter.com/articles/20170160). We each downloaded ours and will now open them up. Let's use the lubridate package to convert the string timestamps to date-time objects and initially take a look at our tweeting patterns overall.


```r
library(lubridate)
library(ggplot2)
library(dplyr)
library(readr)

tweets_julia <- read_csv("data/tweets_julia.csv")
tweets_dave <- read_csv("data/tweets_dave.csv")
tweets <- bind_rows(tweets_julia %>% 
                      mutate(person = "Julia"),
                    tweets_dave %>% 
                      mutate(person = "David")) %>%
  mutate(timestamp = ymd_hms(timestamp))

ggplot(tweets, aes(x = timestamp, fill = person)) +
  geom_histogram(alpha = 0.5, position = "identity")
```

<img src="08-tweet-archives_files/figure-html/setup-1.png" width="768" />

David and Julia tweet at about the same rate currently and joined Twitter about a year apart from each other, but there were about 5 years where David was not active on Twitter and Julia was. In total, Julia has about 4 times as many tweets as David.

## Word frequencies

Let's use `unnest_tokens` to make a tidy dataframe of all the words in our tweets, and remove the common English stop words. There are certain conventions in how people use text on Twitter, so we will do a bit more work with our text here than, for example, we did with the narrative text from Project Gutenberg. The first `mutate` line below removes links and cleans out some characters that we don't want like ampersands. In the call to `unnest_tokens`, we unnest using a regex pattern, instead of just looking for single unigrams (words). This regex pattern is very useful for dealing with Twitter text; it retains hashtags and mentions of usernames with the `@` symbol. Because we have kept these types of symbols in the text, we can't use a simple `anti_join` to remove stop words. Instead, we can take the approach shown in the `filter` line that uses `str_detect` from the stringr library.


```r
library(tidytext)
library(stringr)

reg <- "([^A-Za-z_\\d#@']|'(?![A-Za-z_\\d#@]))"
tidy_tweets <- tweets %>% 
  mutate(text = str_replace_all(text, "https://t.co/[A-Za-z\\d]+|http://[A-Za-z\\d]+|&amp;|&lt;|&gt;|RT|https", "")) %>%
  unnest_tokens(word, text, token = "regex", pattern = reg) %>%
  filter(!word %in% stop_words$word,
         str_detect(word, "[a-z]"))
```

Now we can calculate word frequencies for each person. First, we group by person and count how many times each person used each word. Then we use `left_join` to add a column of the total number of words used by each person. (This is higher for Julia than David since she has more tweets than David.) Finally, we calculate a frequency for each person and word.


```r
frequency <- tidy_tweets %>% 
  group_by(person) %>% 
  count(word, sort = TRUE) %>% 
  left_join(tidy_tweets %>% 
              group_by(person) %>% 
              summarise(total = n())) %>%
  mutate(freq = n/total)

frequency
```

```
## Source: local data frame [23,084 x 5]
## Groups: person [2]
## 
##    person           word     n total        freq
##     <chr>          <chr> <int> <int>       <dbl>
## 1   Julia           time   567 76439 0.007417679
## 2   Julia    @selkie1970   565 76439 0.007391515
## 3   Julia       @skedman   518 76439 0.006776645
## 4   Julia            day   470 76439 0.006148694
## 5   Julia           baby   410 76439 0.005363754
## 6   David        #rstats   359 21984 0.016330058
## 7   Julia     @doctormac   342 76439 0.004474156
## 8   David @hadleywickham   306 21984 0.013919214
## 9   Julia           love   303 76439 0.003963945
## 10  Julia   @haleynburke   291 76439 0.003806957
## # ... with 23,074 more rows
```

This is a nice, tidy data frame but we would actually like to plot those frequencies on the x- and y-axes of a plot, so we will need to use `spread` from tidyr make a differently shaped dataframe.


```r
library(tidyr)

frequency <- frequency %>% 
  select(person, word, freq) %>% 
  spread(person, freq) %>%
  arrange(Julia, David)

frequency
```

```
## # A tibble: 19,565 × 3
##              word        David        Julia
##             <chr>        <dbl>        <dbl>
## 1    @_aaronmiles 4.548763e-05 1.308233e-05
## 2     @_cingraham 4.548763e-05 1.308233e-05
## 3      @amandacox 4.548763e-05 1.308233e-05
## 4      @antoviral 4.548763e-05 1.308233e-05
## 5    @astrobiased 4.548763e-05 1.308233e-05
## 6     @astrokatie 4.548763e-05 1.308233e-05
## 7           @b0rk 4.548763e-05 1.308233e-05
## 8      @benhamner 4.548763e-05 1.308233e-05
## 9       @calli993 4.548763e-05 1.308233e-05
## 10 @clarecorthell 4.548763e-05 1.308233e-05
## # ... with 19,555 more rows
```

Now this is ready for us to plot. Let's use `geom_jitter` so that we don't see the discreteness at the low end of frequency as much.


```r
library(scales)

ggplot(frequency, aes(Julia, David)) +
  geom_jitter(alpha = 0.1, size = 2.5, width = 0.25, height = 0.25) +
  geom_text(aes(label = word), check_overlap = TRUE, vjust = 1.5) +
  scale_x_log10(labels = percent_format()) +
  scale_y_log10(labels = percent_format()) +
  geom_abline(color = "red")
```

<img src="08-tweet-archives_files/figure-html/unnamed-chunk-2-1.png" width="672" />


This may not even need to be pointed out, but David and Julia have used their Twitter accounts rather differently over the course of the past several years. David has used his Twitter account almost exclusively for professional purposes since he became more active, while Julia used it for entirely personal purposes until late 2015. We see these differences immediately in this plot exploring word frequencies, and they will continue to be obvious in the rest of this chapter. Words near the red line in this plot are used with about equal frequencies by David and Julia, while words far away from the line are used much more by one person compared to the other. Words, hashtags, and usernames that appear in this plot are ones that we have both used at least once in tweets or retweets.

## Comparing word usage 

We just made a plot comparing raw word frequencies over our whole Twitter histories; now let's find which words are more or less likely to come from each person's account using the log odds ratio. First, let's restrict the analysis moving forward to tweets from David and Julia sent during 2016. David was consistently active on Twitter for all of 2016 and this was about when Julia transitioned into data science as a career.


```r
tidy_tweets <- tidy_tweets %>%
  filter(timestamp >= as.Date("2016-01-01"))
```

Next, let's use `str_detect` to remove Twitter usernames from the `word` column, because otherwise, the results here are dominated only by people who Julia or David know and the other does not. After removing these, we count how many times each person uses each word and keep only the words used more than 10 times. After a `spread` operation, we can calculate the log odds ratio for each word, using


$$\text{log odds ratio} = \ln{\left(\frac{\left[\frac{n+1}{\text{total}+1}\right]_\text{David}}{\left[\frac{n+1}{\text{total}+1}\right]_\text{Julia}}\right)}$$

where $n$ is the number of times the word in question is used by each person and the total indicates the total words for each person.


```r
word_ratios <- tidy_tweets %>%
  filter(!str_detect(word, "^@")) %>%
  count(word, person) %>%
  filter(sum(n) >= 10) %>%
  spread(person, n, fill = 0) %>%
  ungroup() %>%
  mutate_each(funs((. + 1) / sum(. + 1)), -word) %>%
  mutate(logratio = log(David / Julia)) %>%
  arrange(desc(logratio))
```

What are some words that are about equally likely to come from David or Julia's account?


```r
word_ratios %>% 
  arrange(abs(logratio))
```

```
## # A tibble: 275 × 4
##           word       David       Julia     logratio
##          <chr>       <dbl>       <dbl>        <dbl>
## 1         nice 0.005979499 0.005938634  0.006857602
## 2     hamilton 0.001993166 0.001979545  0.006857602
## 3        table 0.001993166 0.001979545  0.006857602
## 4     tomorrow 0.001993166 0.001979545  0.006857602
## 5         word 0.002562642 0.002639393 -0.029510042
## 6          yep 0.002562642 0.002639393 -0.029510042
## 7         true 0.003416856 0.003299241  0.035028479
## 8        words 0.003416856 0.003299241  0.035028479
## 9  programming 0.002847380 0.002969317 -0.041932562
## 10      python 0.002847380 0.002969317 -0.041932562
## # ... with 265 more rows
```

We are about equally likely to tweet about programming, words, and the Broadway musical *Hamilton*.

Which words are most likely to be from Julia's account or from David's account? Let's just take the top 15 most distinctive words for each account and plot them.


```r
word_ratios %>%
  group_by(logratio < 0) %>%
  top_n(15, abs(logratio)) %>%
  ungroup() %>%
  mutate(word = reorder(word, logratio)) %>%
  ggplot(aes(word, logratio, fill = logratio < 0)) +
  geom_bar(alpha = 0.8, stat = "identity") +
  coord_flip() +
  ylab("log odds ratio (David/Julia)") +
  scale_fill_discrete(name = "", labels = c("David", "Julia"))
```

<img src="08-tweet-archives_files/figure-html/plotratios-1.png" width="672" />

So David has tweeted about matrices, models, and his R packages like broom and gganimate while Julia tweeted about Salt Lake City, the U.S. Census, and maps.

## Changes in word use

REDO THIS SECTION WITH UPDATED ARCHIVES

The section above looked at overall word use, but now let's ask a different question. Which words' frequencies have changed the fastest in our Twitter feeds? Or to state this another way, which words have we tweeted about at a higher or lower rate as time has passed? To do this, we will define a new time variable in the dataframe that defines which unit of time each tweet was posted in. We can use `floor_date()` from lubridate to do this, with a unit of our choosing; using 15 days seems to work well for this year of tweets from both of us.

After we have the time bins defined, we count how many times each of us used each word in each time bin. After that, we add columns to the dataframe for the total number of words used in each time bin by each person and the total number of times each word was used by each person. We can then `filter` to only keep words used at least some minimum number of times (30, in this case).


```r
words_by_time <- tidy_tweets %>%
  filter(!str_detect(word, "^@")) %>%
  mutate(time_floor = floor_date(timestamp, unit = "15 days")) %>%
  count(time_floor, person, word) %>%
  ungroup() %>%
  group_by(person, time_floor) %>%
  mutate(time_total = sum(n)) %>%
  group_by(word) %>%
  mutate(word_total = sum(n)) %>%
  ungroup() %>%
  rename(count = n) %>%
  filter(word_total > 30)

words_by_time
```

```
## # A tibble: 719 × 6
##    time_floor person    word count time_total word_total
##        <dttm>  <chr>   <chr> <int>      <int>      <int>
## 1  2016-01-01  David #rstats     4        243        385
## 2  2016-01-01  David   broom     1        243         37
## 3  2016-01-01  David    cran     1        243         35
## 4  2016-01-01  David    data     2        243        251
## 5  2016-01-01  David     day     2        243         53
## 6  2016-01-01  David    feel     1        243         36
## 7  2016-01-01  David ggplot2     1        243         55
## 8  2016-01-01  David     job     2        243         33
## 9  2016-01-01  David    love     1        243         52
## 10 2016-01-01  David  people     1        243         80
## # ... with 709 more rows
```

Each row in this dataframe corresponds to one person using one word in a given time bin. The `count` column tells us how many times that person used that word in that time bin, the `time_total` column tells us how many words that person used during that time bin, and the `word_total` column tells us how many times that person used that word over the whole year. This is the data set we can use for modeling. 

We can use `nest` from tidyr to make a data frame with a list column that contains little miniature data frames for each word. Let's do that now and take a look at the resulting structure.


```r
nested_models <- words_by_time %>%
  nest(-word, -person) 

nested_models
```

```
## # A tibble: 70 × 3
##    person    word              data
##     <chr>   <chr>            <list>
## 1   David #rstats <tibble [18 × 4]>
## 2   David   broom <tibble [15 × 4]>
## 3   David    cran <tibble [11 × 4]>
## 4   David    data <tibble [15 × 4]>
## 5   David     day  <tibble [9 × 4]>
## 6   David    feel  <tibble [8 × 4]>
## 7   David ggplot2 <tibble [13 × 4]>
## 8   David     job  <tibble [8 × 4]>
## 9   David    love <tibble [11 × 4]>
## 10  David  people <tibble [13 × 4]>
## # ... with 60 more rows
```

This data frame has one row for each person-word combination; the `data` column is a list column that contains data frames, one for each combination of person and word. Let's use `map` from the purrr library to apply our modeling procedure to each of those little data frames inside our big data frame. This is count data so let’s use `glm` with `family = "binomial"` for modeling. We can think about this modeling procedure answering a question like, "Was a given word mentioned in a given time bin? Yes or no? How does the count of word mentions depend on time?"


```r
library(purrr)

nested_models <- nested_models %>%
  mutate(models = map(data, ~ glm(cbind(count, time_total) ~ time_floor, ., 
                                  family = "binomial")))

nested_models
```

Now notice that we have a new column for the modeling results; it is another list column and contains `glm` objects. The next step is to use `map` and `tidy` from the broom package to pull out the slopes for each of these models and find the important ones. We are comparing many slopes here and some of them are not statistically significant, so let's apply an adjustment to the p-values for multiple comparisons.


```r
library(broom)

slopes <- nested_models %>%
  unnest(map(models, tidy)) %>%
  filter(term == "time_floor") %>%
  mutate(adjusted.p.value = p.adjust(p.value))
```

Now let's find the most important slopes. Which words have changed in frequency at a significant level in our tweets?


```r
top_slopes <- slopes %>% 
  filter(adjusted.p.value < 0.05)

top_slopes
```

To visualize our results, we can plot these words' use for both David and Julia over this year of tweets.


```r
words_by_time %>%
  inner_join(top_slopes, by = c("word", "person")) %>%
  ggplot(aes(time_floor, count/time_total, color = word)) +
  facet_wrap(~ person, nrow = 2) +
  geom_line(alpha = 0.8, size = 1.3) +
  labs(x = NULL, y = "Word frequency",
       title = "Trending words in our tweets")
```


## Favorites and retweets

Another important characteristic of tweets is how many times they are favorited or retweeted. Let's explore which words are more likely to be retweeted or favorited for Julia's and David's tweets. When a user downloads their own Twitter archive, favorites and retweets are not included, so we constructed another dataset of the author's tweets that includes this information. We accessed our own tweets via the Twitter API and downloaded about 3200 tweets for each person. In both cases, that is about the last 18 months worth of Twitter activity. This corresponds to a period of increasing activity and increasing numbers of followers for both of us.


```r
tweets_julia <- read_csv("data/juliasilge_tweets.csv")
tweets_dave <- read_csv("data/drob_tweets.csv")
tweets <- bind_rows(tweets_julia %>% 
                      mutate(person = "Julia"),
                    tweets_dave %>% 
                      mutate(person = "David")) %>%
  mutate(created_at = ymd_hms(created_at))
```

Now that we have this second, smaller set of only recent tweets, let's use `unnest_tokens` to transform these tweets to a tidy data set. (Let's also remove any retweets from this data set so we only look at tweets that David and Julia have posted directly.)


```r
tidy_tweets <- tweets %>% 
  filter(!str_detect(text, "^RT")) %>%
  mutate(text = str_replace_all(text, "https://t.co/[A-Za-z\\d]+|http://[A-Za-z\\d]+|&amp;|&lt;|&gt;|RT|https", "")) %>%
  unnest_tokens(word, text, token = "regex", pattern = reg) %>%
  anti_join(stop_words)

tidy_tweets
```

```
## # A tibble: 30,725 × 7
##              id          created_at             source retweets favorites person        word
##           <dbl>              <dttm>              <chr>    <int>     <int>  <chr>       <chr>
## 1  8.043967e+17 2016-12-01 18:48:07 Twitter Web Client        4         6  David         j's
## 2  8.043611e+17 2016-12-01 16:26:39 Twitter Web Client        8        12  David   bangalore
## 3  8.043611e+17 2016-12-01 16:26:39 Twitter Web Client        8        12  David      london
## 4  8.043435e+17 2016-12-01 15:16:48 Twitter for iPhone        0         1  David @rodneyfort
## 5  8.043120e+17 2016-12-01 13:11:37 Twitter for iPhone        0         1  Julia         sho
## 6  8.040632e+17 2016-11-30 20:43:03 Twitter Web Client        0         2  David       arbor
## 7  8.040632e+17 2016-11-30 20:43:03 Twitter Web Client        0         2  David       arbor
## 8  8.040632e+17 2016-11-30 20:43:03 Twitter Web Client        0         2  David         ann
## 9  8.040632e+17 2016-11-30 20:43:03 Twitter Web Client        0         2  David         ann
## 10 8.040582e+17 2016-11-30 20:23:14 Twitter Web Client       30        41  David          sf
## # ... with 30,715 more rows
```

To start with, let's look at retweets. Let's find the total number of retweets for each person.


```r
totals <- tidy_tweets %>% 
  group_by(person, id) %>% 
  summarise(rts = sum(retweets)) %>% 
  group_by(person) %>% 
  summarise(total_rts = sum(rts))

totals
```

```
## # A tibble: 2 × 2
##   person total_rts
##    <chr>     <int>
## 1  David    111863
## 2  Julia     12906
```

Now let's find the median number of retweets for each word and person; we probably want to count each tweet/word combination only once, so we will use `group_by` and `summarise` twice, one right after the other. Next, we can join this to the data frame of retweet totals.


```r
word_by_rts <- tidy_tweets %>% 
  group_by(id, word, person) %>% 
  summarise(rts = first(retweets)) %>% 
  group_by(person, word) %>% 
  summarise(retweets = median(rts)) %>%
  left_join(totals) %>%
  filter(retweets != 0) %>%
  ungroup()

word_by_rts %>% 
  arrange(desc(retweets))
```

```
## # A tibble: 2,422 × 4
##    person         word retweets total_rts
##     <chr>        <chr>    <dbl>     <int>
## 1   David      angrier   1757.0    111863
## 2   David     confirms    878.5    111863
## 3   David       writes    878.5    111863
## 4   David        voted    611.0    111863
## 5   David       county    534.0    111863
## 6   David      teacher    390.5    111863
## 7   David      command    344.0    111863
## 8   David       revert    344.0    111863
## 9   David  dimensional    320.5    111863
## 10  David eigenvectors    320.5    111863
## # ... with 2,412 more rows
```

At the top of this sorted data frame, we see David's tweet about [his blog post on Donald Trump's own tweets](http://varianceexplained.org/r/trump-tweets/) that went viral. A search tells us that this is the only time David has ever used the word "angrier" in his tweets, so that word has an extremely high median retweet rate.

<blockquote class="twitter-tweet" data-lang="en"><p lang="en" dir="ltr">New post: Analysis of Trump tweets confirms he writes only the angrier Android half <a href="https://t.co/HRr4yj30hx">https://t.co/HRr4yj30hx</a> <a href="https://twitter.com/hashtag/rstats?src=hash">#rstats</a> <a href="https://t.co/cmwdNeYSE7">pic.twitter.com/cmwdNeYSE7</a></p>&mdash; David Robinson (@drob) <a href="https://twitter.com/drob/status/763048283531055104">August 9, 2016</a></blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

Now we can plot the words that have contributed the most to each of our retweets.


```r
word_by_rts %>%
  mutate(ratio = retweets / total_rts) %>%  
  group_by(person) %>%
  top_n(10, ratio) %>%
  arrange(ratio) %>%
  mutate(word = factor(word, unique(word))) %>%
  ungroup() %>%
  ggplot(aes(word, ratio, fill = person)) +
  geom_bar(stat = "identity", alpha = 0.8, show.legend = FALSE) +
  facet_wrap(~ person, scales = "free", ncol = 2) +
  coord_flip() +
  scale_y_continuous(labels = percent_format()) +
  labs(x = NULL, 
       y = "proportion of total RTs due to each word",
       title = "Words with highest median retweets")
```

<img src="08-tweet-archives_files/figure-html/unnamed-chunk-6-1.png" width="960" />

We see more words from David's tweet about his Trump blog post, and words from Julia making announcements about blog posts and new package releases. These are some pretty good tweets; we can see why people retweeted them.

<blockquote class="twitter-tweet" data-lang="en"><p lang="en" dir="ltr">NEW POST: Mapping ghost sightings in Kentucky using Leaflet 👻👻👻 <a href="https://twitter.com/hashtag/rstats?src=hash">#rstats</a> <a href="https://t.co/rRFTSsaKWQ">https://t.co/rRFTSsaKWQ</a> <a href="https://t.co/codPf3gy6O">pic.twitter.com/codPf3gy6O</a></p>&mdash; Julia Silge (@juliasilge) <a href="https://twitter.com/juliasilge/status/761667180148641793">August 5, 2016</a></blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

<blockquote class="twitter-tweet" data-lang="en"><p lang="en" dir="ltr">Me: Git makes it easy to revert your local changes<br><br>Them: Great! So what command do I use?<br><br>Me: I said it was easy not that I knew how</p>&mdash; David Robinson (@drob) <a href="https://twitter.com/drob/status/770706647585095680">August 30, 2016</a></blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

We can follow a similar procedure to see which words led to more favorites. Are they different than the words that lead to more retweets?


```r
totals <- tidy_tweets %>% 
  group_by(person, id) %>% 
  summarise(favs = sum(favorites)) %>% 
  group_by(person) %>% 
  summarise(total_favs = sum(favs))

word_by_favs <- tidy_tweets %>% 
  group_by(id, word, person) %>% 
  summarise(favs = first(favorites)) %>% 
  group_by(person, word) %>% 
  summarise(favorites = median(favs)) %>%
  left_join(totals) %>%
  filter(favorites != 0) %>%
  ungroup()
```


```r
word_by_favs %>%
  mutate(ratio = favorites / total_favs) %>%  
  group_by(person) %>%
  top_n(10, ratio) %>%
  arrange(ratio) %>%
  mutate(word = factor(word, unique(word))) %>%
  ungroup() %>%
  ggplot(aes(word, ratio, fill = person)) +
  geom_bar(stat = "identity", alpha = 0.8, show.legend = FALSE) +
  facet_wrap(~ person, scales = "free", ncol = 2) +
  coord_flip() +
  scale_y_continuous(labels = percent_format()) +
  labs(x = NULL, 
       y = "proportion of total favorites due to each word",
       title = "Words with highest median favorites")
```

<img src="08-tweet-archives_files/figure-html/unnamed-chunk-7-1.png" width="960" />

We see some minor differences, especially near the bottom of the top 10 list, but these are largely the same words as for favorites. In general, the same words that lead to retweets lead to favorites. There are some exceptions, though.

<blockquote class="twitter-tweet" data-lang="en"><p lang="en" dir="ltr">🎶 I am writing a Shiny app for my joooooooob 🎶🎶 I am living the dreeeeeeeeeam... 🎶🎶 <a href="https://twitter.com/hashtag/rstats?src=hash">#rstats</a></p>&mdash; Julia Silge (@juliasilge) <a href="https://twitter.com/juliasilge/status/732645241610600448">May 17, 2016</a></blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>