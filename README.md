# demo-trade-flow-r

In one of my projects, I was asked to create a map of trade flows from a certain country to destination countries for a given product and a given year. The problem is that I have never done this before, at all, let alone in R.

Thankfully, after going through the literature I found a paper by [Peterson (2021)](https://journals.sagepub.com/doi/abs/10.1177/00220027211014945) where he used a dataset to do exactly what I wanted.

This repo is meant to document what I did throughout one evening of reading the R man pages, just to create a simple map plot for demo purposes.

## Installing the required packages
I needed to install some packages and call them as libraries first.

```
install.packages(c("ggplot2", "sf", "dplyr", "maps", "geosphere"))
```

Calling the respective libraries into the current R instance:
```
library(ggplot2)
library(sf)
library(dplyr)
library(maps)
library(geosphere)
```

## Preparing the data
Normally, I take data from a multi-gig dataset (which is hard to open in excel--you need a database software for this) like Harvard's Atlas of Economic Complexity (https://atlas.hks.harvard.edu/). But for demonstration purposes, I created vectors to represent a small database of countries with the trade sources, destinations, and the coordinates of both sources and destination countries.

```
trade_data <- data.frame(
  source = c("USA", "China", "Germany", "Japan"),
  dest = c("Mexico", "South Korea", "France", "Australia"),
  value = c(100, 200, 150, 120),
  source_lat = c(37.0902, 35.8617, 51.1657, 36.2048),
  source_long = c(-95.7129, 104.1954, 10.4515, 138.2529),
  dest_lat = c(23.6345, 35.9078, 46.6034, -25.2744),
  dest_long = c(-102.5528, 127.7669, 2.2137, 133.7751)
)
```

## Create the map plot base
Next, before I do anything else, I asked R to create a world map by using the `geom_map()` function of the `ggplot2` package.

```
# Load world map
world <- map_data('world')

# Base map plot
base_map <- ggplot() +
  geom_polygon(data = world, aes(x = long, y = lat, group = group), fill = "lightgray", color = "white")
```

## Adding trade flows and displaying the final plot
I then add the trade flows for each country and destination. Note that, with vectors, the trade flows will originate and go to each data point according to the ordering of the vectors. As you will see later on, this means that USA will always link with Mexico, Japan with Australia, and so forth. Real datasets in a .csv file would usually do this for every row anyway, so this should not be an issue. For this, I used a function to generate geodesic curves using the `geosphere` package.

```
generate_curve <- function(src_long, src_lat, dst_long, dst_lat) {
  # Calculate intermediate points on the geodesic curve
  gc_points <- gcIntermediate(c(src_long, src_lat), c(dst_long, dst_lat), n = 100, addStartEnd = TRUE, breakAtDateLine = TRUE)
  data.frame(
    lon = gc_points[, 1],
    lat = gc_points[, 2],
    group = i
  )
}
```
Let's add the trade flow lines for all of the points. Remember that the ordering within each vector is important in this case.

```
for(i in 1:nrow(trade_data)) {
  curve_data <- generate_curve(trade_data$source_long[i], trade_data$source_lat[i], trade_data$dest_long[i], trade_data$dest_lat[i])
  base_map <- base_map +
    geom_path(data = curve_data, aes(x = lon, y = lat, group = group), color = "green", size = 0.5, alpha = 0.6)
}
```
Next, add the source and destination points:

```
base_map <- base_map +
  geom_point(data = trade_data, aes(x = source_long, y = source_lat), color = "blue", size = 2) +
  geom_point(data = trade_data, aes(x = dest_long, y = dest_lat), color = "black", size = 2)
```

Finally, display the final plot.
```
print(base_map)
```
