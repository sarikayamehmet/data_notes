---
PageTitle: Categorical data in the tidyverse
---
# Introduction to factor variables
Mutate() and summarise() in dplyr both have variants where you can add the suffix if or all to change the operation. mutate_if() and summarise_if apply their second argument, a function, to all columns where the first argument is true, and mutate_all() and summarise_all take one argument, a function, and apply it to all columns. There are more handy functions - summarise_at() and mutate_at(). These let you select columns by name-based select helpers. 

dplyr has two other functions that can come in handy when exploring a dataset. The first is `top_n(x, var)`, which gets us the first x rows of a dataset based on the value of var. The other is `pull()`, which allows us to extract a column and take out the name, leaving only the value(s) from the column. 

`fct_reorder` will reorder the levels of a factor. `fct_rev` organizes a factor from the most to the least relevant.

The following code will organize by each level and from biggest to smallest
```r
# Make a bar plot
ggplot(multiple_choice_responses, aes(fct_rev(fct_infreq(EmployerIndustry)))) + 
    geom_bar() + 
    # flip the coordinates
    coord_flip()
```
A ggplot with arranged factors

```r
multiple_choice_responses %>%
  # remove NAs
  filter(!is.na(EmployerIndustry) & !is.na(Age)) %>%
  # get mean_age by EmployerIndustry
  group_by(EmployerIndustry) %>%
  summarise(mean_age = mean(Age)) %>%
  # reorder EmployerIndustry by mean_age 
  mutate(EmployerIndustry = fct_reorder(EmployerIndustry, mean_age)) %>%
  # make a scatterplot of EmployerIndustry by mean_age
  ggplot(aes(x = EmployerIndustry, y = mean_age)) + 
    geom_point() + 
    coord_flip()
```
The function `fct_reorder()` takes 2 arguments: the factor we want to reorder and the variable we want to reorder by. In ggplot we introduce this function inside asthetics as seen above.

When ploting histograms we care about the frequency of each factor. To achive this we use the function `fct_infreq()` that returns the frequency of each factor. `fct_rev()` reverts the arrange by frequency done by `fct_infreq()`. 

```r
# function mutates from character to factor

column %>% mutate_if(is.character, as.factor)

```
The `mutate_if()` function needs 2 inputs. The first is a function that returns a logical result (TRUE/FALSE),  the second is a function to apply in case the first function returns true. **Is importante to notice that mutate_if applies column wise and not row wise. Therefore the first function will return results per columns and not observations. (for this we can use mutate_at)**.

The function `nlevels()` returns the number of levels in a factor while `levels()` return the labels of each level. In the case that we want to know the levels for all factors in a database we can use the following code.

```r
db %>% summarise_if(is.factor, nlevels)
```
# Manipulating Factor Variables
Untill now we have seen factors with no order and therefore it made sense to analyse either by comparing to other variable or by frequency. 

Nonetheless in some cases there are some order associated to each level. 

In order to plot a factor in the correct order we can use the function `fct_relevel()` inside ggplot aes.

```r
ggplot(aes(nlp_frequency,
  x = fct_relevel(response, "Rarely", "Sometimes", "Often", "Most of the time))) +
  geom_bar() 
```
We don't have to manually adjust all the levels. The default behaviour is that the input represents levels that will be put first. Therefore if we only use one than that will be the first one and all the others will remain the same. In the case we don't want to move this levels to the front we can use the the argument `after = <int>` where <int> represents the position that the first level inputed will occupy. If after is = Inf than it will move the levels to the end.

`fct_recode()` allows us to rename a factor level. The new label stays on the right and the old label on the left.

When we have to many levels we can use `fct_collapse()` to join different levels. code example:

```r
flying_etiquette %>%
  mutate(height = fct_collapse(height, 
             under_5_3 = c("Under 5 ft.", "5'0\"", "5'1\"", "5'2\""),
             over_6_1 = c("6'1\"", "6'2\"", "6'3\"", "6'4\"", 
             "6'5\"", "6'6\" and above"))) %>%
  pull(height) %>%
  levels()
```
Another solution is to use `fct_other()` 

```r
flying_etiquette %>%
  mutate(new_height = fct_other(height, keep = c("6'4\"", "5'1\""))) %>%
  count(new_height)
```
This will only keep to levels and put the rest in other. if we use drop it will keepp all the others except the ones we choose. 

If we want to group the least common we can use `fct_lump()`. In this case we specify the propotion we want to keep.  
Code example:

