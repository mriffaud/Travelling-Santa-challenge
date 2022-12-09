# Travelling Santa challenge

This challenge has been created based on 2018 Kaggle Christmas challenge.

*Rudolph has always believed in working smarter, not harder. And what better way to earn the respect of Comet and Blitzen than showing the initiative to improve Santa's annual route for delivering toys on Christmas Eve?*

*This year, Rudolph believes he can motivate the overworked Reindeer team by wisely choosing the order in which they visit the houses on Santa's list. The houses in prime cities always leave carrots for the Reindeers alongside the usual cookies and milk. These carrots are just the sustenance the Reindeers need to keep pace. In fact, Rudolph has found that if the Reindeer team doesn't originate from a prime city exactly every 10th step, it takes them 10% longer than it normally would to make their next destination!*

*Can you help Rudolph solve the Traveling Santa problem subject to his carrot constraint?*

*His team--and Santa--are counting on you!*

<img src="https://mir-s3-cdn-cf.behance.net/project_modules/disp/c8385876438773.5c759e561c2a9.gif" alt="rudolph_compute" width="300"/>

---
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
# read the data
data = pd.read_csv('../data/worldcities.csv')
```

## Basic Information
This section will provide basic information about the data.

![data_info](https://github.com/mriffaud/Travelling-Santa-challenge/blob/main/image/data_info.PNG)

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
![map_cities](https://github.com/mriffaud/Travelling-Santa-challenge/blob/main/image/map_cities.png)

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
![map_dumb_path](https://github.com/mriffaud/Travelling-Santa-challenge/blob/main/image/map_dumb_path.png)

## Path ordered by coordinates
In this section, we are trying to improve the path by ordering the coordinates by latitudes and longitudes.

First, we order the cities by coordinates starting with the latitude, excluding the first city since Santa lives in Rovaniemi to add it later at each ends of the list to start and finish at the same location. We can then run our `distance()` function to see if this method improve the mileage.
```python:
# order the cities by coordinates
sorted_cities = list(data.iloc[1:,].sort_values(['lat','lng'], ascending=False)['city_id'])
sorted_cities = [0] + sorted_cities + [0]

# calculating the distance travelled for this path:
print('Total distance with the sorted cities path is {} miles'.format(round(distance(data,sorted_cities),0)))
Total distance with the sorted cities path is 1099608.0 miles
```
We then merge the new path with the data to be able to plot it on the world map:
```python:
# merge the path with the data
df_path = pd.DataFrame({'city_id':sorted_cities}).merge(data, how = 'left', on=['city_id'])

# plot the path on a map
fig = px.line_mapbox(df_path, 
                     lat="lat", 
                     lon="lng", 
                     hover_name="city", 
                     hover_data=["country","prime_city"],
                     zoom=1, 
                     height=500)
fig.update_layout(mapbox_style="carto-positron")
fig.update_layout(margin={"r":0,"t":0,"l":0,"b":0})
fig.show()
```
![map_latitudes_path](https://github.com/mriffaud/Travelling-Santa-challenge/blob/main/image/map_latitudes_path.png)

The map above shows Santa going West to East. This is somewhat inefficient. So maybe we could look into ordering the cities but starting with the longitude coordinates:
```python:
# order the cities by coordinates
sorted_cities = list(data.iloc[1:,].sort_values(['lng','lat',], ascending=True)['city_id'])
sorted_cities = [0] + sorted_cities + [0]

# calculating the distance travelled for this path:
df_path = pd.DataFrame({'city_id':sorted_cities}).merge(data, how = 'left', on=['city_id'])
Total distance with the sorted cities path is 505422.0 miles
```
We then merge the new path with the data to be able to plot it on the world map:
```python:
# merge the path with the data
df_path = pd.DataFrame({'city_id':sorted_cities}).merge(data, how = 'left', on=['city_id'])

# plot the path on a map
fig = px.line_mapbox(df_path, 
                     lat="lat", 
                     lon="lng", 
                     hover_name="city", 
                     hover_data=["country","prime_city"],
                     zoom=1, 
                     height=500)
