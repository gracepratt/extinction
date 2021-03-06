Species Extinctions
================
Grace Pratt, Sarah Tang

## Extinctions Module

*Are we experiencing the sixth great extinction?*

What is the current pace of extinction? Is it accelerating? How does it
compare to background extinction rates?

## Computational Topics

In order to assess the current rate of extinction, we:

  - Access data from a RESTful API
  - Handle errors
  - Use the JSON data format
  - Implement regular expressions
  - Work with missing values

## CURL and REST

We use the `httr` package to make a single API query for the species
Acaena exigua.

``` r
genus <- "Acaena"
species <- "exigua"
api_call <- paste0("http://api.iucnredlist.org/index/species/", genus, "-", species, ".json")
resp <- GET(api_call)
resp
```

    Response [http://api.iucnredlist.org/index/species/Acaena-exigua.json]
      Date: 2020-01-10 18:41
      Status: 200
      Content-Type: application/json; charset=utf-8
      Size: 877 B

*By examining the response and the content of the response, we can
determine that the call was successful because the status is 200. The
return type of the object is a json object. Later, we parse the returned
object into test and then present the data as a data frame.*

# Calculating Extinction Rates: Putting it all together

First, we download a data file containing a list of extinct species
names to know how to make queries to the IUCN REST
API.

``` r
extinct = read_csv("https://espm-157.github.io/extinction-module/extinct.csv")
```

Now, we write a function to extract the rationale for the extinction for
all extinct species in the data set (see above file).

``` r
api_fn <- function(genus, species) {
  api_call <- paste0("http://api.iucnredlist.org/index/species/", genus, "-", species, ".json")
  Sys.sleep(0.1)
  resp <- GET(api_call)
  status <- httr::status_code(resp)

  if (status > 300) {
    return(data_frame(Genus = genus, Species = species, rationale = NA, status_code = status))
  } 
  
  out <- content(resp, as = "text")
  df <- jsonlite::fromJSON(out)
  rationale <- df$rationale
  
  data_frame(Genus = genus, Species = species, API = api_call, rationale = rationale, status_code = status)
}

df <- map2_dfr(extinct$Genus, extinct$Species, api_fn)
```

Now we create a function that can extract the date from the rationale.
To do this we make some assumptions. We first assume that any 4 digit
number in the rationale is the year the species went extinct. We also
assume that any number between 21 - 1000 is the “time since” the species
went extinct and that any 2 digit number bellow 21 is the century that
the the species went extinct. For centuries, all dates are the first
year of that century (e.g. 20th century became
1900).

``` r
time_since <- function(x) { #assuming any number between 21 - 1000 is the "time since" the species went extinct
  if_else((x > 21 & x < 1000), (2018 - x), x)
}

century <- function(x) { #assuming any 2 digit number bellow 21 is the century that the the species went extinct
  if_else((x <= 21), ((x - 1) * 100), x)
}

txt <- df$rationale
dates <- as.numeric(stringr::str_extract(txt, "(\\d{2,})"))
df_years <- df %>% mutate(extracted_dates = dates) %>%
  mutate(Year = time_since(century(extracted_dates))) 
df_years
```

    # A tibble: 841 x 7
       Genus  Species  API      rationale    status_code extracted_dates  Year
       <chr>  <chr>    <chr>    <chr>              <int>           <dbl> <dbl>
     1 Acaena exigua   http://… The last kn…         200            1990  1990
     2 Acaly… dikuluw… http://… "<span styl…         200            1959  1959
     3 Acaly… rubrine… http://… <NA>                 200              NA    NA
     4 Acaly… wilderi  http://… "<span styl…         200            1929  1929
     5 Acant… pecaton… http://… <NA>                 200              NA    NA
     6 Achat… abbrevi… http://… <NA>                 200              NA    NA
     7 Achat… buddii   http://… <NA>                 200              NA    NA
     8 Achat… caesia   http://… <NA>                 200              NA    NA
     9 Achat… casta    http://… <NA>                 200              NA    NA
    10 Achat… decora   http://… <NA>                 200              NA    NA
    # ... with 831 more rows

## Histogram of Extinction Dates

We can get a sense for the tempo of extinctions by plotting extinctions
since 1500 in 25-year interval bins.

``` r
df_extinct <- df_years %>% filter(!is.na(Year)) %>%
  filter(Year > 1500)

ggplot(df_extinct, aes(x = Year)) + 
  geom_histogram(breaks = seq(1500, 2025, by = 25)) + 
  labs(title = "Extinctions since 1500 per 25 years", y = "Frequency")
```

![](extinction-assignment_files/figure-gfm/unnamed-chunk-6-1.png)<!-- -->

# Extinctions by group

In order to see how extinctions vary across different species, we
compute the number of extinctions from 1500 - 1900 and from 1900 to
present of each of the following taxonomic groups:

  - Vertebrates
  - Mammals
  - Birds
  - Fish
  - Amphibians
  - Reptiles
  - Insects
  - Plants

First we create dataframes for each taxanomic group.

``` r
mammals <- extinct %>%
  filter(Class == "MAMMALIA") %>%
  left_join(df_years) %>%
  select(Genus, Species, rationale, Year) %>%
  na.omit()

birds <- extinct %>%
  filter(Class == "AVES") %>%
  left_join(df_years) %>%
  select(Genus, Species, rationale, Year) %>%
  na.omit()

fish <- extinct %>%
  filter(Class == "ACTINOPTERYGII" 
         | Class == "CEPHALASPIDOMORPHI"
         | Class == "MYXINI"
         | Class == "ELASMOBRANCHII"
         | Class == "HOLOCEPHALI"
         | Class == "SACROPTERYGII") %>%
  left_join(df_years) %>%
  select(Genus, Species, rationale, Year) %>%
  na.omit()

amphibians <- extinct %>%
  filter(Class == "AMPHIBIA") %>%
  left_join(df_years) %>%
  select(Genus, Species, rationale, Year) %>%
  na.omit()
  
reptiles <- extinct %>%
  filter(Class == "REPTILIA") %>%
  left_join(df_years) %>%
  select(Genus, Species, rationale, Year) %>%
  na.omit()

insects <- extinct %>%
  filter(Class == "INSECTA") %>%
  left_join(df_years) %>%
  select(Genus, Species, rationale, Year) %>%
  na.omit()

plants <- extinct %>%
  filter(Kingdom == "PLANTAE") %>%
  left_join(df_years) %>%
  select(Genus, Species, rationale, Year) %>%
  na.omit()

vertebrates <- mammals %>%
  rbind(birds, amphibians, reptiles, fish) %>%
  left_join(df_years) %>%
  select(Genus, Species, rationale, Year) %>%
  na.omit()
```

Now we filter to get the number of species for each group that went
extict between 1500-1900 and post-1900.

``` r
between_1500_1900 <- function(df){
  df %>%
    filter(Year >= 1500, Year < 1900) %>%
    nrow()
}

post_1900 <- function(df){
  df %>% 
    filter(Year >= 1900) %>%
    nrow()
}

taxonomy_vector <- c("mammals", "birds", "fish", "amphibians", "reptiles", "insects", "plants", "vertebrates")

our_extinctions <- data_frame("Taxonomic Group" = taxonomy_vector, 
           "Extinctions 1500-1900" = 
             c(between_1500_1900(mammals),between_1500_1900(birds), between_1500_1900(fish), between_1500_1900(amphibians),between_1500_1900(reptiles), between_1500_1900(vertebrates), between_1500_1900(plants), between_1500_1900(vertebrates)),
           "Extinctions post-1900" = 
             c(post_1900(mammals), post_1900(birds), post_1900(fish),post_1900(amphibians), post_1900(reptiles), post_1900(insects), post_1900(plants), post_1900(vertebrates)))

our_extinctions
```

    # A tibble: 8 x 3
      `Taxonomic Group` `Extinctions 1500-1900` `Extinctions post-1900`
      <chr>                               <int>                   <int>
    1 mammals                                25                      30
    2 birds                                  96                      44
    3 fish                                    4                      36
    4 amphibians                              1                      30
    5 reptiles                               11                       5
    6 insects                               137                      10
    7 plants                                 12                      31
    8 vertebrates                           137                     145

In order to see how accurate our estimates are, we compare our estimates
to Table 1 of [Ceballos et al
(2015)](http://doi.org/10.1126/sciadv.1400253).

``` r
table1 <- data_frame("Taxonomic Group" = c("mammals", "birds", "fish", "amphibians", "reptiles", "vertebrates"), 
           "Extinctions 1500-1900" = c(42, 83, 0, 2, 13, 140),
           "Extinctions post-1900" = c(35, 57, 66, 32, 8, 198))

comparison <- our_extinctions %>%
  filter(`Taxonomic Group` %in% table1$`Taxonomic Group`) %>%
  mutate("Difference 1500-1900" = `Extinctions 1500-1900` - table1$`Extinctions 1500-1900`)%>%
  mutate("Difference post-1900" = `Extinctions post-1900` - table1$`Extinctions post-1900`)
  
comparison
```

    # A tibble: 6 x 5
      `Taxonomic Grou… `Extinctions 15… `Extinctions po… `Difference 150…
      <chr>                       <int>            <int>            <dbl>
    1 mammals                        25               30              -17
    2 birds                          96               44               13
    3 fish                            4               36                4
    4 amphibians                      1               30               -1
    5 reptiles                       11                5               -2
    6 vertebrates                   137              145               -3
    # ... with 1 more variable: `Difference post-1900` <dbl>

*Overall, the numbers of extinct species we estimated was less than the
numbers of extinctions estimated by Ceballos et al. The only exception
to this was the number of fish that went extinct between 1500 and 1900,
where we estimated 4 more extinct species than Ceballos et al. This
could be due to our taxanomic defintiion of fish, which may be different
from the one used by Ceballos et al.*

## Weighing by number of species

The number of species going extinct per century in a given taxonomic
group will be influenced by how many species are present in the group to
begin with. Overall, these numbers do not change greatly over a period
of a few hundred years, so we were able to make the relative comparisons
between the roughly pre-industrial and post-industrial periods above.

As discussed by Tony Barnosky in the introductory video (or in [Ceballos
et al (2015)](http://doi.org/10.1126/sciadv.1400253) paper), if we want
to compare these extinction rates against the long-term palentological
record, it is necessary to weigh the rates by the total number of
species. That is, to compute the number of extinctions per million
species per year (MSY; equivalently, the number extinctions per 10,000
species per 100 years).

1)  First, we will compute how many species are present in each of the
    taxonomic groups. To do so, we need a table that has not only
    extinct species, but all assessed species. We will once again query
    this information from the IUCN
API.

<!-- end list -->

``` r
api_calls <- map((0:10), function(page_num){paste0("http://apiv3.iucnredlist.org/api/v3/species/page/", page_num, "?token=9bb4facb6d23f48efbf424bb05c0c1ef1cf6f468393bc745d42179ac4aca5fee")})

resp2 <- map(api_calls[1:10], function(url){
  Sys.sleep(0.5)
  GET(url)
})

out <- map(resp2, content, as = "text")
df_page <- map(out, fromJSON)

all_df <- plyr::ldply(df_page, data.frame)
```

2)  Based on the complete data, we count the number of species in each
    group. Then we use these numbers to compute MSY, the number
    extinctions per 10,000 species per 100 years, for each of the
    taxanomic groups.

<!-- end list -->

``` r
allmammals <- all_df %>%
  filter(result.class_name == "MAMMALIA") %>%
  nrow()

allbirds <- all_df %>%
  filter(result.class_name == "AVES") %>%
  nrow()

allfish <- all_df %>%
  filter(result.class_name == "ACTINOPTERYGII" 
         | result.class_name == "CEPHALASPIDOMORPHI"
         | result.class_name == "MYXINI"
         | result.class_name == "ELASMOBRANCHII"
         | result.class_name == "HOLOCEPHALI"
         | result.class_name == "SACROPTERYGII") %>%
  nrow()

allamphibians <- all_df %>%
  filter(result.class_name == "AMPHIBIA") %>%
  nrow()

allreptiles <- all_df %>%
  filter(result.class_name == "REPTILIA") %>%
  nrow()

allinsects <- all_df %>%
  filter(result.class_name == "INSECTA") %>%
  nrow()

allplants <- all_df %>%
  filter(result.kingdom_name == "PLANTAE") %>%
  nrow()

allvertebrates <- allmammals %>%
  sum(allbirds, allamphibians, allreptiles, allfish)

msy_vector <- c(nrow(mammals)/(allmammals/10000)/100,
  nrow(birds)/(allbirds/10000)/100,
  nrow(fish)/(allfish/10000)/100,
  nrow(amphibians)/(allamphibians/10000)/100,
  nrow(reptiles)/(allreptiles/10000)/100,
  nrow(insects)/(allinsects/10000)/100,
  nrow(plants)/(allplants/10000)/100,
  nrow(vertebrates)/(allvertebrates/10000)/100)

msy_df <- data_frame("taxanomic_group" = taxonomy_vector, "MSY" = msy_vector)
msy_df
```

    # A tibble: 8 x 2
      taxanomic_group   MSY
      <chr>           <dbl>
    1 mammals         0.911
    2 birds           1.26 
    3 fish            0.239
    4 amphibians      0.462
    5 reptiles        0.215
    6 insects         0.136
    7 plants          0.151
    8 vertebrates     0.588

*Overall, we can see that our estimates for every group are less than
the overall historical average of about 2 MSY. This is likely due to our
limited algorithm that could not extract dates for all of the extinct
species. Therefore, there are likely many species that went extinct
after 1500 that we aren’t including in our analysis. This makes our MSY
artificially low. Looking at the MSYs calculated by Ceballos et al, we
can see that our MSY estimates should be increased by 4-13 points
depending on the taxanomic group.*

## Improving our algorithm

*In parsing the data with regular expressions and focusing only on four
digit values, we encountered certain data that resulted in missing
values. Our data extraction failed because we were only extracting years
and not accounting for instances when there were 2 or 3 digits. Some
main cases that we identified were in rationale when the date depicted
was a century (ex: 16th century) or when the corresponding number
referred to the number of years ago that a species went extinct (ex: 70
years ago).*

*Ultimately, we went back through our algorithm and modified how we
handled regular expressions to reduce the number of missing values. We
considered the two cases mentioned above and changed our algorithm to
include regular expressions to grab all 2, 3, and 4 digit numbers. With
two digit numbers less than or equal to 21, we subtracted 1 and then
mulitiplied by 100 to get the correct century, a ballpark estimate. For
example, “16th century” would give us 1500. For numbers greater than 21
and less than 1000, as they clearly can’t describe a century, we assumed
them to be x years ago. We outputted 2018 - x for such numbers.*