```r
multiple_choice_responses %>%
    # Create new variable, grouped_titles, by collapsing levels in CurrentJobTitleSelect
    mutate(grouped_titles = fct_collapse(CurrentJobTitleSelect, 
        "Computer Scientist" = c("Programmer", "Software Developer/Software Engineer"), 
        "Researcher" = "Scientist/Researcher", 
        "Data Analyst/Scientist/Engineer" = c("DBA/Database Engineer", "Data Scientist", 
                                              "Business Analyst", "Data Analyst", 
                                              "Data Miner", "Predictive Modeler"))) %>%
    # Turn every title that isn't now one of the grouped_titles into "Other"
    mutate(grouped_titles = fct_other(grouped_titles, 
                             keep = c("Computer Scientist", 
                                     "Researcher", 
                                     "Data Analyst/Scientist/Engineer"))) %>% 
    # Get a count of the grouped titles
    count(grouped_titles)
```
This code collapses some levels and them keep only some by adding all the others to a category other. The last line counts the frequency.

```r
multiple_choice_responses %>%
  # remove NAs of MLMethodNextYearSelect
  filter(!is.na(MLMethodNextYearSelect)) %>%
  # create ml_method, which lumps all those with less than 5% of people into "Other"
  mutate(ml_method = fct_lump(MLMethodNextYearSelect, prop = .05)) %>%
  # count the frequency of your new variable, sorted in descending order
  count(ml_method, sort = TRUE)
```
The next example keeps the 5 most frequent category and keeps all the others as a lump category:

```r
multiple_choice_responses %>%
  # remove NAs 
  filter(!is.na(MLMethodNextYearSelect)) %>%
  # create ml_method, retaining the 5 most common methods and renaming others "other method" 
  mutate(ml_method = fct_lump(MLMethodNextYearSelect, n = 5, other_level = "other method")) %>%
  # count the frequency of your new variable, sorted in descending order
  count(ml_method, sort = TRUE)
```
# Creating factor variables

```r
work_challenges <- multipleChoiceResponses %>%
  select(contains("WorkChallengeFrequency")) %>%
  gather(work_challenge, frequency) %>%
  mutate(work_challenge = str_remove(work_challenge, 
  "WorkChallengeFrequency"))
```
`str_remove()` will remove a part of string.

add_count adds a column to the dataset, n, keeping the same number of rows as the original dataset. Just like count, n defaults to be the number of rows for each group, but you can change that with the wt (weight) argument. You set wt equal to another column to make n now equal to the sum of that column for each group.

```R
perc_useful_platform <- learning_platform_usefulness %>%
  # change dataset to one row per learning_platform usefulness pair with number of entries for each
  count(learning_platform, usefulness) %>%
  # use add_count to create column with total number of answers for that learning_platform
  add_count(learning_platform, wt = n) %>%
  # create a new column, perc, that is the percentage of people giving that response for that learning_platform
  mutate(perc = n / nn)

# create a line graph for each question with usefulness on x-axis and percentage of responses on y
ggplot(perc_useful_platform, aes(x = usefulness, y = perc, group = learning_platform)) + 
  geom_line() + 
  facet_wrap(~ learning_platform)
```

`labs()` changes the labels of axis on ggplot. 

The following example uses the theme function from ggplot to change the labels from horizontal to vertical.

```r
ggplot(job_titles_by_perc,
        aes(x = CurrentJobTitleSelect, y = perc_w_title)) + 
    geom_point() + 
    theme(axis.text.x = element_text(angle = 90, hjust = 1))
```

In order to change an axis to percentage we can use the function `scale_y_continuous()` after labs.

```r
scale_y_continuous(labels = scales::percent_format())
```

Example of using `case_when` to create factor variables

```r
multiple_choice_responses %>%
    # Filter for rows where Age is between 10 and 90
    filter(between(Age,10,90)) %>%
    # Create the generation variable based on age
    mutate(generation = case_when(
    between(Age, 10, 22) ~ "Gen Z", 
    between(Age, 23, 37) ~ "Gen Y", 
    between(Age, 38, 52) ~ "Gen X", 
    between(Age, 53, 71) ~ "Baby Boomer", 
    between(Age, 72, 90) ~ "Silent"
    )) %>%
    # Get a count of how many answers in each generation
    count(generation)
```
# Case Study on Flight Etiquette

REGEX expressions:
`.` - matches any character
`*` - zero or more times

example:

```r
gathered_data %>%
    # Remove everything before and including "rude to "
    mutate(response_var = str_remove(response_var, ".*rude to ")) %>%
    # Remove "on a plane"
    mutate(response_var = str_remove(response_var, "on a plane"))
```

```r
initial_plot + 
  geom_text(aes(label = round(mean_mpg)))
```
adds value to every column in this case the mean.