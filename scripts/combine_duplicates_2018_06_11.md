Reconciling duplicates
================

``` r
library(tidyverse)
```

This short file combines identified duplicate records from fuzzy matching by name and address. Duplicates were identified through a fuzzy matching algorithm in Python, which generated a list of potential matches that were reviewed manually to create a list of duplicates. First, we read in the list of duplicates and create a list of record pairs to be combined. There's two files, one matched based on similar names and the other based on similar addresses. There's some duplicate pairs in common between them. For later use, we combine these pairs to a single column.

``` r
name_dup<-read_csv("data/zane_results/snap_matches_from_names1.csv") %>%
  filter(dup==1) %>%
  select(store1name,store2name_1)

add_dup<-read_csv("data/zane_results/snap_mattches_from_addresses1.csv") %>%
  filter(dup==1) %>%
  select(store1name,store2name_1)

all_dup<-name_dup %>%
  bind_rows(add_dup) %>%
  distinct() %>%
  unite(pair,store1name,store2name_1,sep="--")

all_dup_list<-split(all_dup, seq(nrow(all_dup)))
```

Now we can read in a list of our SNAP retailers.

``` r
stores<-read_csv("data/old/snap_retailers_natl_2018_06_11.csv")
```

Next, we create a function that combines pairs. This function creates a two row dataset for each store pair, combines the year dummy variable column, and then adds that combined record back to the first listing of each pair.

``` r
pair_comb<-function(store_pair) {
  store1=substr(store_pair,1,9)
  store2=substr(store_pair,12,20)
  store1_data<-filter(stores,storeid==store1)
  store2_data<-filter(stores,storeid==store2)
  store_pair_data<-bind_rows(store1_data,store2_data) 
  store_pair_years<-store_pair_data %>%
    gather(Y2008:Y2017,key="year",value="value") %>%
    group_by(year) %>%
    summarise(value=if_else(sum(value)>0,1,0)) %>%
    spread(year,value)
  store_pair_data1<-store_pair_data %>%
    filter(storeid==store1) %>%
    select(-Y2008:-Y2017) %>%
    bind_cols(store_pair_years) %>%
    mutate(dup=1)
  store_pair_data1
}

all_dup_comb<-lapply(all_dup_list,pair_comb) %>%
  bind_rows()
```

Then we add the combined data back to the dataset, first removing any duplicates.

``` r
storedups<-name_dup %>%
  bind_rows(add_dup) %>%
  gather() %>%
  select(-key) %>%
  rename("storeid"=value) %>%
  distinct()

stores_nodup<-stores %>%
  anti_join(storedups) %>% 
  mutate(dup=0) %>%
  bind_rows(all_dup_comb)

write_csv(stores_nodup,"data/snap_retailers_natl_nodup_2018_06_11.csv")
```