fig.update_layout(mapbox_style="carto-positron")
fig.update_layout(margin={"r":0,"t":0,"l":0,"b":0})
fig.show()
```
![map_longitudes_path](https://github.com/mriffaud/Travelling-Santa-challenge/blob/main/image/map_longitudes_path.png)

This map shows Santa going North to South. It is more efficient than ordering the cities by latitude but still quite inefficient so we need to find another method to create the path. 

# Path using Nearest Neighbours
We want to find the nearest city location to the previous city visited starting from Rovaniemi in Finland which is Santa's home. For that, we will be defining our own function called `nearest_neighbour()`. 
This function finds a path visiting all the cities in Santa's list using nearest neighbour. The principle behind nearest neighbour method is to find the point closest in distance to the new point. In this application, it starts in one city and connects with the closest unvisited one, repeating the same distance calculation until every city has been visited. It then returns to the starting city. For more information on nearest neighbour please visit [this page](https://en.wikipedia.org/wiki/Nearest_neighbour_algorithm).

`nearest_neighbour()` starts by storing multiple information such as a copy of the data called `cities`, it creates an array called `ids` where all the `city_ids` are stored and finally, it stores all the cities' coordinates in an array called `xy`. It then sets the path starting point as `path = [0]` since we started in Rovaniemi.

The next part of the function is a while loop to execute the following statements if the length of the array `ids` is greater than zero, in other words, the loop runs as long as we have cities to visit. This ensures we visit all cities. In the loop, the first thing is to set the coordinates from the previous stop which are stored in the path and therefore equal to `path[-1]`. once we have those set and stored in `last_x, last_y` we can calculate the distances from the previous city to all the remaining cities to visit. The minimum distance is selected, and its index is stored in `nearest_index` and appended to the path. The selected nearest city is then deleted from `ids` and its coordinates from `xy`.

Once all the cities from `ids` have been visited, the loop stops, and we append Rovaniemi as the last city.

```python:
def nearest_neighbour():
  '''finds a path through the cities using a nearest neighbour.'''
  cities = data.copy() 
  ids = cities.city_id.values[1:] 
  xy = np.array([cities.lat.values, cities.lng.values]).T[1:] 
  path = [0] 
  while len(ids) > 0: 
    last_x, last_y = cities.lat[path[-1]], cities.lng[path[-1]] 
    dist = ((xy - np.array([last_x, last_y]))**2).sum(-1) 
    nearest_index = dist.argmin()
    path.append(ids[nearest_index]) 
    ids = np.delete(ids, nearest_index, axis=0) 
    xy = np.delete(xy, nearest_index, axis=0)
  path.append(0)
  return path
```

We can now run the `distance()` function to see how the new path is performing:

```python:
nnpath = nearest_neighbour()
print('Total distance with the Nearest Neighbour path is {} miles'.format(round(distance(data,nnpath),0)))
Total distance with the Nearest Neighbour path is 124644.0 miles
```

We can now merge the new path with the data to be able to plot it on the world map:
```python:
# merge the path with the data
df_path = pd.DataFrame({'city_id':nnpath}).merge(data, how = 'left', on=['city_id'])

# plot the path on a map
fig = px.line_mapbox(df_path, 
                     lat="lat", 
                     lon="lng", 
                     hover_name="city", 
                     hover_data=["country","prime_city"],
                     zoom=1, 
                     height=500)
fig.update_layout(mapbox_style="carto-positron")
fig.update_layout(margin={"r":0,"t":0,"l":0,"b":0})
fig.show()
```
![map_nn_path](https://github.com/mriffaud/Travelling-Santa-challenge/blob/main/image/map_nn_path.png)

This path is significantly more efficient than any of the previous ones. However, so far we doid not address the constraint mentioned by Rudolph in this challenge. 

# Path using Nearest Neighbours with the constraint
In the challenge, it is mentioned that a penalty of 10% is applied when the 10th visited city is not a prime city location. In this section we will address this problem and see if taking this constraint into account helps reduce the path mileage.

The first step is to have a look how spread the prime cities across the wold map:
```python:
# plot the map highliting the prime cities
fig = px.scatter_mapbox(data, 
                        lat="lat", 
                        lon="lng", 
                        hover_name="city",
                        color="prime_city", 
                        zoom=1, height=500)
