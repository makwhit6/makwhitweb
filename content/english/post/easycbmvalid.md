---
author: Makayla Whitney
date: "2022-05-03"
description: model development process
tags:
- markdown
- text
series:
- series-setup
tags:
- readability
- easyCBM
- validation
title: Model Validation with easyCBM
---

# easyCBM
Educational researchers within the Behavioral Research and Teaching Facility (BRT) at the University of Oregon developed easyCBM as a tool for assessing students on their reading skills. These short assessments give teachers insight into areas of struggle for their students, what supports teachers can provide, and how effective their overall teaching is. The [easyCBM program](easycbm.com "easyCBM program") uses reading passages that have been developed based on the "Big Five" constructs reported in the 2000 National Reading Panel report. Each of these passages is specific to the grade level assigned; students are given a minute to read their assigned passage. With the permission of researchers, this project uses easyCBM passages to validate the readability model developed in this project. 

## Data
The original data set contained student data associated with easyCBM passages (referred to as forms). This data contained just over 5 million student entries. Individual student data was masked and anonymized. Data was cleaned where form, grade, and year was pulled from the filepath. 
```
files <- dir_ls(here::here("easyCBM", "itemdata"), recurse = TRUE, type = "file")

easycbm_data <- purrr::map_df(
  files,
  read_csv,
  col_types = cols(.default = "c"),
  .id = "filepath"
)

year <- gsub(".+itemdata/(.+)/.+", "\\1", easycbm_data$filepath)
grade <- gsub(".+prf-grade-(.+)-form.+", "\\1", easycbm_data$filepath)
form <- gsub(".+form-(.+)-item.+", "\\1", easycbm_data$filepath)

easycbm_data <- easycbm_data %>% 
  mutate(
    year = year,
    grade = grade,
    form = form) %>% 
  select(year, grade, form, everything()) %>% 
  select(-.data$filepath)

```
Our model requires information regarding the correct words read per minute (WRC). To accommodate this, averages were taken across each year of data, within each form by grade. This resulted in 159 rows of data. 

```
sumdata <- easycbm_data %>%
  mutate(
    score = parse_number(score)) %>%
  group_by(
    grade,
    form) %>% 
  summarise_at(vars("score"), mean)

sds <- tapply(
  parse_number(easycbm_data$score), 
  list(easycbm_data$grade, easycbm_data$form),
  sd,
  na.rm = TRUE
  )
```
Once the student data was finalized, the text data per passage was added. This created a column in the data set for the text files themselves.   
```
cbmpassages <- tibble(
  path = dir_ls(here::here("easyCBM", "prf_txt")),
  form = gsub(".+PRF_?(.+)_.+", "\\1", path),
  grade = as.numeric(gsub(".+Gr(\\d\\d?).+", "\\1", path)),
  text = map(path, read_file)
)

cbmpassages %>% 
  select(grade) %>% 
  dplyr::count(grade)


passages<- cbmpassages %>% 
  dplyr::filter(!form %in% c("18", "19", "20")) %>% 
  mutate(
    grade = as.character(grade),
    form = tolower(form)
  )

passages$text <- map_chr(passages$text, ~.x)

p <- left_join(x= easycbm, y = passages, by = c("form", "grade"))

d <- na.omit(p)

fulleasycbm <- subset(d, select = -path)
```

The final data frame is displayed below. The last column `readability score` was created as an empty column to be filled with the predicted readability scores post-running the model. 

| form_grade  | form_number | score |     sd    |     n    | excerpt | readability score |
| ----------- | ----------- | ----- | --------- | -------- | ------- | ----------------- |
| grade level | form number | mean  | standard  | total    |  form   |  model produced   |
|   of form   | assigned    | WRC   | deviation | students |  text   |  passage level    |

## Results
The readability score predictions were applied to the [Bradley-Terry model](https://www.r-bloggers.com/2022/02/what-is-the-bradley-terry-model/ "Bradley-Terry model"). This is a probability model used for predicting the outcome of a paired comparison. This model results in a number estimate equal to the probably that the comparison turns out to be true. In our case, this is comparisons made between easyCBM excerpts to determine their predicted readability. According to the owner of the readability data set from the original Kaggle competition ([Scott Crossley](https://www.kaggle.com/c/commonlitreadabilityprize/discussion/240423 "Scott Crossley")), the readability score is a result of a Bradley-Terry analysis of over 111,000 pairwise comparisons between excerpts. 

The first time running the easyCBM data through the model and plotting the correlation between the predictions and student scores resulted in the discovery of an outlier. This outlier was form 7 from grade 3; the average WRC was 431.04619 across students, which is over 200 points higher than the rest of the forms. Removing this outlier resulted in the below plot with an average correlation of -0.5569613. 

![easyCBM Readability Predictions](/Plots/readprediction.png)

Bradley-Terry analysis developed a range between -2 and 1 for this data set. A lower value indicates the text is more difficult to read, whereas a higher value indicates an easier text. From the plot above, it is evident that the more words read correctly per minute on average, the more difficult the text was to read. The downward slope indicates that as less words are read, the easier the passage readability overall. This observation may be congruent with passage length in general. A longer passage in theory will be more difficult. Further investigation into the effects of passage length on overall readability will be conducted. 

Please move on to the final post in this series **Model Validation with CORE** to read about a second round of model validation using data from the CORE project. For more information regarding Bradley-Terry analysis used here, refer to Dr. Zopluoglu's lecture [Introduction to Toy Datasets](https://ml-21.netlify.app/notes/lecture-1a.html "Introduction to Toy Datasets").
