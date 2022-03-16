---
author: Makayla Whitney
date: "2022-02-03"
description: model development process
tags:
- markdown
- text
series:
- series-setup
tags:
- readability
- CORE
- validation
title: Model Validation with CORE
---

# CORE
[Computerized Oral Reading Evaluation (CORE)](https://jnese.github.io/core-blog/about.html "Computerized Oral Reading Evaluation (CORE)") is a project led by Dr. Joe Nese housed within the Behavioral Research and Teaching Facility at the University of Oregon. This project is focused on using computerized assessment of oral reading fluency to analyze aspects of the reading process in 2nd - 4th graders. As part of this project, one of the main focuses is the connection between [prosody](https://jnese.github.io/coreprosody/ "[prosody") and oral reading practices. Students were given a passage to read with no time constraints; these passages were relatively short at no more than 5 sentences each. With the permission of researchers, this project uses CORE passages to validate the readability model developed in this project. 

## Data
The original data set contained student data (ranging from 2nd - 4th grade) associated with CORE passages (referred to as forms). This data contained just shy of 15,000 student entries. Individual student data was masked and anonymized. Data was cleaned where form, grade, and year was pulled from the filepath. Average words read correctly per minute across each form categorized by grade was previously calculate and imported into the data frame. Passage data was pulled into the environment as a tibble and merged with the student data frame. 
```
corepassages <- tibble(
  path = fs::dir_ls(here::here("CORE", "passage_text")),
  form_number = str_extract(path, "(?<=text/)(.*?)(?=.txt)"),
  text = map(path, read_file)
)

corepassages <- corepassages %>%
  mutate(form = form_number)%>% 
  select(text, form) %>% 
  select(-.data$path)

core <- merge(x = stu_data, y = corepassages, by = c("form_number"))

fullcore <- subset(core, select = -path)
```

The final data frame is displayed below. The last column `prediction` was created as an empty column to be filled with the predicted readability scores post-running the model. 

| form_grade  | form_number | score |     sd    |     n    | excerpt |     prediction    |
| ----------- | ----------- | ----- | --------- | -------- | ------- | ----------------- |
| grade level | form number | mean  | standard  | total    |  form   |  model produced   |
|   of form   | assigned    | WRC   | deviation | students |  text   |  passage level    |

## Results

The readability score predictions were applied to the [Bradley-Terry model](https://www.r-bloggers.com/2022/02/what-is-the-bradley-terry-model/ "Bradley-Terry model"). This is a probability model used for predicting the outcome of a paired comparison, and was used in the previous example with easyCBM passages. This model results in a number estimate equal to the probably that the comparison turns out to be true. In our case, this is comparisons made between CORE excerpts to determine their predicted readability. According to the owner of the readability data set from the original Kaggle competition ([Scott Crossley](https://www.kaggle.com/c/commonlitreadabilityprize/discussion/240423 "Scott Crossley")), the readability score is a result of a Bradley-Terry analysis of over 111,000 pairwise comparisons between excerpts.

The first time running the easyCBM data through the model and plotting the correlation between the predictions and student scores resulted in the following plot. Data has an average correlation of -0.3419986. 

![CORE Readability Predictions](/Plots/corepredictions.png)

Bradley-Terry analysis developed a range between 0 and 4 for this data set. A lower value indicates the text is more difficult to read, whereas a higher value indicates an easier text. From the plot above, it is difficult to determine if there is a correlation between passage length and overall difficulty. Typically, the longer the text, the lower the level of readability. This plot seems to display three groups of words read correctly per minute with data spread across the analysis range. This could be due to the lack of constraints on students time allotted for the assessment.Further investigation into the effects of passage length and time allotted for each student on overall readability will be conducted. 

For more information regarding Bradley-Terry analysis used here, refer to Dr. Zopluoglu's lecture [Introduction to Toy Datasets](https://ml-21.netlify.app/notes/lecture-1a.html "Introduction to Toy Datasets").