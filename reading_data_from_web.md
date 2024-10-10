Reading Data from the Web
================
Maya Krishnamoorthy
2024-10-10

Load the necessary libraries.

``` r
library(rvest)
```

    ## 
    ## Attaching package: 'rvest'

    ## The following object is masked from 'package:readr':
    ## 
    ##     guess_encoding

``` r
library(httr)
```

## Extracting tables

``` r
url = "http://samhda.s3-us-gov-west-1.amazonaws.com/s3fs-public/field-uploads/2k15StateFiles/NSDUHsaeShortTermCHG2015.htm"
drug_use_html = read_html(url)

drug_use_html
```

    ## {html_document}
    ## <html lang="en">
    ## [1] <head>\n<link rel="P3Pv1" href="http://www.samhsa.gov/w3c/p3p.xml">\n<tit ...
    ## [2] <body>\r\n\r\n<noscript>\r\n<p>Your browser's Javascript is off. Hyperlin ...

Get the pieces I actually need.

``` r
marj_use_df = 
  drug_use_html |> 
  html_table() |> # from rvest package, pulls out all tables on the page
  first() |> # takes first element out - for us it outputs the first table
  slice(-1)

marj_use_df
```

    ## # A tibble: 56 × 16
    ##    State     `12+(2013-2014)` `12+(2014-2015)` `12+(P Value)` `12-17(2013-2014)`
    ##    <chr>     <chr>            <chr>            <chr>          <chr>             
    ##  1 Total U.… 12.90a           13.36            0.002          13.28b            
    ##  2 Northeast 13.88a           14.66            0.005          13.98             
    ##  3 Midwest   12.40b           12.76            0.082          12.45             
    ##  4 South     11.24a           11.64            0.029          12.02             
    ##  5 West      15.27            15.62            0.262          15.53a            
    ##  6 Alabama   9.98             9.60             0.426          9.90              
    ##  7 Alaska    19.60a           21.92            0.010          17.30             
    ##  8 Arizona   13.69            13.12            0.364          15.12             
    ##  9 Arkansas  11.37            11.59            0.678          12.79             
    ## 10 Californ… 14.49            15.25            0.103          15.03             
    ## # ℹ 46 more rows
    ## # ℹ 11 more variables: `12-17(2014-2015)` <chr>, `12-17(P Value)` <chr>,
    ## #   `18-25(2013-2014)` <chr>, `18-25(2014-2015)` <chr>, `18-25(P Value)` <chr>,
    ## #   `26+(2013-2014)` <chr>, `26+(2014-2015)` <chr>, `26+(P Value)` <chr>,
    ## #   `18+(2013-2014)` <chr>, `18+(2014-2015)` <chr>, `18+(P Value)` <chr>

## Learning assessment

Read in cost of living data.

``` r
nyc_cost_df =
  read_html("https://www.bestplaces.net/cost_of_living/city/new_york/new_york") |> 
  html_table(header = TRUE) |> # tell html that the first row is the header
  first()

nyc_cost_df
```

    ## # A tibble: 8 × 4
    ##   Categories       `New York` `New York` `United States`
    ##   <chr>            <chr>      <chr>      <chr>          
    ## 1 Overall          172.5      121.5      100.0          
    ## 2 Grocery          116.6      103.8      100            
    ## 3 Health           127.6      120.7      100.0          
    ## 4 Housing          224.3      127.9      100.0          
    ## 5 Median Home Cost $677,200   $413,600   $338,100       
    ## 6 Utilities        150.5%     115.9%     100.0%         
    ## 7 Transportation   181.1      140.7      100.0          
    ## 8 Miscellaneous    136.4      121.8      100.0

## CSS Selectors

