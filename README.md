---
title: "Text Mining with R"
output:
  word_document: default
  html_document: default
---

Text Mining with R - A Tidy Approach
Julia Silge and David Robinson 
https://www.tidytextmining.com/

# Preface 
Tidytext (Silge and Robinson 2016) R package

## Outlines 
- Chapter 1: outlines the tidy text format and the `unnest_tokens()` function. It also introduces the gutenbergr and janaustenr packages, which provide useful literary text datasets that we'll use through this book.
- Chapter 2: shows how to perform sentiment analysis on a tidy text dataset, using `sentiments` dataset from tidytest and `inner_join` from dplyr.
- Chapter 3: describes the tf-idf statistic (term frequency times inverse document frequency), a quantity used for identifying terms that are especially important to a particular document.
- Chapter 4: introdues n-grams and how to analyze word networks in text using the windyr and ggraph packages

Text won't be tidy at all stages of analaysis, and it is important to be able to convert back and forth between tidy and non-tidy formats. 

- Chapter 5: introduces methods for tidying document-term matrices and corpus objects from the tm and quanteda packages, as well as for casting tidy text datasets into those formats. 
- Chapter 6: explores the concep of topic modeling, and uses the `tidy()`
 method to interpret and visualize the output of the topicmodels package.
 
 We conlude with several case studies that bring together multiple tidy text mining approaches we've learnt. 
 
- Chapter 7: demonstrates an application of a tidy text analysis by analyzing the author's own twitter archives. How do Dave's and Julia's tweeting habits compare? 
- Chapter 8: explores metadata from over 32,000 NASA datasets (available in JSON) by looking at how keywords from the datasets are conected to title and description fields.
- Chapter 9: analyzes a dataset of Usenet messages from a diverse set of newsgroups (focused on topics like politics, hocket, technology, atheism, and more) to understand patterns across the groups.

## Notes 
This Book does not cover following topics
- clustering, classification and prediction
- word embedding
- more complex tokenization
- language other than English

# Chapter 1: The tidy text format

Tidy text format - is a table with one-token-per-row

A token is a meaningful unit of text, such as a word
Tokenization is the process of splitting text into tokens 

Tidy datasets allow manipulation with a standard set of "tidy" tools, including popular packages such as dplyr, tidyr, ggplot2 and broom. By keeping the input and output in tidy tables, users can transition between these packages. 

At the same time, the tidytext package doesn't expect a user to keep text data in a tidy form at all times during an analysis. The packages includes functions to `tidy()` objects, such as tm and quanteda. This allows a workflow where importing, filtering and processing is done using dplyr and other tidy tools, after which the data is convered into a document-term matrix for machine learning applications. The models can the be re-converted into a tidy form for interpreation and visualization with ggplot2.


## 1.1 Contrasting tidy text with other data structure

we difine the tidy text format as being a table with one-token-per-row. Structuring text data in this way means that it conforms to tidy data principles and can be manipulated with a set of consistent tools.

- string: Text can be stored as strings (ie., character vectors)
- corpus: These types of objects typically contain raw strings annotated with additional metadata and details. 
- document-term matrix: sparse matrix describing a collection(ie. corpus) of documents with one row for each document and one column for each term. The value in the matrix is typically word count or tf-idf.

## 1.2 The `unnest_tokens` function

Emily Dickinson wrote some lovely text in her time
```{r}
text <- c("Because I could not stop for Death -",
          "He kindly stopped for me -",
          "The Carriage held but just Ourselves -",
          "and Immortality")
text
```

This is a typical character vector that we might want to analyze. In order to turn it into a tidy text dataset, we forst need to put it into a data frame.
```{r}
library(dplyr)
text_df <- data_frame(line=1:4,text=text)
text_df
```

What does it mean that this data frame has printed out as a "tibble"? A tibble is a modern class of data frame within R, available in the dplyr and tibble packages, that has a convenient print method, will not convert strings to factors, and does not use row names. Tibbles are great for use with tidy tools. 

Within our tidy text framework, we need to both break the text into individual tokens (a process called tokenization) and transform it to a tidy data structure. We use tidytext's  `unnest_tokens` function.
```{r unnest_tokens function}
# install.packages("tidytext")
library(tidytext)

# Ctl + Shift + M: %>% pipe operator
text_df
library(magrittr)
text_df$text

text_df %>% unnest_tokens(word,text)
```

The two basic arguments to `unnest_tokens` used here are column names. First, we have the output column name that will be created (`word`), and the input column that the text comes (`text`). 

After using `unnest_tokens`, we've split each row so that there is one token (word) in each row of the new data frame: the default tokenization in `unnest_tokens` is for single words, as shown here.  

## 1.3 Tidying the works of Jane Austen
We use the text of Jane Austen's six completed, published novels from the janaustenr package, and transform them into a tidy format, and also use `mutate()` to annotate a `linenumber` quantify to keep track of lines in the original format and a `chapter` to find where all the chapters are. 

```{r Tidy works of Jane Austen}
library(janeaustenr)
library(dplyr)
library(stringr)

original_books <- austen_books() %>% 
  group_by(book) %>% 
  mutate(linenumber=row_number(),
         chapter=cumsum(str_detect(text,regex("^chapter [\\divxlc]",
                                              ignore_case = T)))) %>% 
  ungroup()

head(original_books,12)
```

To work with this as a tidy dataset, we need to restructure it in the one-token-per-row format, which as we saw earlier is done with the `unnest_tokens` function.

```{r tidy_books unnest_tokens}
library(tidytext)
tidy_books <- original_books %>% 
  unnest_tokens(word,text) 
tidy_books
```

This function uses the `tokenizers` package to separate each line of text in the original data frame into tokens. The default tokenizing is for words, but other options include characters, n-grams, sentences, lines, paragraphs, or separation around a `regex pattern`.

Now that the data is in one-word-per-row format, we can manipulate it with tidy toos like dplyr. Often in text analysis, we will want to remove stop words: stop words are words that are not useful for an analysis, typically extremely common words such as "the", "of", "to" and so forth in English. 

We can remove stop words (kept in the tidytext dataset `stop_words`) with an `anti_join()`
```{r remove stop words}
# stop words dataset
data(stop_words)

# anti-join() application
tidy_books <- tidy_books %>% 
  anti_join(stop_words)
```

The `stop_words` dataset in the tidytext package contains stop words from three lexicons. We can use them all together, as we have here, or filter to only use one set of stop words.

We ca also use dplyr's count() to find the most common words in all the books as a whole.
```{r tidybooks count word}
tidy_books %>% 
  count(word,sort=TRUE)
```

Because we've been using tidy tools, our word counts are stored in a tidy data frame. This allows us to pipe this directly to the gpplot2 package, for example to create a visualization of the most common words 
```{r tidybooks visualization}
library(ggplot2)

tidy_books %>% 
  count(word,sort=T) %>% 
  filter(n>600) %>% 
  mutate(word=reorder(word,n)) %>% 
  ggplot(aes(word,n))+
  geom_col()+
  xlab(NULL)+
  coord_flip()
```

Note that the `auten_books()` function started us with exactly the text we wanted to analyze, but in other cases we may need to perform cleaning of text data, such as removing copy right headers or formattting. You'll examples of this kind of pre-processing in the case study chapters, especially Chapter 9.1.1.

## 1.4 The gutenbergr package

Now that we’ve used the janeaustenr package to explore tidying text, let’s introduce the gutenbergr package (Robinson 2016). The gutenbergr package provides access to the public domain works from the Project Gutenberg collection. The package includes tools both for downloading books (stripping out the unhelpful header/footer information), and a complete dataset of Project Gutenberg metadata that can be used to find works of interest. In this book, we will mostly use the function `gutenberg_download()` that downloads one or more works from Project Gutenberg by ID, but you can also use other functions to explore metadata, pair Gutenberg ID with title, author, language, etc., or gather information about authors.

https://ropensci.org/tutorials/gutenbergr_tutorial/

```{r}
# install.packages("gutenbergr")
library(gutenbergr)
library(dplyr)
library(magrittr)

gutenberg_metadata

# find Gutenberg ID
gutenberg_metadata %>% 
  filter(title=="Wuthering Heights")
```

In many analyses, you may want to filter just for English works, avoid duplicates, and 

## 1.5 Word frequencies

A common task in text mining is to look at word frequencies, just like we have done above for Jane Austen’s novels, and to compare frequencies across different texts. We can do this intuitively and smoothly using tidy data principles. We already have Jane Austen’s works; let’s get two more sets of texts to compare to. First, let’s look at some science fiction and fantasy novels by H.G. Wells, who lived in the late 19th and early 20th centuries. Let’s get The Time Machine, The War of the Worlds, The Invisible Man, and The Island of Doctor Moreau. We can access these works using `gutenberg_download()` and the Project Gutenberg ID numbers for each novel.

```{r}
library(dplyr)
library(magrittr)
library(tidyverse)
library(tidyr)
library(gutenbergr)
```


```{r 1.5 word frequency 1}
# install.packages("gutenbergr")
# hgwells <- gutenberg_download(c(35, 36, 5230, 159))
```

```{r 1.5 word frequency 2}
library(magrittr)
# tidy_hgwells <- hgwells %>% 
  # unnest_tokens(word,text) %>% 
  # anti_join(stop_words)
```

Just for kicks, what are the most common words in these novels of H.G. Wells?
```{r 1.5 word frequency 3}
# tidy_hgwells %>% 
  # count(word,sort=T)
```

