# tdm_metadata

``` r
library(bigrquery)
library(dplyr)
library(readr)
library(DBI)
```

## Connect to database

``` r
library(DBI)
library(bigrquery)
con <- dbConnect(
  bigrquery::bigquery(),
  project = "api-project-764811344545",
  dataset = "cr_dump_march_20"
)
bq_auth()
```

## Get TDM metadata

Journal article volume by publisher since 2013 by

- Intended use of full-text link. `text-mining` highlights that a
  full-text link can be used for text and data mining through the
  publisher
- MIME Type of full-text provided by the publisher
- Open license. Note only the availability of a CC license was checked

``` sql
WITH
  raw AS (
  SELECT
    link,
    publisher,
    license,
    EXTRACT ( YEAR
    FROM
      issued ) AS cr_year,
    doi,
    fulltext_links,
    CASE
      WHEN REGEXP_CONTAINS(LOWER(license.url), "creativecommons.org") THEN "CC"
    ELSE
    NULL
  END
    AS open_license
  FROM
    `subugoe-collaborative.cr_instant.snapshot`,
    UNNEST(link) AS fulltext_links,
    UNNEST(license) AS license
  WHERE
    type = "journal-article" )
SELECT
  COUNT(DISTINCT doi) AS n_articles, -- Number of articles
  publisher, -- Publisher name, be aware of imprints
  fulltext_links. intended_application, -- Intended use of full-text link
  fulltext_links.content_type, -- MIME Type, 
  open_license -- Is there a CC license?
FROM
  raw
WHERE
  fulltext_links.intended_application IN ("text-mining",
    "unspecified")
GROUP BY
  publisher,
  fulltext_links.content_type,
  fulltext_links.intended_application,
  open_license
ORDER BY
  n_articles DESC
```

Glimpse and back up as csv

``` r
tdm_data
#> # A tibble: 13,779 × 5
#>    n_articles publisher           intended_application content_type open_license
#>         <int> <chr>               <chr>                <chr>        <chr>       
#>  1    7504626 Elsevier BV         text-mining          text/plain   <NA>        
#>  2    7504626 Elsevier BV         text-mining          text/xml     <NA>        
#>  3    2768473 Springer Science a… text-mining          application… <NA>        
#>  4    2768201 Springer Science a… text-mining          text/html    <NA>        
#>  5    1559266 Wiley               text-mining          application… <NA>        
#>  6    1180186 Elsevier BV         text-mining          text/plain   CC          
#>  7    1180186 Elsevier BV         text-mining          text/xml     CC          
#>  8    1033789 Springer Science a… text-mining          application… CC          
#>  9    1033760 Springer Science a… text-mining          text/html    CC          
#> 10     851299 Wiley               text-mining          application… <NA>        
#> # ℹ 13,769 more rows

write_csv(tdm_data, "tdm_md_crossref_journal_articles_since_2013.csv")
```