``` r
swm_url = "https://www.imdb.com/list/ls070150896/"

swm_html = read_html(swm_url)

swm_title_vec = 
  swm_html |> 
  html_elements(".ipc-title-link-wrapper .ipc-title__text") |> 
  html_text()

# run time
swm_runtime_vec = 
  swm_html |> 
  html_elements(".dli-title-metadata-item:nth_child(2)") |> 
  html_text()

swm_score_vec = 
  swm_html |> 
  html_elements(".metacritic-score-box") |> 
  html_text()

swm_df = 
  tibble(
    title = swm_title_vec,
    runtime = swm_runtime_vec,
    score = swm_score_vec
  )
```

## Learning assessment 2

Books: Use a process similar to the one above to extract the book
titles, stars, and prices

``` r
books_url = "https://books.toscrape.com/"
books_html = read_html(books_url)

books_titles = 
  books_html |> 
  html_elements("h3") |> 
  html_text2()

books_stars = 
  books_html |>
  html_elements(".star-rating") |>
  html_attr("class")

books_price = 
  books_html |>
  html_elements(".price_color") |>
  html_text()

books = tibble(
  title = books_titles,
  stars = books_stars,
  price = books_price
)
```

## APIs

``` r
# Import as CSV
nyc_water = 
  GET("https://data.cityofnewyork.us/resource/ia2d-e54m.csv") |> 
  content("parsed")
```

    ## Rows: 45 Columns: 4
    ## ── Column specification ────────────────────────────────────────────────────────
    ## Delimiter: ","
    ## dbl (4): year, new_york_city_population, nyc_consumption_million_gallons_per...
    ## 
    ## ℹ Use `spec()` to retrieve the full column specification for this data.
    ## ℹ Specify the column types or set `show_col_types = FALSE` to quiet this message.

``` r
# Import as JSON
nyc_water = 
  GET("https://data.cityofnewyork.us/resource/ia2d-e54m.json") |> 
  content("text") |>
  jsonlite::fromJSON() |>
  as_tibble()
```

By default, the CDC API limits data to the first 1000 rows. Here I’ve
increased that by changing an element of the API query – I looked around
the website describing the API to find the name of the argument, and
then used the appropriate syntax for GET. To get the full data, I could
increase this so that I get all the data at once or I could try
iterating over chunks of a few thousand rows.

``` r
brfss_smart2010 = 
  GET("https://chronicdata.cdc.gov/resource/acme-vg9e.csv",
      query = list("$limit" = 5000)) |> 
  content("parsed")
```

    ## Rows: 5000 Columns: 23
    ## ── Column specification ────────────────────────────────────────────────────────
    ## Delimiter: ","
    ## chr (16): locationabbr, locationdesc, class, topic, question, response, data...
    ## dbl  (6): year, sample_size, data_value, confidence_limit_low, confidence_li...
    ## lgl  (1): locationid
    ## 
    ## ℹ Use `spec()` to retrieve the full column specification for this data.
    ## ℹ Specify the column types or set `show_col_types = FALSE` to quiet this message.

Pokemon API Example

``` r
poke = 
  GET("http://pokeapi.co/api/v2/pokemon/1") |>
  content()

poke[["name"]]
```

    ## [1] "bulbasaur"

``` r
poke[["height"]]
```

    ## [1] 7

``` r
poke[["abilities"]]
```

    ## [[1]]
    ## [[1]]$ability
    ## [[1]]$ability$name
    ## [1] "overgrow"
    ## 
    ## [[1]]$ability$url
    ## [1] "https://pokeapi.co/api/v2/ability/65/"
    ## 
    ## 
    ## [[1]]$is_hidden
    ## [1] FALSE
    ## 
    ## [[1]]$slot
    ## [1] 1
    ## 
    ## 
    ## [[2]]
    ## [[2]]$ability
    ## [[2]]$ability$name
    ## [1] "chlorophyll"
    ## 
    ## [[2]]$ability$url
    ## [1] "https://pokeapi.co/api/v2/ability/34/"
    ## 
    ## 
    ## [[2]]$is_hidden
    ## [1] TRUE
    ## 
    ## [[2]]$slot
    ## [1] 3