fig.update_layout(mapbox_style="carto-positron")
fig.update_layout(margin={"r":0,"t":0,"l":0,"b":0})
fig.show()
```
![map_prime_cities](https://github.com/mriffaud/Travelling-Santa-challenge/blob/main/image/map_prime_cities.png)

We can see that the prime cities are relatively spread across the map.

Next, we will define a function that uses nearest neighbour with the added constraint called `prime_nearest_neighbour()`. This function follows the same steps as the `nearest_neighbour()` with the exception that in the loop, we add if statements:
* In the first if statement we look at if the next step is going to be a prime city, since the penalty applies when we are leaving a prime city, we add 1 to `stop_number`. We also know that the number of prime cities is lesser than the total number of cities so we can only search for a prime city while then we still have some non-visited. Therefore, the next condition is to have a number of non-visited prime cities greater than zero. If both conditions are met, we calculate the distances from the previous stop to all the available prime cities, select the index of the closest one and store it in `nprime_index`. From here we need to find the selected prime location coordinates in the main array `xy`. This is because the indexes in `xy_primes` do not match the indexes in `xy` but `xy` indexes match the data ones. Once we have found the coordinates from the prime location in `xy` we can return the index and store it in `nearest_index` and proceed with removing the city from the various arrays before running the next loop.
If the conditions above are not matching, this means that we do not need to leave from a prime city on the next step and therefore we can proceed as we did in the `nearest_neighbour()`.
* The second if statement is nested in the previous one when the conditions are not met. It checks if the city selected is a prime city if gets located in `xy_primes` array and deleted.
We then add a stop to the total `stop_number` and append the final location to the path.

```python:
def prime_nearest_neighbour():
  '''finds a path through the cities using a nearest neighbour including the carrot constraint using the prime cities.'''
  cities = data.copy()
  ids = cities.city_id.values[1:] 
  xy = np.array([cities.lat.values, cities.lng.values]).T[1:] 
  xy_primes = np.array([cities[cities.prime_city == 1].lat.values, cities[cities.prime_city == 1].lng.values]).T[:] 
  path = [0] 
  stop_number = 1 
  while len(ids) > 0: 
    last_x, last_y = cities.lat[path[-1]], cities.lng[path[-1]] 
    if ((stop_number+1) % 10 == 0) and len(xy_primes) > 0: 
      dist = ((xy_primes - np.array([last_x, last_y]))**2).sum(-1) 
      nprime_index = dist.argmin() 
      nearest_index = np.where(xy == np.array([xy_primes[nprime_index][0], xy_primes[nprime_index][1]]))[0][0] 
      path.append(ids[nearest_index]) 
      ids = np.delete(ids, nearest_index, axis=0)
      xy = np.delete(xy, nearest_index, axis=0)
      xy_primes = np.delete(xy_primes, nprime_index, axis=0) 
    else:
      dist = ((xy - np.array([last_x, last_y]))**2).sum(-1)
      nearest_index = dist.argmin() 
      path.append(ids[nearest_index]) 
      if np.any(xy_primes == np.array([xy[nearest_index][0], xy[nearest_index][1]])):
        nprime_index = np.where(xy_primes == np.array([xy[nearest_index][0], xy[nearest_index][1]]))[0][0] 
        xy_primes = np.delete(xy_primes, nprime_index, axis=0) 
      #  xy_primes = np.delete(xy_primes, nearest_index, axis=0) 
      ids = np.delete(ids, nearest_index, axis=0) 
      xy = np.delete(xy, nearest_index, axis=0) 
    stop_number = stop_number+1 
  path.append(0)
  return path
```
We can now run the `distance()` function to see if adding the constraint reduced the total distance:

```python:
pnn_path = prime_nearest_neighbour()
print('Total distance with the Carrot Constraint Nearest Neighbour with path is {} miles'.format(round(distance(data,pnn_path),0)))
Total distance with the Carrot Constraint Nearest Neighbour with path is 158091.0 miles
```

We merge the path with the data and plot it a map:
```python:
# merge the path with the data
df_path = pd.DataFrame({'city_id':pnn_path}).merge(data, how = 'left', on=['city_id'])

# plot the path on a map
fig = px.line_mapbox(df_path, 
                     lat="lat", 
                     lon="lng", 
                     hover_name="city", 
                     hover_data=["country","prime_city"],
                     zoom=1, 
                     height=500)
fig.update_layout(mapbox_style="carto-positron")
fig.update_layout(margin={"r":0,"t":0,"l":0,"b":0})
fig.show()
```
![map_pnn_path](https://github.com/mriffaud/Travelling-Santa-challenge/blob/main/image/map_pnn_path.png)

We notice that the rewards from leaving from a prime city every 10th step does not outperform the penalty of not choosing the next city solely based on its distance. This may be because some prime cities are too far from some world regions and cannot effectively be added without compromising on efficiency.

Thank you for taking part in this year's workshop! 

Have a lovely Christmas holiday.

<img src="https://mir-s3-cdn-cf.behance.net/project_modules/max_1200/21a4e076438773.5c759e561c5e2.gif" alt="rudolph_thumbs_up" width="300"/>










