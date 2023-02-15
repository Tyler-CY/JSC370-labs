Lab 06 - Regular Expressions and Web Scraping
================

# Learning goals

- Use a real world API to make queries and process the data.
- Use regular expressions to parse the information.
- Practice your GitHub skills.

# Lab description

In this lab, we will be working with the [NCBI
API](https://www.ncbi.nlm.nih.gov/home/develop/api/) to make queries and
extract information using XML and regular expressions. For this lab, we
will be using the `httr`, `xml2`, and `stringr` R packages.

This markdown document should be rendered using `github_document`
document ONLY and pushed to your *JSC370-labs* repository in
`lab06/README.md`.

## Question 1: How many sars-cov-2 papers?

Build an automatic counter of sars-cov-2 papers using PubMed. You will
need to apply XPath as we did during the lecture to extract the number
of results returned by PubMed in the following web address:

    https://pubmed.ncbi.nlm.nih.gov/?term=sars-cov-2

Complete the lines of code:

``` r
# Downloading the website
pub <- 'https://pubmed.ncbi.nlm.nih.gov/?term=sars-cov-2'
website <- xml2::read_html(pub)
# Finding the counts
counts <- xml2::xml_find_first(website,
                               "/html/body/main/div[9]/div[2]/div[2]/div[1]/div[1]/span")
# Turning it into text
counts <- as.character(counts)
# Extracting the data using regex
stringr::str_extract(counts, "[0-9,]+")
```

    ## [1] "192,677"

- How many sars-cov-2 papers are there?

There are 192, 677 papers.

Don’t forget to commit your work!

## Question 2: Academic publications on COVID19 and Hawaii

Use the function `httr::GET()` to make the following query:

1.  Baseline URL:
    <https://eutils.ncbi.nlm.nih.gov/entrez/eutils/esearch.fcgi>

2.  Query parameters:

    - db: pubmed
    - term: covid19 hawaii
    - retmax: 1000

The parameters passed to the query are documented
[here](https://www.ncbi.nlm.nih.gov/books/NBK25499/).

``` r
base_url <- 'https://eutils.ncbi.nlm.nih.gov/entrez/eutils/esearch.fcgi'

library(httr)
query_ids <- GET(
  url   = base_url,
  query = list(
    db = 'pubmed',
    term = 'covid19 hawaii',
    retmax = 1000
  )
)
# Extracting the content of the response of GET
ids <- httr::content(query_ids)
```

The query will return an XML object, we can turn it into a character
list to analyze the text directly with `as.character()`. Another way of
processing the data could be using lists with the function
`xml2::as_list()`. We will skip the latter for now.

Take a look at the data, and continue with the next question (don’t
forget to commit and push your results to your GitHub repo!).

## Question 3: Get details about the articles

The Ids are wrapped around text in the following way:
`<Id>... id number ...</Id>`. we can use a regular expression that
extract that information. Fill out the following lines of code:

``` r
# Turn the result into a character vector
ids <- as.character(ids)
# Find all the ids 
ids <- stringr::str_extract_all(ids, "<Id>\\d*</Id>")[[1]]
# Remove all the leading and trailing <Id> </Id>. Make use of "|"
ids <- stringr::str_remove_all(ids, "<Id>|</Id>")
```

With the ids in hand, we can now try to get the abstracts of the papers.
As before, we will need to coerce the contents (results) to a list
using:

1.  Baseline url:
    <https://eutils.ncbi.nlm.nih.gov/entrez/eutils/efetch.fcgi>

2.  Query parameters:

    - db: pubmed
    - id: A character with all the ids separated by comma, e.g.,
      “1232131,546464,13131”
    - retmax: 1000
    - rettype: abstract

**Pro-tip**: If you want `GET()` to take some element literal, wrap it
around `I()` (as you would do in a formula in R). For example, the text
`"123,456"` is replaced with `"123%2C456"`. If you don’t want that
behavior, you would need to do the following `I("123,456")`.

``` r
base_url <- 'https://eutils.ncbi.nlm.nih.gov/entrez/eutils/efetch.fcgi'

library(httr)
publications <- GET(
  url   = base_url,
  query = list(
    db = "pubmed",
    retmax = 1000,
    rettype = 'abstract',
    id = paste(ids, collapse=',')
    )
)

# status_code(publications)

# Turning the output into character vector
publications <- httr::content(publications)
publications_txt <- as.character(publications)
```

With this in hand, we can now analyze the data. This is also a good time
for committing and pushing your work!

## Question 4: Distribution of universities, schools, and departments

Using the function `stringr::str_extract_all()` applied on
`publications_txt`, capture all the terms of the form:

1.  University of …
2.  … Institute of …

Write a regular expression that captures all such instances

``` r
institution <- stringr::str_extract_all(
  publications_txt,
  "University\\s*of\\s*[[:alpha:]\\s]+|[[:alpha:]\\s]+\\s*Institute\\s*of [[:alpha:]\\s]+"
  ) 
institution <- unlist(institution)
head(as.data.frame(table(institution)))
```

    ##                                                                                                                           institution
    ## 1                                                                                    Beijing Institute of Pharmacology and Toxicology
    ## 2                                                                                                          Berlin Institute of Health
    ## 3                                                                      Breadfruit Institute of the National Tropical Botanical Garden
    ## 4  Dr Ganesan reported receiving grants from the Defense Health Program and the National Institute of Allergy and Infectious Disease 
    ## 5                                                                                                 Institute of Allied Health Sciences
    ## 6                                                                                                    Institute of Atmospheric Physics
    ##   Freq
    ## 1    2
    ## 2    4
    ## 3    1
    ## 4    1
    ## 5    1
    ## 6    1

Repeat the exercise and this time focus on schools and departments in
the form of

1.  School of …
2.  Department of …

And tabulate the results

``` r
schools_and_deps <- stringr::str_extract_all(
  publications_txt,
  "School\\s*of\\s*[[:alpha:]\\s]+|Department\\s*of\\s*[[:alpha:]\\s]+"
  )
head(as.data.frame(table(schools_and_deps)))
```

    ##                                    schools_and_deps Freq
    ## 1                   Department of Ageing and Health    1
    ## 2 Department of Agricultural and Resource Economics    1
    ## 3                         Department of Agriculture    6
    ## 4   Department of Agriculture and Consumer Services    1
    ## 5        Department of Agriculture Research Service    1
    ## 6                             Department of Anatomy   30

## Question 5: Form a database

We want to build a dataset which includes the title and the abstract of
the paper. The title of all records is enclosed by the HTML tag
`ArticleTitle`, and the abstract by `Abstract`.

Before applying the functions to extract text directly, it will help to
process the XML a bit. We will use the `xml2::xml_children()` function
to keep one element per id. This way, if a paper is missing the
abstract, or something else, we will be able to properly match PUBMED
IDS with their corresponding records.

``` r
pub_char_list <- xml2::xml_children(publications)
pub_char_list <- sapply(pub_char_list, as.character)
```

Now, extract the abstract and article title for each one of the elements
of `pub_char_list`. You can either use `sapply()` as we just did, or
simply take advantage of vectorization of `stringr::str_extract`

``` r
abstracts <- stringr::str_extract(pub_char_list, "<Abstract>(\\n|.)*</Abstract>")
# abstracts <- stringr::str_remove_all(abstracts, "<Abstract>|</Abstract>")
abstracts <- stringr::str_remove_all(abstracts, '<(/)?[A-Za-z]+\\s?[a-zA-Z0-9=\\"\\s]+>')
abstracts <- stringr::str_replace_all(abstracts, "\\s+", ' ')
table(is.na(abstracts))
```

    ## 
    ## FALSE  TRUE 
    ##   285    53

- How many of these don’t have an abstract?

53 of them don’t have an abstract.

Now, the title

``` r
titles <- stringr::str_extract(pub_char_list, "<ArticleTitle>(\\n|.)*</ArticleTitle>")
# titles <- stringr::str_remove_all(titles, "<ArticleTitle>|</ArticleTitle>")
titles <- stringr::str_remove_all(titles, '<(/)?[A-Za-z]+\\S?[a-zA-Z0-9=\\"\\s]+>')

titles <- stringr::str_replace_all(titles, "\\s+", ' ')
table(is.na(titles))
```

    ## 
    ## FALSE 
    ##   338

- How many of these don’t have a title ?

None. All of them have a title.

Finally, put everything together into a single `data.frame` and use
`knitr::kable` to print the results

``` r
database <- data.frame(
  titles, abstracts
)
head(knitr::kable(database))
```

    ## [1] "|titles                                                                                                                                                                                                                               |abstracts                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |"      
    ## [2] "|:------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|:------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|"      
    ## [3] "|A machine learning approach identifies distinct early-symptom cluster phenotypes which correlate with hospitalization, failure to return to activities, and prolonged COVID-19 symptoms.                                             |Accurate COVID-19 prognosis is a critical aspect of acute and long-term clinical management. We identified discrete clusters of early stage-symptoms which may delineate groups with distinct disease severity phenotypes, including risk of developing long-term symptoms and associated inflammatory profiles. 1,273 SARS-CoV-2 positive U.S. Military Health System beneficiaries with quantitative symptom scores (FLU-PRO Plus) were included in this analysis. We employed machine-learning approaches to identify symptom clusters and compared risk of hospitalization, long-term symptoms, as well as peak CRP and IL-6 concentrations. We identified three distinct clusters of participants based on their FLU-PRO Plus symptoms: cluster 1 (\"Nasal cluster\") is highly correlated with reporting runny/stuffy nose and sneezing, cluster 2 (\"Sensory cluster\") is highly correlated with loss of smell or taste, and cluster 3 (\"Respiratory/Systemic cluster\") is highly correlated with the respiratory (cough, trouble breathing, among others) and systemic (body aches, chills, among others) domain symptoms. Participants in the Respiratory/Systemic cluster were twice as likely as those in the Nasal cluster to have been hospitalized, and 1.5 times as likely to report that they had not returned-to-activities, which remained significant after controlling for confounding covariates (P &lt; 0.01). Respiratory/Systemic and Sensory clusters were more likely to have symptoms at six-months post-symptom-onset (P = 0.03). We observed higher peak CRP and IL-6 in the Respiratory/Systemic cluster (P &lt; 0.01). We identified early symptom profiles potentially associated with hospitalization, return-to-activities, long-term symptoms, and inflammatory profiles. These findings may assist in patient prognosis, including prediction of long COVID risk. Copyright: This is an open access article, free of all copyright, and may be freely reproduced, distributed, transmitted, modified, built upon, or otherwise used by anyone for any lawful purpose. The work is made available under the Creative Commons CC0 public domain dedication.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |"
    ## [4] "|Barriers and Challenges for Career Milestones Among Faculty Mentees.                                                                                                                                                                 |'Critical' career milestones for faculty (e.g., tenure, securing grant funding) relate to career advancement, job satisfaction, service/leadership, scholarship/research, clinical or teaching activities, professionalism, compensation, and work-life balance. However, barriers and challenges to these milestones encountered by junior faculty have been inadequately studied, particularly those affecting underrepresented minorities in science (URM-S). Additionally, little is known about how barriers and challenges to career milestones have changed during the COVID-19 pandemic for URM-S and non-URM faculty mentees in science. In this study, we conducted semi-structured interviews with 31 faculty mentees from four academic institutions (located in New Mexico, Arizona, Idaho, and Hawaii), including 22 URM-S (women or racial/ethnic). Respondents were given examples of 'critical' career milestones and were asked to identify and discuss barriers and challenges that they have encountered or expect to encounter while working toward achieving these milestones. We performed thematic descriptive analysis using NVivo software in an iterative, team-based process. Our preliminary analysis identified five key themes that illustrate barriers and challenges encountered: Job and career development, Discrimination and a lack of workplace diversity; Lack of interpersonal relationships and inadequate social support at the workplace; Personal and family matters; and Unique COVID-19-related issues. COVID-19 barriers and challenges were related to online curriculum creation and administration, interpersonal relationship development, inadequate training/service/conference opportunities, and disruptions in childcare and schooling. Although COVID-19 helped create new barriers and challenges for junior faculty mentees, traditional barriers and challenges for 'critical' career milestones continue to be reported among our respondents. URM-S respondents also identified discrimination and diversity-related barriers and challenges. Subsequent interviews will focus on 12-month and 24-month follow-ups and provide additional insight into the unique challenges and barriers to 'critical' career milestones that URM and non-URM faculty in science have encountered during the unique historical context of the COVID-19 pandemic.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |"      
    ## [5] "|COVID-19 Information on YouTube: Analysis of Quality and Reliability of Videos in Eleven Widely Spoken Languages across Africa.                                                                                                      |Whilst the coronavirus disease 2019 (COVID-19) vaccination rollout is well underway, there is a concern in Africa where less than 2% of global vaccinations have occurred. In the absence of herd immunity, health promotion remains essential. YouTube has been widely utilised as a source of medical information in previous outbreaks and pandemics. There are limited data on COVID-19 information on YouTube videos, especially in languages widely spoken in Africa. This study investigated the quality and reliability of such videos. Medical information related to COVID-19 was analysed in 11 languages (English, isiZulu, isiXhosa, Afrikaans, Nigerian Pidgin, Hausa, Twi, Arabic, Amharic, French, and Swahili). Cohen's Kappa was used to measure inter-rater reliability. A total of 562 videos were analysed. Viewer interaction metrics and video characteristics, source, and content type were collected. Quality was evaluated using the Medical Information Content Index (MICI) scale and reliability was evaluated by the modified DISCERN tool. Kappa coefficient of agreement for all languages was <i>p</i> &lt; 0.01. Informative videos (471/562, 83.8%) accounted for the majority, whilst misleading videos (12/562, 2.13%) were minimal. Independent users (246/562, 43.8%) were the predominant source type. Transmission of information (477/562 videos, 84.9%) was most prevalent, whilst content covering screening or testing was reported in less than a third of all videos. The mean total MICI score was 5.75/5 (SD 4.25) and the mean total DISCERN score was 3.01/5 (SD 1.11). YouTube is an invaluable, easily accessible resource for information dissemination during health emergencies. Misleading videos are often a concern; however, our study found a negligible proportion. Whilst most videos were fairly reliable, the quality of videos was poor, especially noting a dearth of information covering screening or testing. Governments, academic institutions, and healthcare workers must harness the capability of digital platforms, such as YouTube to contain the spread of misinformation. Copyright © 2023 Kapil Narain et al.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |"      
    ## [6] "|BNT162b2 against COVID-19-associated Emergency Department and Urgent Care Visits among Children 5-11 Years of Age: a Test Negative Design.                                                                                           |In a 1:1 matched test-negative design among 5-11-year-olds in the Kaiser Permanente Southern California health system (n=3984), BNT162b2 effectiveness against omicron-related emergency department or urgent care encounters was 60% [95%CI: 47-69] &lt;3 months post-dose-two and 28% [8-43] after ≥3 months. A booster improved protection to 77% [53-88]. © The Author(s) 2023. Published by Oxford University Press on behalf of The Journal of the Pediatric Infectious Diseases Society.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |"

Done! Knit the document, commit, and push.

## Final Pro Tip (optional)

You can still share the HTML document on github. You can include a link
in your `README.md` file as the following:

``` md
View [here](https://cdn.jsdelivr.net/gh/Tyler-CY/JSC370-labs/lab06/JSC370_lab06.html) 
```

For example, if we wanted to add a direct link the HTML page of lecture
6, we could do something like the following:

``` md
View Week 6 Lecture [here](https://cdn.jsdelivr.net/gh/JSC370/jsc370-2023/slides/JSC370-slides-06.pdf)
```