Now let’s get some well-known works of the Brontë sisters, whose lives overlapped with Jane Austen’s somewhat but who wrote in a rather different style. Let’s get Jane Eyre, Wuthering Heights, The Tenant of Wildfell Hall, Villette, and Agnes Grey. We will again use the Project Gutenberg ID numbers for each novel and access the texts using gutenberg_download().
```{r word frequency 4}
# bronte <- gutenberg_download(c(1260, 768, 969, 9182, 767))

# tidy_bronte <- bronte %>% 
  # unnest_tokens(word,text) %>% 
  # anti_join(stop_words)
```

What are the most common words in these novels of the Brontë sisters?
```{r 1.5 word frequency 5}
# tidy_bronte %>% 
#   count(word,sort=T)
```

Interesting that “time”, “eyes”, and “hand” are in the top 10 for both H.G. Wells and the Brontë sisters.

Now, let’s calculate the frequency for each word for the works of Jane Austen, the Brontë sisters, and H.G. Wells by binding the data frames together. We can use spread and gather from tidyr to reshape our dataframe so that it is just what we need for plotting and comparing the three sets of novels.

```{r 1.5 word frequency 6}
# 
# library(tidyr)
# frequency <- bind_rows(mutate(tidy_bronte, author = "Brontë Sisters"),
#                        mutate(tidy_hgwells, author = "H.G. Wells"), 
#                        mutate(tidy_books, author = "Jane Austen")) %>% 
#   mutate(word = str_extract(word, "[a-z']+")) %>%
#   count(author, word) %>%
#   group_by(author) %>%
#   mutate(proportion = n / sum(n)) %>% 
#   select(-n) %>% 
#   spread(author, proportion) %>% 
#   gather(author, proportion, `Brontë Sisters`:`H.G. Wells`)```
```

We use `str_extract()` here because the UTF-8 encoded texts from Project Gutenberg have some examples of words with underscores around them to indicate emphasis (like italics). The tokenizer treated these as words, but we don’t want to count “_any_” separately from “any” as we saw in our initial data exploration before choosing to use `str_extract()`.

Now let’s plot (Figure 1.3).
```{r 1.5 word frequency 7}
# library(scales)
# # expect a warning about rows with missing values being removed
# ggplot(frequency, aes(x = proportion, y = `Jane Austen`, color = abs(`Jane Austen` - proportion))) +
#   geom_abline(color = "gray40", lty = 2) +
#   geom_jitter(alpha = 0.1, size = 2.5, width = 0.3, height = 0.3) +
#   geom_text(aes(label = word), check_overlap = TRUE, vjust = 1.5) +
#   scale_x_log10(labels = percent_format()) +
#   scale_y_log10(labels = percent_format()) +
#   scale_color_gradient(limits = c(0, 0.001), low = "darkslategray4", high = "gray75") +
#   facet_wrap(~author, ncol = 2) +
#   theme(legend.position="none") +
#   labs(y = "Jane Austen", x = NULL)
```

Words that are close to the line in these plots have similar frequencies in both sets of texts, for example, in both Austen and Brontë texts (“miss”, “time”, “day” at the upper frequency end) or in both Austen and Wells texts (“time”, “day”, “brother” at the high frequency end). Words that are far from the line are words that are found more in one set of texts than another. For example, in the Austen-Brontë panel, words like “elizabeth”, “emma”, and “fanny” (all proper nouns) are found in Austen’s texts but not much in the Brontë texts, while words like “arthur” and “dog” are found in the Brontë texts but not the Austen texts. In comparing H.G. Wells with Jane Austen, Wells uses words like “beast”, “guns”, “feet”, and “black” that Austen does not, while Austen uses words like “family”, “friend”, “letter”, and “dear” that Wells does not.

Overall, notice in Figure 1.3 that the words in the Austen-Brontë panel are closer to the zero-slope line than in the Austen-Wells panel. Also notice that the words extend to lower frequencies in the Austen-Brontë panel; there is empty space in the Austen-Wells panel at low frequency. These characteristics indicate that Austen and the Brontë sisters use more similar words than Austen and H.G. Wells. Also, we see that not all the words are found in all three sets of texts and there are fewer data points in the panel for Austen and H.G. Wells.

Let’s quantify how similar and different these sets of word frequencies are using a correlation test. How correlated are the word frequencies between Austen and the Brontë sisters, and between Austen and Wells?

```{r}
# cor.test(data=frequency[frequency$author == "Brontë Sisters",],
#          ~proportion+"Jane Austen")
# 
# cor.test(data = frequency[frequency$author == "H.G. Wells",], 
#          ~ proportion + `Jane Austen`)
```

Just as we saw in the plots, the word frequencies are more correlated between the Austen and Brontë novels than between Austen and H.G. Wells.

## 1.6 Summary
In this chapter, we explored what we mean by tidy data when it comes to text, and how tidy data principles can be applied to natural language processing. When text is organized in a format with one token per row, tasks like removing stop words or calculating word frequencies are natural applications of familiar operations within the tidy tool ecosystem. The one-token-per-row framework can be extended from single words to n-grams and other meaningful units of text, as well as to many other analysis priorities that we will consider in this book.

# Reference

Wickham, Hadley. 2014. “Tidy Data.” Journal of Statistical Software 59 (1): 1–23. doi:10.18637/jss.v059.i10.

Wickham, Hadley, and Romain Francois. 2016. dplyr: A Grammar of Data Manipulation. https://CRAN.R-project.org/package=dplyr.

Wickham, Hadley. 2016. tidyr: Easily Tidy Data with  
spread()and gather()
  Functions. https://CRAN.R-project.org/package=tidyr.

Wickham, Hadley. 2009. ggplot2: Elegant Graphics for Data Analysis. Springer-Verlag New York. http://ggplot2.org.

Robinson, David. 2017. broom: Convert Statistical Analysis Objects into Tidy Data Frames. https://CRAN.R-project.org/package=broom.

Ingo Feinerer, Kurt Hornik, and David Meyer. 2008. “Text Mining Infrastructure in R.” Journal of Statistical Software 25 (5): 1–54. http://www.jstatsoft.org/v25/i05/.

Benoit, Kenneth, and Paul Nulty. 2016. quanteda: Quantitative Analysis of Textual Data. https://CRAN.R-project.org/package=quanteda.

Silge, Julia. 2016. janeaustenr: Jane Austen’s Complete Novels. https://CRAN.R-project.org/package=janeaustenr.

Robinson, David. 2016. gutenbergr: Download and Process Public Domain Works from Project Gutenberg. https://cran.rstudio.com/package=gutenbergr.

# Chapter 2: Sentiment analysis with tidy data
In the previous chapter, we explored in depth what we mean by the tidy text format and showed how this format can be used to approach questions about word frequency. This allowed us to analyze which words are used most frequently in documents and to compare documents, but now let’s investigate a different topic. Let’s address the topic of opinion mining or sentiment analysis. When human readers approach a text, we use our understanding of the emotional intent of words to infer whether a section of text is positive or negative, or perhaps characterized by some other more nuanced emotion like surprise or disgust. We can use the tools of text mining to approach the emotional content of text programmatically, as shown in Figure 2.1.

One way to analyze the sentiment of a text is to consider the text as a combination of its individual words and the sentiment content of the whole text as the sum of the sentiment content of the individual words. This isn’t the only way to approach sentiment analysis, but it is an often-used approach, and an approach that naturally takes advantage of the tidy tool ecosystem

## 2.1 The `sentiment` dataset
As discussed above, there are a variety of methods and dictionaries that exist for evaluating the opinion or emotion in text. The tidtext package contains severail sentiment lexicons in the several sentiment lexicons in the `sentiments` dataset.
```{r}
library(tidytext)
sentiments
```

The three greneral-purpose lexicons are
- `AFINN` from from Finn Årup Nielsen,
- `bing`Bing Liu and collaborators, and
- `nrc` from Saif Mohammad and Peter Turney.

All three of these lexicons are based on unigrams, i.e., single words. These lexicons contain many English words and the words are assigned scores for positive/negative sentiment, and also possibly emotions like joy, anger, sadness, and so forth. 

The `nrc` lexicon categorizes words in a binary fashion (“yes”/“no”) into categories of positive, negative, anger, anticipation, disgust, fear, joy, sadness, surprise, and trust. The `bing` lexicon categorizes words in a binary fashion into positive and negative categories. The `AFINN` lexicon assigns words with a score that runs between -5 and 5, with negative scores indicating negative sentiment and positive scores indicating positive sentiment. All of this information is tabulated in the sentiments dataset, and tidytext provides a function `get_sentiments()` to get specific sentiment lexicons without the columns that are not used in that lexicon.
```{r}
get_sentiments("afinn")
```

```{r}
get_sentiments("bing")
```

```{r}
get_sentiments("nrc")
```

How were these sentiment lexicons put together and validated? They were constructed via either crowdsourcing (using, for example, Amazon Mechanical Turk) or by the labor of one of the authors, and were validated using some combination of crowdsourcing again, restaurant or movie reviews, or Twitter data. 

Given this information, we may hesitate to apply these sentiment lexicons to styles of text dramatically different from what they were validated on, such as narrative fiction from 200 years ago. While it is true that using these sentiment lexicons with, for example, Jane Austen’s novels may give us less accurate results than with tweets sent by a contemporary writer, we still can measure the sentiment content for words that are shared across the lexicon and the text.

here are also some domain-specific sentiment lexicons available, constructed to be used with text from a specific content area. Section 5.3.1 explores an analysis using a sentiment lexicon specifically for finance.

*Dictionary-based methods like the ones we are discussing find the total sentiment of a piece of text by adding up the individual sentiment scores for each word in the text.

Not every English word is in the lexicons because many English words are pretty neutral. It is important to keep in mind that these methods do not take into account qualifiers before a word, such as in “no good” or “not true”; a lexicon-based method like this is based on unigrams only. For many kinds of text (like the narrative examples below), there are not sustained sections of sarcasm or negated text, so this is not an important effect. Also, we can use a tidy text approach to begin to understand what kinds of negation words are important in a given text; see Chapter 9 for an extended example of such an analysis.

One last caveat is that the size of the chunk of text that we use to add up unigram sentiment scores can have an effect on an analysis. A text the size of many paragraphs can often have positive and negative sentiment averaged out to about zero, while sentence-sized or paragraph-sized text often works better.

## 2.2 Sentiment analysis with inner join
With data in a tidy format, sentiment analysis can be done as an inner join. This is another of the great successes of viewing text mining as a tidy data analysis task; much as removing stop words is an antijoin operation, performing sentiment analysis is an inner join operation.

Let’s look at the words with a joy score from the NRC lexicon. What are the most common joy words in Emma? First, we need to take the text of the novels and convert the text to the tidy format using `unnest_tokens()`, just as we did in Section 1.3. Let’s also set up some other columns to keep track of which line and chapter of the book each word comes from; we use group_by and mutate to construct those columns.
```{r 2.2 Sentiment analysis 1}
# install.packages("janeaustenr")
library(janeaustenr)
library(dplyr)
library(stringr)

