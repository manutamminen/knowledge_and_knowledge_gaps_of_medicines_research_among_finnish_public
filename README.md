<!-- badges: start -->
[![DOI](https://zenodo.org/badge/255566850.svg)](https://zenodo.org/badge/latestdoi/255566850)
<!-- badges: end -->

# A quantitative online survey of self-perceived knowledge and knowledge gaps of medicines research and development among Finnish general public

## Data - Knowledge about medicines research and development 

Questionnaire raw data available at [https://raw.githubusercontent.com/manutamminen/knowledge_and_knowledge_gaps_of_medicines_research_among_finnish_public/master/questionnaire.csv](https://raw.githubusercontent.com/manutamminen/knowledge_and_knowledge_gaps_of_medicines_research_among_finnish_public/master/questionnaire.csv)

### Background variables

| Variable name| Explanation |
| --- | --- |
| Q013        | Age group                                                       |
| TT11edu1    | Education                                                       |
| T_gender    | Gender                                                          |
| T_Q001_2cat | Are you taking part or have you taken part in medical research? |

### Age responses

| Variable name | Level | Explanation |
| --- | --- | --- |
| Q013 | 1 | 16-24 years |
| Q013 | 2 | 25-34 years |
| Q013 | 3 | 35-44 years |
| Q013 | 4 | 45-54 years |
| Q013 | 5 | 55-64 years |

### Education responses

| Variable name | Level | Explanation |
| --- | --- | --- |
| TT11edu1 | 1 | Vocational education and training                |
| TT11edu1 | 2 | High school degree                               |
| TT11edu1 | 3 | Post-secondary education                         |
| TT11edu1 | 4 | Bachelor's degree or equivalent                  |
| TT11edu1 | 5 | Master's degree or higher                        |

### Gender responses

| Variable name | Level | Explanation |
| --- | --- | --- |
| T_gender | 1 | Female |
| T_gender | 2 | Male   |

### Medical research responses

| Variable name | Level | Explanation |
| --- | --- | --- |
| T_Q001_2cat | 1 | Yes |
| T_Q001_2cat | 2 | No  |

### Current knowledge

| Variable name| Explanation |
| --- | --- |
| Q002_1_slice | Medicines development, drug discovery |
| Q002_2_slice | Medicines safety                      |
| Q002_3_slice | Patients role and responsibilities    |
| Q002_4_slice | Personalized medicine                 |
| Q002_5_slice | Predictive medicine                   |
| Q002_6_slice | Clinical trials                       |
| Q002_7_slice | HTA                                   |
| Q002_8_slice | Pharmacoeconomics, cost-effectiveness |
| Q002_9_slice | Regulation                            |

### Knowledge responeses

| Variable level| Explanation |
| --- | --- |
| 1 | No knowledge        |
| 2 | Poor knowledge      |
| 3 | Average knowledge   |
| 4 | Good knowledge      |
| 5 | Very good knowledge |

### Interest in learning more

| Variable name| Explanation |
| --- | --- |
| Q003_1_slice | Medicines development, drug discovery |
| Q003_2_slice | Medicines safety                      |
| Q003_3_slice | Patients role and responsibilities    |
| Q003_4_slice | Personalized medicine                 |
| Q003_5_slice | Predictive medicine                   |
| Q003_6_slice | Clinical trials                       |
| Q003_7_slice | HTA                                   |
| Q003_8_slice | Pharmacoeconomics, cost-effectiveness |
| Q003_9_slice | Regulation                            |

### Interest responses

| Variable level| Explanation |
| --- | --- |
| 1 | No interest       |
| 2 | Marginal interest |
| 3 | Some interest     |
| 4 | Interested        |
| 5 | Highly interested |


## Analyses


```R

library(tidyverse)
library(ggh4x)
library(readxl)
library(broom)
library(janitor)

all_data <-
    read_csv("questionnaire.csv")

######################
## Chi square tests ##
######################

q00p_responses <- 
    all_data %>% 
    dplyr::select(Q00P, Q013, T_gender, TT11edu1, T_Q001_2cat) %>% 
    mutate(Q00P = case_when(
               Q00P %in% 1:3 ~ 0,
               Q00P %in% 4:5 ~ 1,
               TRUE ~ -1)) %>% 
    filter(Q00P != -1)

# Experience in medical research
q00p_responses %>%
    tabyl(T_Q001_2cat, Q00P) %>% 
    mutate(rsums = rowSums(.[-1]),
           perc = `1` / rsums * 100) %>% 
    dplyr::select(2:3) %>% 
    as.matrix %>% 
    chisq.test

# Gender
q00p_responses %>%
    tabyl(T_gender, Q00P) %>% 
    mutate(rsums = rowSums(.[-1]),
           perc = `1` / rsums * 100) %>% 
    dplyr::select(2:3) %>% 
    as.matrix %>% 
    chisq.test

# Age
 q00p_responses %>%
    tabyl(Q013, Q00P) %>% 
    mutate(rsums = rowSums(.[-1]),
           perc = `1` / rsums * 100) %>% 
    dplyr::select(2:3) %>% 
    as.matrix %>% 
    chisq.test

# Education
 q00p_responses %>%
    tabyl(TT11edu1, Q00P) %>% 
    mutate(rsums = rowSums(.[-1]),
           perc = `1` / rsums * 100) %>% 
    dplyr::select(2:3) %>% 
    as.matrix %>% 
    chisq.test

##################################
## Logistic regression analyses ##
##################################

prep_logistic_regression <- function(dep, data = binary_responses)
{
    model <-
        paste(dep, "~ T_gender + Q013 + TT11edu1 + T_Q001_2cat") %>%
        as.formula
    glm(formula = model,
        family = binomial(link = 'logit'), 
        data = data) %>%
        tidy %>%
        mutate(Dep_variable=dep)
}

binary_responses <- 
    all_data %>% 
    dplyr::select(contains("slice"), Q013, T_gender, TT11edu1, T_Q001_2cat) %>% 
    mutate(across(contains("slice"),
                  ~ case_when(. < 4 ~ 0,
                              . >= 4 ~ 1))) %>% 
    filter(complete.cases(.))

logistic_test_results <- 
    all_data %>% 
    dplyr::select(contains("slice")) %>% 
    names %>% 
    map_df( ~ prep_logistic_regression(., data = binary_responses))

logistic_test_results %>%
    filter(p.value < 0.05) %>% 
    pivot_wider(id_cols = "Dep_variable",
                names_from = "term",
                values_from = "p.value") %>% 
    data.frame

logistic_test_results %>%
  filter(p.value < 0.05) %>%
  write_csv("sig_responses.csv")

############################
## Prepare numeric tables ##
############################

long_binary_data <- 
    all_data %>% 
    dplyr::select(contains("slice"), Q013, T_gender, TT11edu1, T_Q001_2cat) %>% 
    mutate(across(contains("slice"),
                  ~ case_when(. %in% 1:3 ~ 0,
                              . %in% 4:5 ~ 1,
                              TRUE ~ -1))) %>%
    pivot_longer(
        cols=c("Q013", "TT11edu1"),
        names_to="Question1",
        values_to="Answer1") %>% 
    pivot_longer(
        cols=c("T_gender", "T_Q001_2cat"),
        names_to="Question2",
        values_to="Answer2") %>% 
    pivot_longer(
        cols=c("Q002_1_slice", "Q002_2_slice", "Q002_3_slice",
               "Q002_4_slice", "Q002_5_slice", "Q002_6_slice", 
               "Q002_7_slice", "Q002_8_slice", "Q002_9_slice",
               "Q003_1_slice", "Q003_2_slice", "Q003_3_slice",
               "Q003_4_slice", "Q003_5_slice", "Q003_6_slice", 
               "Q003_7_slice", "Q003_8_slice", "Q003_9_slice"),
        names_to="Question3",
        values_to="Answer3")

long_binary_data %>% 
    filter(Question2 == "T_gender",
           Answer3 != -1) %>% 
    unite(Slice_Question, Question3, Answer3, sep = "_") %>% 
    dplyr::select(Question2, Answer2, Slice_Question) %>% 
    count(Answer2, Slice_Question) %>% 
    pivot_wider(id_cols = Slice_Question,
                names_from = Answer2,
                values_from = n) %>% 
    mutate(across(is.numeric, ~ . / 2)) %>% 
    data.frame

long_binary_data %>% 
    filter(Question2 == "T_Q001_2cat",
           Answer3 != -1) %>% 
    unite(Slice_Question, Question3, Answer3, sep = "_") %>% 
    dplyr::select(Question2, Answer2, Slice_Question) %>% 
    count(Answer2, Slice_Question) %>% 
    pivot_wider(id_cols = Slice_Question,
                names_from = Answer2,
                values_from = n) %>% 
    mutate(across(is.numeric, ~ . / 2)) %>% 
    data.frame

long_binary_data %>% 
    filter(Question1 == "Q013",
           Answer3 != -1) %>% 
    unite(Slice_Question, Question3, Answer3, sep = "_") %>% 
    dplyr::select(Question1, Answer1, Slice_Question) %>% 
    count(Answer1, Slice_Question) %>% 
    pivot_wider(id_cols = Slice_Question,
                names_from = Answer1,
                values_from = n) %>% 
    mutate(across(is.numeric, ~ . / 2)) %>% 
    data.frame

long_binary_data %>% 
    filter(Question1 == "TT11edu1",
           Answer3 != -1) %>% 
    unite(Slice_Question, Question3, Answer3, sep = "_") %>% 
    dplyr::select(Question1, Answer1, Slice_Question) %>% 
    count(Answer1, Slice_Question) %>% 
    pivot_wider(id_cols = Slice_Question,
                names_from = Answer1,
                values_from = n) %>% 
    mutate(across(is.numeric, ~ . / 2)) %>% 
    data.frame

#########################
## Prepare the figures ##
#########################

long_data <- 
    all_data %>% 
    mutate(across(contains("slice") & contains("Q002"),
                  ~ case_when(. < 4 ~ 0,
                              . >= 4 ~ 1))) %>% 
    mutate(across(contains("slice") & contains("Q003"),
                  ~ case_when(. < 4 ~ 2,
                              . >= 4 ~ 3))) %>% 
    pivot_longer(
        cols=c("Q013", "TT11edu1"),
        names_to="Question1",
        values_to="Answer1") %>% 
    pivot_longer(
        cols=c("T_gender", "T_Q001_2cat"),
        names_to="Question2",
        values_to="Answer2") %>% 
    pivot_longer(
        cols=c("Q002_1_slice", "Q002_2_slice", "Q002_3_slice",
               "Q002_4_slice", "Q002_5_slice", "Q002_6_slice", 
               "Q002_7_slice", "Q002_8_slice", "Q002_9_slice",
               "Q003_1_slice", "Q003_2_slice", "Q003_3_slice",
               "Q003_4_slice", "Q003_5_slice", "Q003_6_slice", 
               "Q003_7_slice", "Q003_8_slice", "Q003_9_slice"),
        names_to="Question3",
        values_to="Answer3") %>% 
    mutate(
        Question_type = ifelse(str_detect(Question3, "Q002"), "Pre", "Post"),
        Answer1=as.factor(Answer1),
        Answer2=as.factor(Answer2), 
        Answer3=as.factor(Answer3))

pdf("age_education2.pdf")
long_data %>% 
    filter(complete.cases(.)) %>% 
    ggplot(data=., aes(x=1, fill=Answer3)) +
    geom_bar(position="dodge") +
    geom_text(stat='count', aes(label=..count..), vjust=-1) + 
    facet_grid(Question3 ~ Question1 + Question_type + Answer1, scales="free_x") + 
    guides(fill=FALSE) + 
    scale_y_continuous(labels = scales::percent) + 
    theme(strip.text.y = element_text(angle = 0, size = 4),
          strip.text.x = element_text(size = 4))
dev.off()

pdf("gender_participation2.pdf")
long_data %>% 
    filter(complete.cases(.)) %>% 
    ggplot(data=., aes(x=1, fill=Answer3)) +
    geom_bar(position="dodge") +
    geom_text(stat='count', aes(label=..count..), vjust=-1) + 
    facet_grid(Question3 ~ Question2 + Question_type + Answer2, scales="free_x") + 
    guides(fill=FALSE) + 
    scale_y_continuous(labels = scales::percent) + 
    theme(strip.text.y = element_text(angle = 0, size = 4),
          strip.text.x = element_text(size = 4))
dev.off()

pdf("all_data.pdf")
long_data %>% 
    filter(complete.cases(.)) %>% 
    ggplot(data=., aes(x=1, fill=Answer3)) +
    geom_bar(position="dodge") +
    facet_grid(Question3 ~ Question_type, scales="free_x") + 
    guides(fill=FALSE) + 
    scale_y_continuous(labels = scales::percent) + 
    theme(strip.text.y = element_text(angle = 0, size = 4),
          strip.text.x = element_text(size = 4))
dev.off()
```

## Session info 

R version 3.6.0 (2019-04-26)
Platform: x86_64-apple-darwin13.4.0 (64-bit)
Running under: macOS  10.15.4

Matrix products: default
BLAS/LAPACK: /Users/mavatam/miniconda3/lib/R/lib/libRblas.dylib

locale:
[1] C

attached base packages:
[1] stats     graphics  grDevices utils     datasets  methods   base     

other attached packages:
 [1] janitor_2.0.1   broom_0.5.5     readxl_1.3.1    ggh4x_0.1.1    
 [5] forcats_0.5.0   stringr_1.4.0   dplyr_0.8.5     purrr_0.3.3    
 [9] readr_1.3.1     tidyr_1.0.2     tibble_3.0.0    ggplot2_3.3.0  
[13] tidyverse_1.3.0

loaded via a namespace (and not attached):
 [1] Rcpp_1.0.1       cellranger_1.1.0 pillar_1.4.3     compiler_3.6.0  
 [5] dbplyr_1.4.2     tools_3.6.0      lubridate_1.7.4  jsonlite_1.6.1  
 [9] lifecycle_0.2.0  nlme_3.1-141     gtable_0.3.0     lattice_0.20-41 
[13] pkgconfig_2.0.3  rlang_0.4.5      reprex_0.3.0     rstudioapi_0.11 
[17] cli_2.0.2        DBI_1.1.0        haven_2.2.0      withr_2.1.2     
[21] xml2_1.3.1       httr_1.4.1       fs_1.2.7         generics_0.0.2  
[25] vctrs_0.2.4      hms_0.5.3        grid_3.6.0       tidyselect_1.0.0
[29] snakecase_0.11.0 glue_1.4.0       R6_2.4.1         fansi_0.4.1     
[33] modelr_0.1.6     magrittr_1.5     backports_1.1.6  scales_1.0.0    
[37] ellipsis_0.3.0   rvest_0.3.5      assertthat_0.2.1 colorspace_1.4-1
[41] stringi_1.4.3    munsell_0.5.0    crayon_1.3.4    
