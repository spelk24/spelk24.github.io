Beer Me Methodology
================

In the “Beer Me” analysis, several steps were performed before reaching
the end outputs on the “Beer Comparisons” tab. At a high level here was
the process:

1.  Scrape the data from MolsonCoors, which can be found here:
    [molsoncoors.com](https://www.molsoncoors.com/sites/molsonco/files/Combined%20Website%20Update%2006112020_0.pdf)
    - This data was *messy* - as it was free text in a PDF - The
    scraping process was not as automated as I had hoped for, but it was
    certainly faster than manual copying and pasting
2.  Cleaning the data
      - Some basic EDA and feature selection
      - Removed variables with zero or low variance
      - Combined ingredients that were similar
      - Created dummy-variables from the ingredients
      - The end result was a 222x186 data set
3.  Partitioning Around Medoids (PAM) cluster algorithm
      - Unsupervised algorithm similar to k-means
      - Added cluster labels to the data set
4.  Uniform Manifold Approximation and Projection (UMAP)
      - Dimensionality reduction technique
      - Very good at maintaining local and global structure of the data
5.  Nearest Neighbors
      - Extract nearest neighbors for each beer
      - These neighbors are the recommendations

Parts 1 and 2 are not important to the methodology used in the
application, so I will focus mainly parts 3,4, and 5.

# Libraries & Data

The following packages are needed.

``` r
library(tidyverse)
library(stringr)
library(ggtext)
library(tidyr)
library(plotly)
library(StatMatch)
library(cluster)
library(uwot)
library(kableExtra)
library(formattable)
library(webshot)
library(magick)
```

# Partitioning around Medoids

Partition around medics (PAM) is an algorithm that is intended to find
objects (medoids) that are centrally located in clusters. The goal of
the algorithm is to minimize the average dissimilarity of objects to
their closest selected object.

I used the PAM clustering algorithm on the full beer data set. Before I
ran the model, I had to create a distance matrix form the data. The
distance matrix has pairwise distances for each data point. I used the
[gower
distance](%22https://medium.com/@rumman1988/clustering-categorical-and-numerical-datatype-using-gower-distance-ab89b3aa90d9#:~:text=Gower%20Distance%20is%20a%20distance,of%20categorical%20and%20numerical%20values.%22),
because I had a mix of numerical and categorical variables.

``` r
# Gower Dist
cluster_data_input <- beer_full %>% select(!c("Brand","Brand_Style","Ingredients")) # Don't include raw text columns
cluster_dist_input <- dist(cluster_data_input, method = "gower")
```

For the PAM cluster, I chose k = 10 (clusters) based on the \[Silhouette
method\](<https://en.wikipedia.org/wiki/Silhouette_(clustering)>, which
is a measure of how similar objects are to their cluster label. Values
for k = 5:20 produced similar silhouette scores, but I chose 10 based on
some domain knowledge of the beer data
set.

``` r
pam_cluster <- pam(x = cluster_dist_input, k = 10, diss = TRUE, cluster.only = TRUE)
beer_data_with_clusters <- beer_full %>%
  mutate(Ingr_Cluster = as.factor(pam_cluster))
```

There isn’t a ton we can do with the cluster labels yet. Ideally, I want
to be able to visualize the beers on a scatter plot, and be able to
recommend 5 other beers, based on any selection. This is where UMAP
comes in.

# Uniform Manifold Approximation and Projection (UMAP)

UMAP is a dimensionality reduction technique that uses neighbor graphs
to project data down to a lower space. In simple terms, I want to go
from a 222x177 (222 beers, 177 features) matrix to a 222x2 matrix. If I
can project the data set into 2 dimensions, I can visualize all the
beers on a scatter plot.

One of the cool things about UMAP is that it tries to optimize for both
the local and global structures; something t-SNE (another dimensionality
reduction technique) does not do well. This is important because when we
visualize the UMAP output, there might be two clusters of beers close to
each other. The two clusters might have some similarities (high ABV,
high calories, fruity, etc.).

For more technical details on UMAP, check out this awesome talk from
PyData [Modern Approaches to Dimension
Reduction](https://www.youtube.com/watch?v=YPJQydzTLwQ).

I’m using the [uwot package](https://github.com/jlmelville/uwot)
implementation of UMAP. This is one of my favorite implementations of
UMAP because for a smaller data set (n \< 4096), the UMAP algorithm
allows you to extract the exact nearest neighbors. I only have 222 beers
in the data, so the nearest neighbors will be the “Beer Me”
recommendations.

The code below shows the UMAP implementations. I’ve included the cluster
labels as a target variable, and gave that variable a 50% weight. This
is not mandatory - I don’t even have to use a target variable, but the
cluster labels likely provide meaningful information in are otherwise
sparse data set.

``` r
## UMAP
umap_data_input <- beer_data_with_clusters %>% 
  select(!contains("Brand")) %>%
  select(!Ingredients)

umap_output <- uwot::umap(umap_data_input,
                          y = umap_data_input$Ingr_Cluster,
                          target_weight = .5, # 50% on the target var (cluster label)
                          metric = "euclidean",
                          n_neighbors = 6,
                          n_components = 2,
                          spread = .75,
                          min_dist = .01,
                          scale = TRUE,
                          ret_nn = TRUE) # retain the nearest neighbors
```

For more information of the hypyerparameters that you can tune in
uwot::umap(), check out this link [UMAP
hyperparameters](https://rdrr.io/cran/uwot/man/umap.html).

Now that I have the 2-D output from UMAP, I can plot this using ggplot2
and add the cluster label as the color. This is a pretty good visual
representation of the beer data set. Since I gave a 50% weight to the
target feature (the cluster label), objects in the same cluster tend to
be pretty close together, but not always. In the scatterplot, the
cluster label is the color.

``` r
# Plotting UMAP
umap_plotting_data <- cbind(beer_data_with_clusters,umap_output$embedding)
umap_plotting_data <- umap_plotting_data %>% rename(UMAP_X = `1`,UMAP_Y = `2`)
gg_umap_plot <- ggplot(data = umap_plotting_data,aes(x = UMAP_X,
                                                y = UMAP_Y,
                                                colour = Ingr_Cluster,
                                                text = paste0(Brand,"<br>",Brand_Style,"<br>","ABV: ",ABV))) +
  geom_point(colour = "#fe9929",
             alpha = .6) +
  labs(title = "UMAP 2-D Representation of Beer Data",
       x = "UMAP 1",
       y = "UMAP 2") +
  theme(plot.title = element_markdown(size = 14),
        plot.subtitle = element_markdown(size = 10),
        plot.caption = element_markdown(),
        legend.position = "bottom",
        panel.background = element_rect(fill = "white", colour = "white"),
        panel.grid.major = element_line(colour = "#f0f0f0",size = .1)) + 
  geom_jitter(width = 1.5, height = 1.5,alpha = .6,
              show.legend = FALSE) #used because some beers are *too* similar and overlap

gg_umap_plot
```

![](BeerMeMethodology_files/figure-gfm/unnamed-chunk-6-1.png)<!-- -->

# Nearest Neighbors & Recommendations

As I mentioned earlier, because I have a smaller data set (n = 222),
UMAP can extract the exact nearest neighbors. The code below creates a
“nearest neighbors” data set, which I’ll use to show beer
recommendations based on any given selection.

``` r
# Extract Nearest Neigbor Data
beer_nn_data <- cbind(beer_data %>% select("Brand","Brand_Style"),umap_output$nn)

beer_nn_data <- beer_nn_data %>%
  pivot_longer(starts_with("euclidean"),
               names_to = c(".value", "set"),
               names_pattern = "(euclidean\\.[a-z]*\\.)(.)")
  
names(beer_nn_data) <- c("Brand","Brand_Style","Neighbor_Rk","Neighbor_Idx","Neighbor_Dist")
```

I’ll show two examples. One go-to beer, and one beer that’s not so
basic. Here are the top recommendations if you enjoy Miller Lite, but
you’re looking to try a different domestic MolsonCoors product.

``` r
selection <- c("Miller Lite")
idxs <- beer_nn_data %>% filter(Brand %in% selection, Neighbor_Rk <= 5) %>% pull(Neighbor_Idx)
neighbor_points <- beer_full %>%
  filter(Brand != selection) %>%
  slice(idxs) %>%
  select(Brand, Brand_Style, ABV, Calories)
neighbor_points_miller <- neighbor_points %>%
  mutate(ABV = color_tile("white","#df8d03")(ABV),
         Calories = color_bar("#fae96f")(Calories))

kable(neighbor_points_miller, escape = F) %>% 
  kable_styling(full_width = FALSE, position = "left",
                bootstrap_options = c("hover")) %>% 
  row_spec(1:nrow(neighbor_points), color = "black") %>%
  column_spec(3:4,width = "1cm") %>% 
  add_header_above(c(" "= 2, "Nutrition" = 2))
```

<table class="table table-hover" style="width: auto !important; ">

<thead>

<tr>

<th style="border-bottom:hidden" colspan="2">

</th>

<th style="border-bottom:hidden; padding-bottom:0; padding-left:3px;padding-right:3px;text-align: center; " colspan="2">

<div style="border-bottom: 1px solid #ddd; padding-bottom: 5px; ">

Nutrition

</div>

</th>

</tr>

<tr>

<th style="text-align:left;">

Brand

</th>

<th style="text-align:left;">

Brand\_Style

</th>

<th style="text-align:left;">

ABV

</th>

<th style="text-align:left;">

Calories

</th>

</tr>

</thead>

<tbody>

<tr>

<td style="text-align:left;color: black !important;">

Miller64

</td>

<td style="text-align:left;color: black !important;">

American-Style Light
Lager

</td>

<td style="text-align:left;color: black !important;width: 1cm; ">

<span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffffff">2.8</span>

</td>

<td style="text-align:left;color: black !important;width: 1cm; ">

<span style="display: inline-block; direction: rtl; border-radius: 4px; padding-right: 2px; background-color: #fae96f; width: 44.76%">64</span>

</td>

</tr>

<tr>

<td style="text-align:left;color: black !important;">

Molson Canadian

</td>

<td style="text-align:left;color: black !important;">

North American-Style
Lager

</td>

<td style="text-align:left;color: black !important;width: 1cm; ">

<span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #df8d03">5.0</span>

</td>

<td style="text-align:left;color: black !important;width: 1cm; ">

<span style="display: inline-block; direction: rtl; border-radius: 4px; padding-right: 2px; background-color: #fae96f; width: 100.00%">143</span>

</td>

</tr>

<tr>

<td style="text-align:left;color: black !important;">

Keystone Light

</td>

<td style="text-align:left;color: black !important;">

American-Style Light
Lager

</td>

<td style="text-align:left;color: black !important;width: 1cm; ">

<span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ecbb6a">4.1</span>

</td>

<td style="text-align:left;color: black !important;width: 1cm; ">

<span style="display: inline-block; direction: rtl; border-radius: 4px; padding-right: 2px; background-color: #fae96f; width: 70.63%">101</span>

</td>

</tr>

<tr>

<td style="text-align:left;color: black !important;">

Coors Light

</td>

<td style="text-align:left;color: black !important;">

American-Style Light
Lager

</td>

<td style="text-align:left;color: black !important;width: 1cm; ">

<span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #eab65e">4.2</span>

</td>

<td style="text-align:left;color: black !important;width: 1cm; ">

<span style="display: inline-block; direction: rtl; border-radius: 4px; padding-right: 2px; background-color: #fae96f; width: 71.33%">102</span>

</td>

</tr>

<tr>

<td style="text-align:left;color: black !important;">

Keystone Premium

</td>

<td style="text-align:left;color: black !important;">

American-Style
Lager

</td>

<td style="text-align:left;color: black !important;width: 1cm; ">

<span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #e7ac47">4.4</span>

</td>

<td style="text-align:left;color: black !important;width: 1cm; ">

<span style="display: inline-block; direction: rtl; border-radius: 4px; padding-right: 2px; background-color: #fae96f; width: 75.52%">108</span>

</td>

</tr>

</tbody>

</table>

The Miller Lite comparisons shouldn’t come as a big surprise, but if
you’re looking for something lighter with a little extra ABV, Molson
Canadian might be your new lager\!

The next example will be for Blue Moon Honey Wheat. The recommendations
for this one aren’t quite as obvious.

``` r
selection <- c("Blue Moon Honey Wheat")
idxs <- beer_nn_data %>% filter(Brand %in% selection, Neighbor_Rk <= 5) %>% pull(Neighbor_Idx)
neighbor_points <- beer_full %>%
  filter(Brand != selection) %>%
  slice(idxs) %>%
  select(Brand, Brand_Style, ABV, Calories)
neighbor_points_BlueMoon <- neighbor_points %>%
  mutate(ABV = color_tile("white","#df8d03")(ABV),
         Calories = color_bar("#fae96f")(Calories))
kable(neighbor_points_BlueMoon, escape = F) %>% 
  kable_styling(full_width = FALSE, position = "left",
                bootstrap_options = c("hover")) %>% 
  row_spec(1:nrow(neighbor_points), color = "black") %>%
  column_spec(3:4,width = "1cm") %>% 
  add_header_above(c(" "= 2, "Nutrition" = 2))
```

<table class="table table-hover" style="width: auto !important; ">

<thead>

<tr>

<th style="border-bottom:hidden" colspan="2">

</th>

<th style="border-bottom:hidden; padding-bottom:0; padding-left:3px;padding-right:3px;text-align: center; " colspan="2">

<div style="border-bottom: 1px solid #ddd; padding-bottom: 5px; ">

Nutrition

</div>

</th>

</tr>

<tr>

<th style="text-align:left;">

Brand

</th>

<th style="text-align:left;">

Brand\_Style

</th>

<th style="text-align:left;">

ABV

</th>

<th style="text-align:left;">

Calories

</th>

</tr>

</thead>

<tbody>

<tr>

<td style="text-align:left;color: black !important;">

Blue Moon Iced Coffee Blonde

</td>

<td style="text-align:left;color: black !important;">

Blonde
Ale

</td>

<td style="text-align:left;color: black !important;width: 1cm; ">

<span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f1ce92">5.4</span>

</td>

<td style="text-align:left;color: black !important;width: 1cm; ">

<span style="display: inline-block; direction: rtl; border-radius: 4px; padding-right: 2px; background-color: #fae96f; width: 91.58%">185</span>

</td>

</tr>

<tr>

<td style="text-align:left;color: black !important;">

Leinenkugel’s Light

</td>

<td style="text-align:left;color: black !important;">

American-Style Light
Lager

</td>

<td style="text-align:left;color: black !important;width: 1cm; ">

<span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #ffffff">4.2</span>

</td>

<td style="text-align:left;color: black !important;width: 1cm; ">

<span style="display: inline-block; direction: rtl; border-radius: 4px; padding-right: 2px; background-color: #fae96f; width: 54.95%">111</span>

</td>

</tr>

<tr>

<td style="text-align:left;color: black !important;">

Leinenkugel’s Northwoods Lager

</td>

<td style="text-align:left;color: black !important;">

American
Lager

</td>

<td style="text-align:left;color: black !important;width: 1cm; ">

<span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f7e2c0">4.9</span>

</td>

<td style="text-align:left;color: black !important;width: 1cm; ">

<span style="display: inline-block; direction: rtl; border-radius: 4px; padding-right: 2px; background-color: #fae96f; width: 80.20%">162</span>

</td>

</tr>

<tr>

<td style="text-align:left;color: black !important;">

Terrapin Frenchy’s Blues

</td>

<td style="text-align:left;color: black !important;">

Berliner
Weisse

</td>

<td style="text-align:left;color: black !important;width: 1cm; ">

<span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #f3d6a5">5.2</span>

</td>

<td style="text-align:left;color: black !important;width: 1cm; ">

<span style="display: inline-block; direction: rtl; border-radius: 4px; padding-right: 2px; background-color: #fae96f; width: 75.74%">153</span>

</td>

</tr>

<tr>

<td style="text-align:left;color: black !important;">

Saint Archer Citra 7 IPA

</td>

<td style="text-align:left;color: black !important;">

American-Style
IPA

</td>

<td style="text-align:left;color: black !important;width: 1cm; ">

<span style="display: block; padding: 0 4px; border-radius: 4px; background-color: #df8d03">7.0</span>

</td>

<td style="text-align:left;color: black !important;width: 1cm; ">

<span style="display: inline-block; direction: rtl; border-radius: 4px; padding-right: 2px; background-color: #fae96f; width: 100.00%">202</span>

</td>

</tr>

</tbody>

</table>

I’ve personally had none of these beers, so I have a few to try out;
especially that Blue Moon Iced Coffee Blonde.

# Conclusion

I hope this mini-tutorial was a helpful introduction on how to use
clustering and UMAP to build a recommender system. I’d love to expand
this to all breweries & beers that have publicly available nutrition
information, but that would take quite a bit of research on finding and
combining the data. If you are reading this and you know of a large,
publicly available “beer-database”, please send the information my way
by emailing me at <spelkofer24@yahoo.com>.

Cheers\!

@Spelk24