# install.packages("tidytext")
library(tidytext) #unnest_tokens()

tidy_books <- austen_books() %>% 
  group_by(book) %>% 
  mutate(linenumber = row_number(),
         chapter=cumsum(str_detect(text,regex("^chapter [\\divxlc]",
                                              ignore_case = T)))) %>% 
  ungroup() %>% 
  unnest_tokens(word,text)
```

Notice that we chose the name word for the output column from `unnest_tokens()`. This is a convenient choice because the sentiment lexicons and stop word datasets have columns named `word`; performing `inner joins` and `anti-joins` is thus easier.

Now that the text is in a tidy format with one word per row, we are ready to do the sentiment analysis. First, let’s use the NRC lexicon and `filter()` for the joy words. Next, let’s `filter()` the data frame with the text from the books for the words from Emma and then use `inner_join()` to perform the sentiment analysis. What are the most common joy words in Emma? Let’s use `count()` from `dplyr`.

```{r 2.2 Sentiment analysis 2}
nrc_joy <- get_sentiments("nrc") %>% 
  filter(sentiment=="joy")

tidy_books %>% 
  filter(book=="Emma") %>% 
  inner_join(nrc_joy) %>% 
  count(word,sort=T)
```

We see mostly positive, happy words about hope, friendship, and love here. We also see some words that may not be used joyfully by Austen (“found”, “present”); we will discuss this in more detail in Section 2.4.

We can also examine how sentiment changes throughout each novel. We can do this with just a handful of lines that are mostly dplyr functions. First, we find a sentiment score for each word using the Bing lexicon and `inner_join()`.

The %/% operator does integer division (x %/% y is equivalent to floor(x/y)) so the index keeps track of which 80-line section of text we are counting up negative and positive sentiment in.

Small sections of text may not have enough words in them to get a good estimate of sentiment while really large sections can wash out narrative structure. For these books, using 80 lines works well, but this can vary depending on individual texts, how long the lines were to start with, etc. We then use `spread()` so that we have negative and positive sentiment in separate columns, and lastly calculate a net sentiment (positive - negative).

```{r 2.2 Sentiment analysis 3}
library(tidyr)

jane_austen_sentiment <- tidy_books %>% 
  inner_join(get_sentiments("bing")) %>% 
  count(book,index=linenumber %/% 80, sentiment) %>%   
  spread(sentiment,n,fill=0) %>% 
  mutate(sentiment=positive-negative)
```

Now we can plot these sentiment scores across the plot trajectory of each novel. Notice that we are plotting against the `index` on the x-axis that keeps track of narrative time in sections of text.
```{r 2.2 Sentiment analysis 4}
library(ggplot2)
ggplot(jane_austen_sentiment,aes(index,sentiment,fill=book))+
  geom_col(show.legend=FALSE)+
  facet_wrap(~book,ncol=2,scales="free_x")
```

We can see in Figure 2.2 how the plot of each novel changes toward more positive or negative sentiment over the trajectory of the story.

## 2.3 Comparing the three sentiment dictionaries

With several options for sentiment lexicons, you might want some more information on which one is appropriate for your purposes. Let’s use all three sentiment lexicons and examine how the sentiment changes across the narrative arc of Pride and Prejudice. First, let’s use `filter()` to choose only the words from the one novel we are interested in.

```{r 2.3 Comparing three dictionaries 1}
tidy_books %>% 
  dplyr::distinct(book)

pride_prejudice <- tidy_books %>% 
  filter(book == "Pride & Prejudice")
pride_prejudice
```

Now, we can use `inner_join()` to calculate the sentiment in different ways. 

> Remeber from above that the AFINN lexicon measures sentiment with a numeric score between -5 and 5, while the other two lexicons categorize words in a binary fashion, either positive or negative. To find a sentiment score in chunks of text throughout the novel, we will need to use a different pattern for the AFINN lexicon than for the other two.

Let’s again use integer division (`%/%`) to define larger sections of text that span multiple lines, and we can use the same pattern with `count()`, `spread()`, and `mutate()` to find the net sentiment in each of these sections of text.
```{r Comparing three dictionaries 2}
afinn <- pride_prejudice %>% 
  inner_join(get_sentiments("afinn")) %>% 
  group_by(index = linenumber %/% 80) %>% 
  summarise(sentiment = sum(score)) %>% 
  mutate(method = "AFINN")

bing_and_nrc <- bind_rows(pride_prejudice %>% 
                            inner_join(get_sentiments("bing")) %>%
                            mutate(method = "Bing et al."),
                          pride_prejudice %>% 
                            inner_join(get_sentiments("nrc") %>% 
                                         filter(sentiment %in% c("positive", 
                                                                 "negative"))) %>%
                            mutate(method = "NRC")) %>%
  count(method, index = linenumber %/% 80, sentiment) %>%
  spread(sentiment, n, fill = 0) %>%
  mutate(sentiment = positive - negative)
```

We now have an estimate of the net sentiment (positive - negative) in each chunk of the novel text for each sentiment lexicon. Let’s bind them together and visualize them in Figure 2.3.
```{r Comparing three dictionaries 3}
bind_rows(afinn,
          bing_and_nrc) %>% 
  ggplot(aes(index,sentiment,fill=method))+
  geom_col(show.legend = FALSE)+
  facet_wrap(~method,ncol=1,scales="free_y")
```

The three different lexicons for calculating sentiment give results that are different in an absolute sense but have similar relative trajectories through the novel. We see similar dips and peaks in sentiment at about the same places in the novel, but the absolute values are significantly different. The AFINN lexicon gives the largest absolute values, with high positive values. The lexicon from Bing et al. has lower absolute values and seems to label larger blocks of contiguous positive or negative text. The NRC results are shifted higher relative to the other two, labeling the text more positively, but detects similar relative changes in the text. We find similar differences between the methods when looking at other novels; the NRC sentiment is high, the AFINN sentiment has more variance, the Bing et al. sentiment appears to find longer stretches of similar text, but all three agree roughly on the overall trends in the sentiment through a narrative arc.

Why is, for example, the result for the NRC lexicon biased so high in sentiment compared to the Bing et al. result? Let’s look briefly at how many positive and negative words are in these lexicons.
```{r Comparing three dictionaries 4}
get_sentiments("nrc") %>% 
  filter(sentiment %in% c("positive",
                          "negative")) %>% 
  count(sentiment)

## OR
get_sentiments("nrc") %>% 
  filter(sentiment=="positive" | sentiment=="negative") %>% 
  count(sentiment)

3324/(3324+2312)
```

```{r}
get_sentiments("bing") %>% 
  count(sentiment)
4782/(4782+2006)
```

Both lexicons have more negative than positive words, but the ratio of negative to positive words is higher in the Bing lexicon than the NRC lexicon. This will contribute to the effect we see in the plot above, as will any systematic difference in word matches, e.g. if the negative words in the NRC lexicon do not match the words that Jane Austen uses very well. Whatever the source of these differences, we see similar relative trajectories across the narrative arc, with similar changes in slope, but marked differences in absolute sentiment from lexicon to lexicon. This is all important context to keep in mind when choosing a sentiment lexicon for analysis.

## 2.4 Most common positive and negative words
One advantage of having the data frame with both sentiment and word is that we can analyze word counts that contribute to each sentiment. By implementing `count()` here with arguments of both `word` and sentiment, we find out how much each word contributed to each sentiment.
```{r 2.4-1}
being_word_counts <- tidy_books %>% 
  inner_join(get_sentiments("bing")) %>% 
  count(word,sentiment,sort=T) %>% 
  ungroup()
