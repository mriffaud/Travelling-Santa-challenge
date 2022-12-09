# Travelling Santa challenge

This challenge has been created based on 2018 Kaggle Christmas challenge and is design to be ran using Azure Storage and Databricks

*Rudolph has always believed in working smarter, not harder. And what better way to earn the respect of Comet and Blitzen than showing the initiative to improve Santa's annual route for delivering toys on Christmas Eve?

*This year, Rudolph believes he can motivate the overworked Reindeer team by wisely choosing the order in which they visit the houses on Santa's list. The houses in prime cities always leave carrots for the Reindeers alongside the usual cookies and milk. These carrots are just the sustenance the Reindeers need to keep pace. In fact, Rudolph has found that if the Reindeer team doesn't originate from a prime city exactly every 10th step, it takes them 10% longer than it normally would to make their next destination!

*Can you help Rudolph solve the Traveling Santa problem subject to his carrot constraint? His team--and Santa--are counting on you!

<img src="https://mir-s3-cdn-cf.behance.net/project_modules/disp/c8385876438773.5c759e561c2a9.gif" alt="rudolph_compute" width="300"/>


## Prerequisites and Imports
The first step is to import the necessary libraries. These are commonly used ones so the experiment is reproducible.
```python: 
import numpy as np 
import pandas as pd 
from math import radians, cos, sin, asin, sqrt 
import geopandas 
import plotly.express as px 
```

## Loading the data
The dataset has been extracted from the open source website [simplemaps](https://simplemaps.com/data/world-cities), the top three cities of each country in the world were selected when they had over 200,000 inhabitants.

#### Data Dictionary
* city: name of the city 	
* lat: latitude of the city			
* lng: longitude of the city
* country: name of the city's country
* population: estimate of the city's population
* city_id: generated unique ID
* prime_city: cities for which the city_id is a prime number

```python: 
# calling the data from Azure Storage
data = spark.read.csv("YOUR_PATH")
# converting spark dataframe to python pandas dataframe
data = data.toPandas()
```
## Basic Information
This section will provide basic information about the data.

DATA INFO PICTURE HERE

The data frame is composed of 369 rows each representing a city and 7 features. The information above shows that we have no missing values which means we won't have to manipulate the data before running the model. However, the `prime_city` column is shown as an integer but we want to transform it into a binary column, so it is easier to use later on in the script.

Using `mapbox` available in `express` we can place each city from our list on a map to see where Santa is supposed to stop:
```python: 
# plot the cities in data
fig = px.scatter_mapbox(data, lat="lat", 
                        lon="lng", 
                        hover_name="city", 
                        hover_data=["country","prime_city"],
                        color="city", 
                        zoom=1, 
                        height=500)
fig.update_layout(mapbox_style="carto-positron")
fig.update_layout(margin={"r":0,"t":0,"l":0,"b":0})
fig.show()
```
MAP HERE

We map above shows how uneven the cities are spread across the globe. We can see dense areas in Europe, Africa, and South America and some more isolated locations in Australia and New Zealand as well as Canada and the United States of America.

## Calculating the Path Distance

In this challenge, we want to optimise the path Santa is going to take to visit all cities on his list. A way to assess the efficiency of a path is to calculate the total distance travelled by Santa and his reindeer. For that, we are going to define a function called `distance()`:
`distance()` takes two arguments:
* `df` a data frame with the city_id, city's coordinates and the prime_city
* `path` a calculated path stored as a list of city IDs

The first step is to define the `previous_stop`. Since we are starting our journey from Santa's home in Lapland every path should start there, this means that the first previous_stop will always be `path[0]`. We then, look at the prime cities and store them as a list oin `prime_cities`. We will be used it later on in the function to increase the distance when the 10th step is not a prime city since we know that it takes the reindeer team 10% longer than it normally would, to make their next destination. We also want to keep track of the `total_distance` starting at zero miles and the `stop_number` starting at stop 1.

We calculate the distance between each city available in the path from the `previous_stop` using the [haversine formula](https://en.wikipedia.org/wiki/Haversine_formula) formula which determines the great-circle distance between two points since Santa and his reindeer can fly using his sleigh and store it in `distance`. We add 10% extra on the mileage every 10th step when it not coming from a prime city and store the final result in `total_distance` until we visited all the cities on our list.

```python:
def distance(df,path):
  '''Calculates the distance in miles for a provided path. it takes two arguments:
      - df: a data frame with the city_id, city's coordinates and the prime_city
      - path: a calculated path stored as a list of city IDs
  '''
  previous_stop = path[0] 
  prime_cities = list(df.city_id[df.prime_city == 1]) 
  total_distance = 0
  stop_number = 1
  for city in path[1:]:
    next_stop = city 
    # to calculate the total distance of the path, we use haversine distance formula since Santa's can fly using his sleigh 
    # convert decimal degrees to radians 
    lng_previous_stop, lat_previous_stop, lng_next_city, lat_next_city = map(radians, [df.lng[previous_stop], 
                                                                                       df.lat[previous_stop], 
                                                                                       df.lng[next_stop], 
                                                                                       df.lat[next_stop]])
    # haversine formula
    distance_lng = lng_previous_stop - lng_next_city
    distance_lat = lat_previous_stop - lat_next_city

    a = sin(distance_lat/2)**2 + cos(lat_previous_stop) * cos(lat_next_city) * sin(distance_lng/2)**2
    c = 2 * asin(sqrt(a)) 
    r = 3956 # radius of earth in miles. Use 6371 for kilometers.
    distance = c * r
    # add 10% extra to the distance when the 10th step is not a prime city:
    if previous_stop in prime_cities:
      extra_distance = distance # if prime city is true then we keep going
    else:
      extra_distance = distance * (1+ 0.1*((stop_number % 10 == 0)))
    # add the final distance on the total path distance 
    total_distance = total_distance + extra_distance
    # amend the previous city 
    previous_stop = next_stop
    # add an extra stop the the stop number
    stop_number = stop_number + 1
  return total_distance
```
## Creating a dumb Path
In this section, we do not try to find the efficient path to take but follow the list order. This will give us a baseline to compare the performance of the different paths.

```python:
# calculates the dumb path using the city IDs
dumb_path = list(data.city_id[:].append(pd.Series([0])))
# calculating the distance travelled for this path:
print('Total distance with the dumb path is {} miles'.format(round(distance(data,dumb_path),0)))
Total distance with the dumb path is 1783555.0 miles
```
We then merge the path created with the data to be able to plot it on the world map:
```python:
# merge the path with the data
df_path = pd.DataFrame({'city_id':dumb_path}).merge(data, how = 'left', on=['city_id'])

# plot the path on a map
fig = px.line_mapbox(df_path, 
                     lat="lat", 
                     lon="lng", 
                     hover_name="city", 
                     hover_data=["country","prime_city"],
                     zoom=1, 
                     height=500)
fig.update_layout(mapbox_style="carto-positron")
fig.update_layout(margin={"t":0,"r":0,"b":0,"l":0}) # Dimensions of each margin. (To remember order, think trouble).
fig.show()
```
PICTURE DUMB PATH MAP

## Path ordered by coordinates
In this section, we are trying to improve the path by ordering the coordinates by latitudes and longitudes.









