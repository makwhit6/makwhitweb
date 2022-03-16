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
Educational researchers within the Behavioral Research and Teaching Facility (BRT) at the University of Oregon developed easyCBM as a tool for assessing students on their reading skills. These short assessments give teachers insight into areas of struggle for their students, what supports teachers can provide, and how effective their overall teaching is. The [easyCBM program](easycbm.com "easyCBM program") uses reading passages that have been developed based on the "Big Five" constructs reported in the 2000 National Reading Panel report. Each of these passages is specific to the grade level assigned; students are given a minute to read their assigned passage. With the permission of researchers, this project uses easyCBM passages to train and test the readability model. 

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
The first time running the easyCBM data through the model and plotting the correlation between the predictions and student scores resulted in the discovery of an outlier. This outlier was form 7 from grade 3; the average WRC was 431.04619 across students, which is over 200 points higher than the rest of the forms. Removing this outlier resulted in the below plot with an average correlation of -0.5569613. 


[easyCBM Readability Predictions](Plots/readabilitypredictions.jpeg)