being_word_counts
```

This can be shown visually, and we can pipe straight into ggplot2, if we like, because of the way we are consistently using tools built for handling tidy data frames.
```{r 2.4-2}
being_word_counts %>% 
  group_by(sentiment) %>% 
  top_n(10) %>% 
  ungroup() %>% 
  mutate(word=reorder(word,n)) %>% 
  ggplot(aes(word,n,fill=sentiment))+
  geom_col(show.legend = F)+
  facet_wrap(~sentiment,scales="free_y")+
  labs(x="Contribution to Sentiment",x=NULL)+
  coord_flip()
```

Figure 2.4 lets us spot an anomaly in the sentiment analysis; the word “miss” is coded as negative but it is used as a title for young, unmarried women in Jane Austen’s works. If it were appropriate for our purposes, we could easily add “miss” to a custom stop-words list using `bind_rows()`. We could implement that with a strategy such as this.

```{r 2.4-3}
custom_stop_words <- bind_rows(data_frame(word=c("miss"),
                                          lexicon=c("custom")),
                               stop_words)
custom_stop_words
```

## 2.5 Wordclouds
We’ve seen that this tidy text mining approach works well with ggplot2, but having our data in a tidy format is useful for other plots as well.

For example, consider the wordcloud package, which uses base R graphics. Let’s look at the most common words in Jane Austen’s works as a whole again, but this time as a wordcloud in Figure 2.5.
```{r 2.5-1}
# install.packages("wordcloud")
library(wordcloud)
tidy_books %>%
  anti_join(stop_words) %>%
  count(word) %>%
  with(wordcloud(word, n, max.words = 100))
```

In other functions, such as `comparison.cloud()`, you may need to turn the data frame into a matrix with reshape2’s `acast()`. Let’s do the sentiment analysis to tag positive and negative words using an inner join, then find the most common positive and negative words. Until the step where we need to send the data to `comparison.cloud()`, this can all be done with joins, piping, and dplyr because our data is in tidy format.
```{r 2.5-2}
library(reshape2)
tidy_books %>% 
  inner_join(get_sentiments("bing")) %>% 
  count(word,sentiment,sort=T) %>% 
  acast(word~sentiment,value.var = "n",fill=0) %>% 
  comparison.cloud(colors=c("gray20","gray80"),max.words = 100)
```

The size of a word’s text in Figure 2.6 is in proportion to its frequency within its sentiment. We can use this visualization to see the most important positive and negative words, but the sizes of the words are not comparable across sentiments.

## 2.6 Looking at units beyond just words

Lots of useful work can be done by tokenizing at the word level, but sometimes it is useful or necessary to look at different units of text. For example, some sentiment analysis algorithms look beyond only unigrams (i.e. single words) to try to understand the sentiment of a sentence as a whole. These algorithms try to understand that

> I am not having a good day.

is a sad sentence, not a happy one, because of negation. R packages included coreNLP (T. Arnold and Tilton 2016), cleanNLP (T. B. Arnold 2016), and sentimentr (Rinker 2017) are examples of such sentiment analysis algorithms. For these, we may want to tokenize text into sentences, and it makes sense to use a new name for the output column in such a case.
```{r 2.6-1}
PandP_sentences <- data_frame(text = prideprejudice) %>% 
  unnest_tokens(sentence, text, token = "sentences")
PandP_sentences
```

Let's look at just one. 
```{r 2.6-2}
PandP_sentences$sentence[2]
```

The sentence tokenizing does seem to have a bit of trouble with UTF-8 encoded text, especially with sections of dialogue; it does much better with punctuation in ASCII. One possibility, if this is important, is to try using `iconv()`, with something like `iconv(text, to = 'latin1')` in a mutate statement before unnesting.
```{r 2.6-3}
austen_chapters <- austen_books() %>% 
  group_by(book) %>% 
  unnest_tokens(chapter,text,token="regex",
                pattern="Chapter|CHAPTER[\\dIVXLC]") %>% 
  ungroup()

austen_chapters %>% 
  group_by(book) %>% 
  summarise(chapters=n())
```

We have recovered the correct number of chapters in each novel (plus an “extra” row for each novel title). In the `austen_chapters` data frame, each row corresponds to one chapter.

Near the beginning of this chapter, we used a similar regex to find where all the chapters were in Austen’s novels for a tidy data frame organized by one-word-per-row. We can use tidy text analysis to ask questions such as what are the most negative chapters in each of Jane Austen’s novels? First, let’s get the list of negative words from the Bing lexicon. Second, let’s make a data frame of how many words are in each chapter so we can normalize for the length of chapters. Then, let’s find the number of negative words in each chapter and divide by the total words in each chapter. For each book, which chapter has the highest proportion of negative words?
```{r 2.6-4}
bingnegative <- get_sentiments("bing") %>% 
  filter(sentiment=="negative")

wordcount <- tidy_books %>% 
  group_by(book,chapter) %>% 
  summarize(words=n())

tidy_books %>% 
  semi_join(bingnegative) %>% 
  group_by(book,chapter) %>% 
  summarize(negativewords=n()) %>% 
  left_join(wordcount,by=c("book","chapter")) %>% 
  mutate(ratio=negativewords/words) %>% 
  filter(chapter!=0) %>% 
  top_n(1) %>% 
  ungroup()
```

These are the chapters with the most sad words in each book, normalized for number of words in the chapter. What is happening in these chapters? In Chapter 43 of Sense and Sensibility Marianne is seriously ill, near death, and in Chapter 34 of Pride and Prejudice Mr. Darcy proposes for the first time (so badly!). Chapter 46 of Mansfield Park is almost the end, when everyone learns of Henry’s scandalous adultery, Chapter 15 of Emma is when horrifying Mr. Elton proposes, and in Chapter 21 of Northanger Abbey Catherine is deep in her Gothic faux fantasy of murder, etc. Chapter 4 of Persuasion is when the reader gets the full flashback of Anne refusing Captain Wentworth and how sad she was and what a terrible mistake she realized it to be.

## 2.7 Summary
Sentiment analysis provides a way to understand the attitudes and opinions expressed in texts. In this chapter, we explored how to approach sentiment analysis using tidy data principles; when text data is in a tidy data structure, sentiment analysis can be implemented as an inner join. We can use sentiment analysis to understand how a narrative arc changes throughout its course or what words with emotional and opinion content are important for a particular text. We will continue to develop our toolbox for applying sentiment analysis to different kinds of text in our case studies later in this book.

Reference
Arnold, Taylor, and Lauren Tilton. 2016. coreNLP: Wrappers Around Stanford CoreNLP Tools. https://cran.r-project.org/package=coreNLP.

Arnold, Taylor B. 2016. cleanNLP: A Tidy Data Model for Natural Language Processing. https://cran.r-project.org/package=cleanNLP.

Rinker, Tyler W. 2017. sentimentr: Calculate Text Polarity Sentiment. Buffalo, New York: University at Buffalo/SUNY. http://github.com/trinker/sentimentr.

# Chapter 3: Analyzing words and document frequency: tf-idf
A central question in text mining and natural language processing is how to quantify what a document is about. Can we do this by looking at the words that make up the document? One measure of how important a word may be is its term frequency (tf), how frequently a word occurs in a document, as we examined in Chapter 1. There are words in a document, however, that occur many times but may not be important; in English, these are probably words like “the”, “is”, “of”, and so forth. We might take the approach of adding words like these to a list of stop words and removing them before analysis, but it is possible that some of these words might be more important in some documents than others. A list of stop words is not a very sophisticated approach to adjusting term frequency for commonly used words.

It is a rule-of-thumb or heuristic quantity; while it has proved useful in text mining, search engines, etc., its theoretical foundations are considered less than firm by information theory experts. The inverse document frequency for any given term is defined as
$$
idf(term)=ln\frac{n_{documents}}{n_{documents containing term}}
$$

We can use tidy data principles, as described in Chapter 1, to approach tf-idf analysis and use consistent, effective tools to quantify how important various terms are in a document that is part of a collection.

## 3.1 Term frequency in Japne Austen's novels

Let’s start by looking at the published novels of Jane Austen and examine first term frequency, then tf-idf. We can start just by using dplyr verbs such as `group_by()` and `join()`. What are the most commonly used words in Jane Austen’s novels? (Let’s also calculate the total words in each novel here, for later use.)

```{r 3-1 Term frequency 1}
library(dplyr)
library(janeaustenr)
library(tidytext)

book_words <- austen_books() %>% 
  unnest_tokens(word,text) %>% 
  count(book,word,sort=T) %>% 
  ungroup()

total_words <- book_words %>% 
  group_by(book) %>% 
  summarize(total=sum(n))
book_words <- left_join(book_words,total_words)
book_words
```

There is one row in this `book_words` data frame for each word-book combination; `n` is the number of times that word is used in that book and `total` is the total words in that book. The usual suspects are here with the highest `n`, “the”, “and”, “to”, and so forth. In Figure 3.1, let’s look at the distribution of `n/total` for each novel, the number of times a word appears in a novel divided by the total number of terms (words) in that novel. This is exactly what term frequency is.
```{r 3-1-2}
library(ggplot2)
ggplot(book_words, aes(n/total, fill = book)) +
  geom_histogram(show.legend = FALSE) +
  xlim(NA, 0.0009) +
  facet_wrap(~book, ncol = 2, scales = "free_y")
