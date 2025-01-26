# demo-trade-flow-r

In one of my projects, I was asked to create a map of trade flows from a certain country to destination countries for a given product and a given year. The problem is that I have never done this before, at all, let alone in R.

Thankfully, after going through the literature I found a paper by [Peterson (2021)](https://journals.sagepub.com/doi/abs/10.1177/00220027211014945) where he used a dataset to do exactly what I wanted.

This repo is meant to document what I did throughout multiple evenings of reading the R man pages, just to create a simple map plot for demo purposes.

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
Normally, I take data from a multi-gig dataset which is hard to open in excel (you either use a database software or load them to RStudio directly using e.g. `read.csv`) like Harvard's Atlas of Economic Complexity (https://atlas.hks.harvard.edu/). But for demonstration purposes, I created vectors to represent a small database of countries with the trade sources, destinations, and the coordinates of both sources and destination countries.

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
(I won't be using the `value` vector for this demo. It is there for completeness' sake).

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
I then add the trade flows for each country and destination. Note that, with vectors, the trade flows will originate and go to each data point according to the ordering of the vectors. As you will see later on, this means that USA will always link with Mexico, Japan with Australia, and so forth. Real datasets in a .csv file would usually do this for every row anyway, so this should not be an issue. 

For this, I used a function to generate geodesic curves using the `geosphere` package and, since it's a geodesic curve (the shortest path on a curve plane, like earth), I would need to create a path finding algorithm for a curve. Otherwise, the lines would all be straight, which does not make sense. Some reading for this [here](https://cran.r-project.org/web/packages/geosphere/vignettes/geosphere.pdf).

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
The end result is the following map plot:
![Map plot](https://github.com/hiu-sasongkojati/demo-trade-flow-r/blob/main/result%20map%20plot.png)

## How about having one country as the trade source, while all the other countries are made as trade destinations?
Some changes will have to be done to do that. Remember that using vectors to create a dataset means that you will need to have the same number of data points per vector. So even if you want only one country as the source, you will need to keep repeating it for every new destination country you entered. See below:
```
trade_data <- data.frame(
  source = c("Japan", "Japan", "Japan", "Japan"),
  dest = c("USA", "China", "Germany", "Australia"),
  value = c(100, 200, 150, 120),
  source_lat = c(36.2048, 36.2048, 36.2048, 36.2048),
  source_long = c(138.2529, 138.2529, 138.2529, 138.2529),
  dest_lat = c(37.0902, 35.8617, 51.1657, -25.2744),
  dest_long = c(-95.7129, 104.1954, 10.4515, 133.7751)
)
```
Then, since there is only one source country with multiple destination countries of varying distances, I need to take into account the international date line. 

Let's consider Japan and USA -- two countries at opposite ends of one another. When a geodesic line is calculated between these two points, the resulting path may lie on a single continuous line that crosses the international date line. I want to make sure that this line is plotted so that it appears on the map without being cut off by the international date line. Otherwise, the line would jump to the opposite side of the map to continue. Here is a simple visual representation of it:

`Without accounting for the date line:          Japan ----> USA`

`Accounting for the date line:                  Japan ---> Date line (split) ---> USA`

I managed this using the `gcIntermediate` function by splitting the line into multiple segments. 

```
generate_curve <- function(src_long, src_lat, dst_long, dst_lat) {

  # Calculate intermediate points on the geodesic curve
  gc_points <- gcIntermediate(c(src_long, src_lat), c(dst_long, dst_lat), n = 100, addStartEnd = TRUE, breakAtDateLine = TRUE)

  # Handle single segment output and multiple segments (split lines at date line)
  if (is.matrix(gc_points)) {
    data.frame(
      lon = gc_points[, 1],
      lat = gc_points[, 2],
      group = i
    )
  } else {
    do.call(
      rbind,
      lapply(gc_points, function(segment) {
        data.frame(
          lon = segment[, 1],
          lat = segment[, 2],
          group = i
        )
      })
    )
  }
}
```

The other code segments should be similar to the originals above. Just copy and paste them and display the final plot using `print(base_map)`, like so:
![Map plot with multiple lines](https://github.com/hiu-sasongkojati/demo-trade-flow-r/blob/main/result%20map%20plot%20-%20multiple%20lines.png)

## Multiple segment lines across the international date line
To have a trade flow line that does not produce a straight line in between segment breaks like the one above, you will need to assign these lines into groups.

Essentially, for every single segment for a trade flow line, it is to be grouped in `i`. Else, if it breaks the international date line (thus producing multiple segments), concatenate the group to `j`. As written below under the `generate_curve` function. [(but why `i` and `j`?)](https://www.reddit.com/r/programming/comments/egka6/why_are_variables_i_and_j_used_for_counters/)

```
...
else {
    # Multiple segments
    do.call(
      rbind,
      lapply(1:length(gc_points), function(j) {
        data.frame(
          lon = gc_points[[j]][, 1],
          lat = gc_points[[j]][, 2],
          group = paste0(i, "-", j)
        )
      }
...
```
Run this along with the rest of the segments above and you will get something like this:
![map with split segments](https://github.com/hiu-sasongkojati/demo-trade-flow-r/blob/main/result%20map%20plot%20-%20multiple%20lines%20(split%20at%20date%20line).png)