```

There are very long tails to the right for these novels (those extremely common words!) that we have not shown in these plots. These plots exhibit similar distributions for all the novels, with many words that occur rarely and fewer words that occur frequently.

## 3.2 Zipf'z law
Distributions like those shown in Figure 3.1 are typical in language. In fact, those types of long-tailed distributions are so common in any given corpus of natural language (like a book, or a lot of text from a website, or spoken words) that the relationship between the frequency that a word is used and its rank has been the subject of study; a classic version of this relationship is called Zipf’s law, after George Zipf, a 20th century American linguist.

Since we have the data frame we used to plot term frequency, we can examine Zipf’s law for Jane Austen’s novels with just a few lines of dplyr functions.

```{r 3.2-1}
freq_by_rank <- book_words %>% 
  group_by(book) %>% 
  mutate(rank=row_number(),
         `term frequency`=n/total)
freq_by_rank
```

The `rank` column here tells us the rank of each word within the frequency table; the table was already ordered by n so we could use `row_number()` to find the rank. Then, we can calculate the term frequency in the same way we did before. Zipf’s law is often visualized by plotting rank on the x-axis and term frequency on the y-axis, on logarithmic scales. Plotting this way, an inversely proportional relationship will have a constant, negative slope.
```{r 3.2-2}
freq_by_rank %>% 
  ggplot(aes(rank,`term frequency`,color=book))+
  geom_line(size=1.1,alpha=0.8,show.legend = F)+
  scale_x_log10()+
  scale_y_log10()
```

Notice that Figure 3.2 is in log-log coordinates. We see that all six of Jane Austen’s novels are similar to each other, and that the relationship between rank and frequency does have negative slope. It is not quite constant, though; perhaps we could view this as a broken power law with, say, three sections. Let’s see what the exponent of the power law is for the middle section of the rank range.
```{r 3.2-3}
rank_subset <- freq_by_rank %>% 
  filter(rank<500,
         rank>10)
lm(log10(`term frequency`)~log10(rank),data=rank_subset)
```

Classic versions of Zipf’s law have

$$
frequency ∝ \frac{1}{rank}
$$
 
and we have in fact gotten a slope close to -1 here. Let’s plot this fitted power law with the data in Figure 3.3 to see how it looks.
```{r}
freq_by_rank %>% 
  ggplot(aes(rank, `term frequency`, color = book)) + 
  geom_abline(intercept = -0.62, slope = -1.1, color = "gray50", linetype = 2) +
  geom_line(size = 1.1, alpha = 0.8, show.legend = FALSE) + 
  scale_x_log10() +
  scale_y_log10()
```

We have found a result close to the classic version of Zipf’s law for the corpus of Jane Austen’s novels. The deviations we see here at high rank are not uncommon for many kinds of language; a corpus of language often contains fewer rare words than predicted by a single power law. The deviations at low rank are more unusual. Jane Austen uses a lower percentage of the most common words than many collections of language. This kind of analysis could be extended to compare authors, or to compare any other collections of text; it can be implemented simply using tidy data principles.

## 3.3 The `bind_tf_idf` function
The idea of tf-idf is to find the important words for the content of each document by decreasing the weight for commonly used words and increasing the weight for words that are not used very much in a collection or corpus of documents, in this case, the group of Jane Austen’s novels as a whole. Calculating tf-idf attempts to find the words that are important (i.e., common) in a text, but not too common. Let’s do that now.

The `bind_tf_idf` function in the tidytext package takes a tidy text dataset as input with one row per token (term), per document. One column (`word` here) contains the terms/tokens, one column contains the documents (`book` in this case), and the last necessary column contains the counts, how many times each document contains each term (n in this example). We calculated a `total` for each book for our explorations in previous sections, but it is not necessary for the `bind_tf_idf` function; the table only needs to contain all the words in each document.

```{r 3.3-1}
book_words <- book_words %>% 
  bind_tf_idf(word,book,n)
book_words
```

Notice that idf and thus tf-idf are zero for these extremely common words. These are all words that appear in all six of Jane Austen’s novels, so the idf term (which will then be the natural log of 1) is zero. The inverse document frequency (and thus tf-idf) is very low (near zero) for words that occur in many of the documents in a collection; this is how this approach decreases the weight for common words. The inverse document frequency will be a higher number for words that occur in fewer of the documents in the collection.

Let’s look at terms with high tf-idf in Jane Austen’s works.

```{r 3.3-2}
book_words %>% 
  select(-total) %>% 
  arrange(desc(tf_idf))
```

Here we see all proper nouns, names that are in fact important in these novels. None of them occur in all of novels, and they are important, characteristic words for each text within the corpus of Jane Austen’s novels.

Let’s look at a visualization for these high tf-idf words in Figure 3.4.
```{r 3.3-3}
library(magrittr)
library(tidytext)
library(dplyr)
library(ggplot2)

book_words %>% 
  arrange(desc(tf_idf)) %>% 
  mutate(word=factor(word,levels = rev(unique(word)))) %>% 
  group_by(book) %>% 
  top_n(15) %>% 
  ungroup %>% 
  ggplot(aes(word,tf_idf,fill=book))+
  geom_col(show.legend = F)+
  labs(x=NULL,y="tf-idf")+
  facet_wrap(~book,ncol=2,scales="free")+
  coord_flip()
```
  
Still all proper nouns in Figure 3.4! These words are, as measured by tf-idf, the most important to each novel and most readers would likely agree. What measuring tf-idf has done here is show us that Jane Austen used similar language across her six novels, and what distinguishes one novel from the rest within the collection of her works are the proper nouns, the names of people and places. This is the point of tf-idf; it identifies words that are important to one document within a collection of documents.  
  
## 3.4 A corpus of physics texts  
Let’s work with another corpus of documents, to see what terms are important in a different set of works. In fact, let’s leave the world of fiction and narrative entirely. Let’s download some classic physics texts from Project Gutenberg and see what terms are important in these works, as measured by tf-idf. Let’s download 

- Discourse on Floating Bodies by Galileo Galilei, 
http://www.gutenberg.org/ebooks/37729
- Treatise on Light by Christiaan Huygens,
http://www.gutenberg.org/ebooks/14725
- Experiments with Alternate Currents of High Potential and High - Frequency by Nikola Tesla, and 
http://www.gutenberg.org/ebooks/13476
- Relativity: The Special and General Theory by Albert Einstein.
http://www.gutenberg.org/ebooks/5001

This is a pretty diverse bunch. They may all be physics classics, but they were written across a 300-year timespan, and some of them were first written in other languages and then translated to English. Perfectly homogeneous these are not, but that doesn’t stop this from being an interesting exercise!
```{r 3.4-1}
# library(gutenbergr)
# physics <- gutenberg_download(c(37729, 14725, 13476, 5001), 
#                               meta_fields = "author")
```

Now that we have the texts, let’s use `unnest_tokens()` and `count()` to find out how many times each word was used in each text.
```{r 3.4-2}
# physics_words <- physics %>%
#   unnest_tokens(word, text) %>%
#   count(author, word, sort = TRUE) %>%
#   ungroup()
# 
# physics_words
```

Here we see just the raw counts; we need to remember that these documents are all different lengths. Let’s go ahead and calculate tf-idf, then visualize the high tf-idf words in Figure 3.5.
```{r 3.4-3}
# plot_physics <- physics_words %>%
#   bind_tf_idf(word, author, n) %>%
#   arrange(desc(tf_idf)) %>%
#   mutate(word = factor(word, levels = rev(unique(word)))) %>%
#   mutate(author = factor(author, levels = c("Galilei, Galileo",
#                                             "Huygens, Christiaan", 
#                                             "Tesla, Nikola",
#                                             "Einstein, Albert")))
# 
# plot_physics %>% 
#   group_by(author) %>% 
#   top_n(15, tf_idf) %>% 
#   ungroup() %>%
#   mutate(word = reorder(word, tf_idf)) %>%
#   ggplot(aes(word, tf_idf, fill = author)) +
#   geom_col(show.legend = FALSE) +
#   labs(x = NULL, y = "tf-idf") +
#   facet_wrap(~author, ncol = 2, scales = "free") +
#   coord_flip()
```

Very interesting indeed. One thing we see here is “eq” in the Einstein text?!
```{r 3.4-4}
# library(stringr)
# physics %>% 
#   filter(str_detect(text, "eq\\.")) %>% 
#   select(text)
```

Some cleaning up of the text may be in order. “K1” is the name of a coordinate system for Einstein:
```{r 3.4-5}
# physics %>% 
#   filter(str_detect(text,"K1")) %>% 
#   select(text)
```

Maybe it makes sense to keep this one. Also notice that in this line we have “co-ordinate”, which explains why there are separate “co” and “ordinate” items in the high tf-idf words for the Einstein text; the `unnest_tokens()` function separates around punctuation. Notice that the tf-idf scores for “co” and “ordinate” are close to same!

“AB”, “RC”, and so forth are names of rays, circles, angles, and so forth for Huygens.
```{r 3.4-6}
# physics %>% 
#   filter(str_detect(text, "AK")) %>% 
#   select(text)
```

Let’s remove some of these less meaningful words to make a better, more meaningful plot. Notice that we make a custom list of stop words and use `anti_join()` to remove them; this is a flexible approach that can be used in many situations. We will need to go back a few steps since we are removing words from the tidy data frame.
```{r 3.4-7}
# mystopwords <- data_frame(word = c("eq", "co", "rc", "ac", "ak", "bn", 
#                                    "fig", "file", "cg", "cb", "cm"))
# physics_words <- anti_join(physics_words, mystopwords, by = "word")
# plot_physics <- physics_words %>%
#   bind_tf_idf(word, author, n) %>%
#   arrange(desc(tf_idf)) %>%
#   mutate(word = factor(word, levels = rev(unique(word)))) %>%
#   group_by(author) %>% 
#   top_n(15, tf_idf) %>%
#   ungroup %>%
#   mutate(author = factor(author, levels = c("Galilei, Galileo",
#                                             "Huygens, Christiaan",
#                                             "Tesla, Nikola",
#                                             "Einstein, Albert")))
# 
# ggplot(plot_physics, aes(word, tf_idf, fill = author)) +
#   geom_col(show.legend = FALSE) +
#   labs(x = NULL, y = "tf-idf") +
#   facet_wrap(~author, ncol = 2, scales = "free") +
#   coord_flip()
```

One thing we can conclude from Figure 3.6 is that we don’t hear enough about ramparts or things being ethereal in physics today.

## 3.5 Summary
Using term frequency and inverse document frequency allows us to find words that are characteristic for one document within a collection of documents, whether that document is a novel or physics text or webpage. Exploring term frequency on its own can give us insight into how language is used in a collection of natural language, and dplyr verbs like `count()` and `rank()` give us tools to reason about term frequency. The tidytext package uses an implementation of tf-idf consistent with tidy data principles that enables us to see how different words are important in documents within a collection or corpus of documents.

# Chapter 4. Relatinships between words: n-grams and correlations
So far we’ve considered words as individual units, and considered their relationships to sentiments or to documents. However, many interesting text analyses are based on the relationships between words, whether examining which words tend to follow others immediately, or that tend to co-occur within the same documents.

In this chapter, we’ll explore some of the methods `tidytext` offers for calculating and visualizing relationships between words in your text dataset. This includes the `token` = `"ngrams"` argument, which tokenizes by pairs of adjacent words rather than by individual ones. We’ll also introduce two new packages: `ggraph`, which extends `ggplot2` to construct network plots, and `widyr`, which calculates pairwise correlations and distances within a tidy data frame. Together these expand our toolbox for exploring text within the tidy data framework.

## 4.1 Tokening by n-grams
We’ve been using the `unnest_tokens` function to tokenize by word, or sometimes by sentence, which is useful for the kinds of sentiment and frequency analyses we’ve been doing so far. But we can also use the function to tokenize into consecutive sequences of words, called n-grams. By seeing how often word X is followed by word Y, we can then build a model of the relationships between them.

We do this by adding the `token = "ngrams"` option to `unnest_tokens()`, and setting `n` to the number of words we wish to capture in each n-gram. When we set `n` to 2, we are examining pairs of two consecutive words, often called “bigrams”:
```{r 4.1-1}
library(dplyr)
library(tidytext)
library(janeaustenr)

austen_bigrams <- austen_books() %>% 
  unnest_tokens(bigram,text,token="ngrams",n=2)
austen_bigrams
```

This data structure is still a variation of the tidy text format. It is structured as one-token-per-row (with extra metadata, such as `book`, still preserved), but each token now represents a bigram.

### 4.1.1 Counting and filtering n-grams
Our usual tidy tools apply equally well to n-gram analysis. We can examine the most common bigrams using dplyr’s `count()`:
```{r 4.1.1-1}
austen_bigrams %>% 
  count(bigram,sort=T)
```

As one might expect, a lot of the most common bigrams are pairs of common (uninteresting) words, such as `of the` and `to be`: what we call “stop-words” (see Chapter 1). This is a useful time to use tidyr’s `separate()`, which splits a column into multiple based on a delimiter. This lets us separate it into two columns, “word1” and “word2”, at which point we can remove cases where either is a stop-word.
```{r 4.1.1-2}
library(tidyverse)

bigrams_separated <- austen_bigrams %>% 
  separate(bigram,c("word1","word2"),sep=" ")

bigrams_filtered <- bigrams_separated %>% 
  filter(!word1 %in% stop_words$word) %>% 
  filter(!word2 %in% stop_words$word)

# new bigram counts:
bigram_counts <- bigrams_filtered %>% 
  count(word1, word2,sort=T)
bigram_counts
```

We can see that names (whether first and last or with a salutation) are the most common pairs in Jane Austen books.

In other analyses, we may want to work with the recombined words. tidyr’s `unite()` function is the inverse of `separate()`, and lets us recombine the columns into one. Thus, “`separate/filter/count/unite`” let us find the most common bigrams not containing stop-words.
```{r 4.1.1-3}
bigrams_filtered
bigrams_united <- bigrams_filtered %>% 
  unite(bigram,word1, word2, sep=" ")
bigrams_united
```

In other analyses you may be interested in the most common trigrams, which are consecutive sequences of 3 words. We can find this by setting `n = 3`:
```{r 4.1.1-4}
austen_books() %>% 
  unnest_tokens(trigram,text,token="ngrams",n=3) %>% 
  separate(trigram,c("word1","word2","word3"),sep=" ") %>% 
  filter(!word1 %in% stop_words$word,
         !word2 %in% stop_words$word,
         !word3 %in% stop_words$word) %>% 
  count(word1,word2,word3,sort=T)
```

### 4.1.2 Analyzing bigrams
This one-bigram-per-row format is helpful for exploratory analyses of the text. As a simple example, we might be interested in the most common “streets” mentioned in each book:
```{r 4.1.2-1}
bigrams_filtered %>% 
  filter(word2=="street") %>% 
  count(book,word1,sort=T)
```

A bigram can also be treated as a term in a document in the same way that we treated individual words. For example, we can look at the tf-idf (Chapter 3) of bigrams across Austen novels. These tf-idf values can be visualized within each book, just as we did for words (Figure 4.1).
```{r 4.1.2-2}
bigram_tf_idf <- bigrams_united %>% 
  count(book,bigram) %>% 
  bind_tf_idf(bigram,book,n) %>% 
  arrange(desc(tf_idf))
bigram_tf_idf
```

Much as we discovered in Chapter 3, the units that distinguish each Austen book are almost exclusively names. We also notice some pairings of a common verb and a name, such as “replied elizabeth” in Pride & Prejudice, or “cried emma” in Emma.

There are advantages and disadvantages to examining the tf-idf of bigrams rather than individual words. Pairs of consecutive words might capture structure that isn’t present when one is just counting single words, and may provide context that makes tokens more understandable (for example, “pulteney street”, in Northanger Abbey, is more informative than “pulteney”). However, the per-bigram counts are also sparser: a typical two-word pair is rarer than either of its component words. Thus, bigrams can be especially useful when you have a very large text dataset.

### 4.1.3 Using bigrams to provide context in sentiment analysis
Our sentiment analysis approach in Chapter 2 simply counted the appearance of positive or negative words, according to a reference lexicon. One of the problems with this approach is that a word’s context can matter nearly as much as its presence. For example, the words “happy” and “like” will be counted as positive, even in a sentence like “I’m not happy and I don’t like it!”
```{r 4.1.3-1}
bigrams_separated %>% 
  filter(word1=="not") %>% 
  count(word1,word2,sort=T)
```

By performing sentiment analysis on the bigram data, we can examine how often sentiment-associated words are preceded by “not” or other negating words. We could use this to ignore or even reverse their contribution to the sentiment score.

Let’s use the AFINN lexicon for sentiment analysis, which you may recall gives a numeric sentiment score for each word, with positive or negative numbers indicating the direction of the sentiment.
```{r 4.1.3-2}
AFINN <- get_sentiments("afinn")
AFINN
```

We can then examine the most frequent words that were preceded by “not” and were associated with a sentiment.
```{r 4.1.3-3}
not_words <- bigrams_separated %>% 
  filter(word1=="not") %>% 
  inner_join(AFINN,by=c(word2="word")) %>% 
  count(word2,score,sort=T) %>% 
  ungroup()
not_words
```

For example, the most common sentiment-associated word to follow “not” was “like”, which would normally have a (positive) score of 2.

It’s worth asking which words contributed the most in the “wrong” direction. To compute that, we can multiply their score by the number of times they appear (so that a word with a score of +3 occurring 10 times has as much impact as a word with a sentiment score of +1 occurring 30 times). We visualize the result with a bar plot (Figure 4.2).
```{r 4.1.3-4}
not_words %>% 
  mutate(contribution=n*score) %>% 
  arrange(desc(abs(contribution))) %>% 
  head(20) %>% 
  mutate(word2=reorder(word2, contribution)) %>% 
  ggplot(aes(word2,n*score,fill=n*score>0))+
  geom_col(show.legend = F)+
  xlab("Words preceded by \"not\"")+
  ylab("Sentiment score * number of occurences")+
  coord_flip()
```

Figure 4.2: The 20 words preceded by ‘not’ that had the greatest contribution to sentiment scores, in either a positive or negative direction

The bigrams “not like” and “not help” were overwhelmingly the largest causes of misidentification, making the text seem much more positive than it is. But we can see phrases like “not afraid” and “not fail” sometimes suggest text is more negative than it is.

“Not” isn’t the only term that provides some context for the following word. We could pick four common words (or more) that negate the subsequent term, and use the same joining and counting approach to examine all of them at once.
```{r 4.1.3-5}
negation_words <- c("not","no","never","without")

negated_words <- bigrams_separated %>% 
  filter(word1 %in% negation_words) %>% 
  inner_join(AFINN,by=c(word2="word")) %>% 
  count(word1,word2,score,sort=T) %>% 
  ungroup()

# data vis
negated_words
negated_words %>% 
  head(20) %>% 
ggplot(aes(word2,n*score,fill=n*score>0))+
  geom_col(show.legend = F)+
  xlab("Words preceded by negation term")+
  ylab("Sentiment score * number of occurences")+
  coord_flip()+
  facet_wrap(~word1)
```

### 4.1.4 Visualizing a network of bigrams with ggraph

We may be interested in visualizing all of the relationships among words simultaneously, rather than just the top few at a time. As one common visualization, we can arrange the words into a network, or “graph.” Here we’ll be referring to a “graph” not in the sense of a visualization, but as a combination of connected nodes. A graph can be constructed from a tidy object since it has three variables:

- from: the node an edge is coming from
- to: the node an edge is going towards
- weight: A numeric value associated with each edge

The igraph package has many powerful functions for manipulating and analyzing networks. One way to create an igraph object from tidy data is the `graph_from_data_frame()` function, which takes a data frame of edges with columns for “from”, “to”, and edge attributes (in this case `n`):
```{r 4.1.4-1}
# install.packages("igraph")
library(igraph)
bigram_counts
```

```{r 4.1.4-2}
# filter for only relatively common combinations
bigram_graph <- bigram_counts %>%
  filter(n > 20) %>%
  graph_from_data_frame()

bigram_graph
```

igraph has plotting functions built in, but they’re not what the package is designed to do, so many other packages have developed visualization methods for graph objects. We recommend the ggraph package (Pedersen 2017), because it implements these visualizations in terms of the grammar of graphics, which we are already familiar with from ggplot2.

We can convert an igraph object into a `ggraph` with the ggraph function, after which we add layers to it, much like layers are added in ggplot2. For example, for a basic graph we need to add three layers: nodes, edges, and text.
```{r 4.1.4-3}
# install.packages("ggraph")
library(ggraph)
set.seed(2017)

ggraph(bigram_graph,layout="fr")+
  geom_edge_link()+
  geom_node_point()+
  geom_node_text(aes(label=name),vjust=1,hjust=1)
```

In Figure 4.4, we can visualize some details of the text structure. For example, we see that salutations such as “miss”, “lady”, “sir”, “and”colonel" form common centers of nodes, which are often followed by names. We also see pairs or triplets along the outside that form common short phrases (“half hour”, “thousand pounds”, or “short time/pause”).

We conclude with a few polishing operations to make a better looking graph (Figure 4.5):

- We add the `edge_alpha` aesthetic to the link layer to make links transparent based on how common or rare the bigram is
- We add directionality with an arrow, constructed using `grid::arrow()`, including an `end_cap` option that tells the arrow to end before touching the node
- We tinker with the options to the node layer to make the nodes more attractive (larger, blue points)
- We add a theme that’s useful for plotting networks, `theme_void()`
```{r}
set.seed(2016)

a <- grid::arrow(type="closed",length=unit(.15,"inches"))

ggraph(bigram_graph,layout="fr")+
  geom_edge_link(aes(edge_alpha=n),show.legend = F,
                 arrow=a,end_cap=circle(.07,'inches'))+
  geom_node_point(color="lightblue",size=5)+
  geom_node_text(aes(label=name),vjust=1,hjust=1)+
  theme_void()
```

It may take some experimentation with ggraph to get your networks into a presentable format like this, but the network structure is useful and flexible way to visualize relational tidy data.

Note that this is a visualization of a Markov chain, a common model in text processing. In a Markov chain, each choice of word depends only on the previous word. In this case, a random generator following this model might spit out “dear”, then “sir”, then “william/walter/thomas/thomas’s”, by following each word to the most common words that follow it. To make the visualization interpretable, we chose to show only the most common word to word connections, but one could imagine an enormous graph representing all connections that occur in the text.

### 4.1.5 Visualizing bigrams in other texts
We went to a good amount of work in cleaning and visualizing bigrams on a text dataset, so let’s collect it into a function so that we easily perform it on other text datasets.

```{r 4.1.5-1}

# package
library(dplyr)
library(tidyr)
library(tidytext)
library(ggplot2)
library(igraph)
library(ggraph)

count_bigrams <- function(dateset){
  dataset %>% 
    unnest_tokens(bigram,text,token="ngrams",n=2) %>% 
    separate(bigram,c("word1","word2"),sep=" ") %>% 
    filter(!word1 %in% stop_words$word,
           !word2 %in% stop_words$word) %>% 
    count(word1,word2,sort=T)
}

visualize_bigrams <- function(bigrams){
  set.seed(2016)
  a <- grid::arrow(type="closed",length=unit(.15,"inches"))
  
  bigrams %>% 
    graph_from_data_frame() %>% 
    ggraph(layout="fr")+
    geom_edge_link(aes(edge_alpha=n),show.legend = F,arrow=a)+
    geom_node_point(color="lightblue",size=5)+
    geom_node_text(aes(label=name),vjust=1,hjust=1)+
    theme_void()
}
```

At this point, we could visualize bigrams in other works, such as the King James Version of the Bible:
```{r}
# the King James version is book 10 on project Gutenberg
# library(gutenbergr)
# kjv <- gutenberg_download(10)
```

```{r}
# library(stringr)
# 
# kjv_bigrams <- kjv %>% 
#   count_bigrams()
# 
# # filter out rare combinations, as well as digits
# kjv_bigrams %>% 
#   filter(n>40,
#          !str_detect(word1, "\\d"),
#          !str_detect(word2,"\\d")) %>% 
#   visualize_bigrams()
```

Figure 4.6 thus lays out a common “blueprint” of language within the Bible, particularly focused around “thy” and “thou” (which could probably be considered stopwords!) You can use the gutenbergr package and these `count_bigrams/visualize_bigrams` functions to visualize bigrams in other classic books you’re interested in.

## 4.2 Counting and correlating pairs of words with the widyr package
Tokenizing by n-gram is a useful way to explore pairs of adjacent words. However, we may also be interested in words that tend to co-occur within particular documents or particular chapters, even if they don’t occur next to each other.

Tidy data is a useful structure for comparing between variables or grouping by rows, but it can be challenging to compare between rows: for example, to count the number of times that two words appear within the same document, or to see how correlated they are. Most operations for finding pairwise counts or correlations need to turn the data into a wide matrix first.

We’ll examine some of the ways tidy text can be turned into a wide matrix in Chapter 5, but in this case it isn’t necessary. The widyr package makes operations such as computing counts and correlations easy, by simplifying the pattern of “widen data, perform an operation, then re-tidy data” (Figure 4.7). We’ll focus on a set of functions that make pairwise comparisons between groups of observations (for example, between documents, or sections of text).

### 4.2.1 Counting and correlating among sections 
Consider the book “Pride and Prejudice” divided into 10-line sections, as we did (with larger sections) for sentiment analysis in Chapter 2. We may be interested in what words tend to appear within the same section.

```{r 4.2.1-1}
austen_section_words <- austen_books() %>% 
  filter(book=="Pride & Prejudice") %>% 
  mutate(section=row_number() %/% 10) %>% 
  filter(section>0) %>% 
  unnest_tokens(word,text) %>% 
  filter(!word %in% stop_words$word)

austen_section_words
```

One useful function from widyr is the `pairwise_count()` function. The prefix `pairwise_` means it will result in one row for each pair of words in the `word` variable. This lets us count common pairs of words co-appearing within the same section:
```{r 4.2.1-2}
# install.packages("widyr")
library(widyr)

# count words co-occuring within sections
word_pairs <- austen_section_words %>% 
  pairwise_count(word,section,sort=T)
word_pairs
```

Notice that while the input had one row for each pair of a document (a 10-line section) and a word, the output has one row for each pair of words. This is also a tidy format, but of a very different structure that we can use to answer new questions.

For example, we can see that the most common pair of words in a section is “Elizabeth” and “Darcy” (the two main characters). We can easily find the words that most often occur with Darcy:
```{r 4.2.1-3}
word_pairs %>% 
  filter(item1=="darcy")
```

### 4.2.2 Pairwise correlation
Pairs like “Elizabeth” and “Darcy” are the most common co-occurring words, but that’s not particularly meaningful since they’re also the most common individual words. We may instead want to examine correlation among words, which indicates how often they appear together relative to how often they appear separately.

In particular, here we’ll focus on the phi coefficient, a common measure for binary correlation. The focus of the phi coefficient is how much more likely it is that either both word X and Y appear, or neither do, than that one appears without the other.

Consider the following table:

\begin{table}{htb}
  \begin{tabular}{lcrr}
   Has word X & n11 & n10 & n1
   
  
  \end{tabular}
\end{table}

For example $n_{11}$ represents the number of documents where both word X and word Y appear, $n_{00}$ the number where neither appears, and $n_{10}$ and $n_{01}$ the cases one apprear without the other. In terms of this table, the phi coefficient is:
$$
\phi =\frac{n_{11}n_{00}-n_{10}n_{01}}{\sqrt{n_1n_2n_0n_1}}
$$

* The phi coefficient is equivalent to the Pearson correlation, which you may have heard of elsewhere, when it is applied to binary data).

The `pairwise_cor()` function in widyr lets us find the phi coefficient between words based on how often they appear in the same section. Its syntax is similar to `pairwise_count()`.
```{r 4.2.2-1}
# we need to filter for at least relatively common words first
library(widyr)
word_cors <- austen_section_words %>% 
  group_by(word) %>% 
  filter(n()>=20) %>% 
  pairwise_cor(word,section,sort=T)
word_cors
```

This output format is helpful for exploration. For example, we could find the words most correlated with a word like “pounds” using a `filter` operation.
```{r 4.2.2-2}
head(word_cors)
word_cors %>% 
  filter(item1=="pounds")
```

This lets us pick particular interesting words and find the other words most associated with them (Figure 4.8).
```{r 4.2.2-3}
word_cors %>% 
  filter(item1 %in% c("elizabeth","pounds","married","pride")) %>% 
  group_by(item1) %>% 
  top_n(6) %>% 
  ungroup() %>% 
  mutate(item2=reorder(item2,correlation)) %>% 
  ggplot(aes(item2,correlation))+
  geom_bar(stat="identity") +
  facet_wrap(~item1,scale="free")+
  coord_flip()

```

Just as we used ggraph to visualize bigrams, we can use it to visualize the correlations and clusters of words that were found by the widyr package (Figure 4.9).
```{r 4.4.2-4}
set.seed(2016)

word_cors %>% 
  filter(correlation>.15) %>% 
  graph_from_data_frame() %>% 
  ggraph(layout="fr")+
  geom_edge_link(aes(edge_alpha=correlation),show.legend = F)+
  geom_node_point(color="lightblue",size=5)+
  geom_node_text(aes(label=name),repel=T)+
  theme_void()
```

Note that unlike the bigram analysis, the relationships here are symmetrical, rather than directional (there are no arrows). We can also see that while pairings of names and titles that dominated bigram pairings are common, such as “colonel/fitzwilliam”, we can also see pairings of words that appear close to each other, such as “walk” and “park”, or “dance” and “ball”.

## 4.3 Summary
This chapter showed how the tidy text approach is useful not only for analyzing individual words, but also for exploring the relationships and connections between words. Such relationships can involve n-grams, which enable us to see what words tend to appear after others, or co-occurences and correlations, for words that appear in proximity to each other. This chapter also demonstrated the ggraph package for visualizing both of these types of relationships as networks. These network visualizations are a flexible tool for exploring relationships, and will play an important role in the case studies in later chapters.

# Chapter 5: Converting to and from non-tidy formats
In the previous chapters, we’ve been analyzing text arranged in the tidy text format: a table with one-token-per-document-per-row, such as is constructed by the `unnest_tokens()` function. This lets us use the popular suite of tidy tools such as dplyr, tidyr, and ggplot2 to explore and visualize text data. We’ve demonstrated that many informative text analyses can be performed using these tools.

However, most of the existing R tools for natural language processing, besides the tidytext package, aren’t compatible with this format. The CRAN Task View for Natural Language Processing lists a large selection of packages that take other structures of input and provide non-tidy outputs. These packages are very useful in text mining applications, and many existing text datasets are structured according to these formats.

Computer scientist Hal Abelson has observed that “No matter how complex and polished the individual operations are, it is often the quality of the glue that most directly determines the power of the system” (Abelson 2008). In that spirit, this chapter will discuss the “glue” that connects the tidy text format with other important packages and data structures, allowing you to rely on both existing text mining 

Figure 5.1: A flowchart of a typical text analysis that combines tidytext with other tools and data formats, particularly the tm or quanteda packages. This chapter shows how to convert back and forth between document-term matrices and tidy data frames, as well as converting from a Corpus object to a text data frame.

Figure 5.1 illustrates how an analysis might switch between tidy and non-tidy data structures and tools. This chapter will focus on the process of tidying document-term matrices, as well as casting a tidy data frame into a sparse matrix. We’ll also explore how to tidy Corpus objects, which combine raw text with document metadata, into text data frames, leading to a case study of ingesting and analyzing financial articles.

## 5.1 Tidying a document term matrix

One of the most common structures that text mining packages work with is the document-term matrix (or DTM). This is a matrix where:

- each row represents one document (such as a book or article),
- each column represents one term, and
- each value (typically) contains the number of appearances of that term in that document.

Since most pairings of document and term do not occur (they have the value zero), DTMs are usually implemented as sparse matrices. These objects can be treated as though they were matrices (for example, accessing particular rows and columns), but are stored in a more efficient format. We’ll discuss several implementations of these matrices in this chapter.

DTM objects cannot be used directly with tidy tools, just as tidy data frames cannot be used as input for most text mining packages. Thus, the tidytext package provides two verbs that convert between the two formats.

- `tidy()` turns a document-term matrix into a tidy data frame. This verb comes from the broom package (Robinson 2017), which provides similar tidying functions for many statistical models and objects.
- `cast()` turns a tidy one-term-per-row data frame into a matrix. tidytext provides three variations of this verb, each converting to a different type of matrix: `cast_sparse()` (converting to a sparse matrix from the Matrix package), `cast_dtm()` (converting to a `DocumentTermMatrix` object from tm), and `cast_dfm()` (converting to a `dfm` object from quanteda).

As shown in Figure 5.1, a DTM is typically comparable to a tidy data frame after a `count` or a `group_by`/`summarize` that contains counts or another statistic for each combination of a term and document.

### 5.1.1 Tidying DocumentTermMatrix objects

Perhaps the most widely used implementation of DTMs in R is the `DocumentTermMatrix` class in the tm package. Many available text mining datasets are provided in this format. For example, consider the collection of Associated Press newspaper articles included in the topicmodels package.
```{r 5.1.1-1}
# install.packages("tm")
# install.packages("topicmodels")

library(tm)
library(topicmodels)

data("AssociatedPress", package = "topicmodels")
AssociatedPress
```

We see that this dataset contains documents (each of them an AP article) and terms (distinct words). Notice that this DTM is 99% sparse (99% of document-word pairs are zero). We could access the terms in the document with the `Terms()` function.

```{r 5.1.1-2}
terms <- Terms(AssociatedPress)
head(terms)
```

If we wanted to analyze this data with tidy tools, we would first need to turn it into a datha frame with one-token-per-document-per-row. The broom package introduced the `tidy()` verb, which takes a non-tidy object and turns it into a tidy data frame. The tidytext package implements this method for `DocumentTermMatrix` objects.
```{r 5.1.1-2}
library(dplyr)
library(tidytext)

ap_td <- tidy(AssociatedPress)
ap_td
```

Notice that we now have a tidy three-column tbl_df, with variables `document`, `term`, and `count`. This tidying operation is similar to the `melt()` function from the reshape2 package (Wickham 2007) for non-sparse matrices.

As we’ve seen in previous chapters, this form is convenient for analysis with the dplyr, tidytext and ggplot2 packages. For example, you can perform sentiment analysis on these newspaper articles with the approach described in Chapter 2.
```{r 5.1.1-3}
ap_sentiments <- ap_td %>% 
  inner_join(get_sentiments("bing"),by=c(term="word"))
ap_sentiments

ap_sentiments %>% 
  count(sentiment,term,wt=count) %>% 
  filter(n>=200) %>% 
  arrange(desc(n))
```

This would let us visualize which words from the AP articles most often contributed to positive or negative sentiment, seen in Figure 5.2. We can see that the most common positive words include “like”, “work”, “support”, and “good”, while the most negative words include “killed”, “death”, and “vice”. (The inclusion of “vice” as a negative term is probably a mistake on the algorithm’s part, since it likely usually refers to “vice president”).
```{r 5.1.1-4}
library(ggplot2)

ap_sentiments %>% 
  count(sentiment,term,wt=count) %>% 
  ungroup() %>% 
  filter(n>=200) %>% 
  mutate(n=ifelse(sentiment=="negative",-n,n)) %>% 
  mutate(term=reorder(term,n)) %>% 
  ggplot(aes(term,n,fill=sentiment))+
  geom_bar(stat="identity")+
  ylab("Contribution tosentiment")+
  coord_flip()
```

### 5.1.2 Tidying dfm objects
Other text mining packages provide alternative implementations of document-term matrices, such as the `dfm` (document-feature matrix) class from the quanteda package (Benoit and Nulty 2016). For example, the quanteda package comes with a corpus of presidential inauguration speeches, which can be converted to a `dfm` using the appropriate function.
```{r 5.1.2-1}
install.packages("quanteda")
library(quanteda)
data("data_corpus_inaugural",package="quanteda")
inaug_dfm <- quanteda::dfm(data_corpus_inaugural,verbose=F)
inaug_dfm
```

The `tidy` method works on these document-feature matrices as well, turning into a one-token-per-document-per-row table:
```{r 15.1.2-2}
inaug_td <- tidy(inaug_dfm)
inaug_td
```

We may be interested in finding the words most specific to each of the inaugural speeches. This could be quantified by calculating the tf-idf of each term-speech pair using the `bind_tf_idf()` function, as described in Chapter 3.
```{r 15.1.2-3}
inaug_tf_idf <- inaug_td %>% 
  bind_tf_idf(term,document,count) %>% 
  arrange(desc(tf_idf))
inaug_tf_idf
```

We could use this data to pick four notable inaugural addresses (from Presidents Lincoln, Roosevelt, Kennedy, and Obama), and visualize the words most specific to each speech, as shown in Figure 5.3.
```{r 15.1.2-4}
inaug_tf_idf %>% 
ggplot(aes(term,tf_idf,fill=document))+
  geom_bar(stat="identity")+
  ylab("tf-idf")+coord_flip()+facet_wrap(~document)
```






