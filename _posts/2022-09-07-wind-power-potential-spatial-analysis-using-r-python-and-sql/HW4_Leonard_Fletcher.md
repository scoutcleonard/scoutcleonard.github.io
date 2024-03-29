# Assignment 4: Wind Power Potential Using Python

## EDS223 Spatial Analysis / ESM 267 Advanced GIS

Student Collaborators: 
   - [Scout Leonard](https://github.com/scoutcleonard), MEDS 2022
   - [Meghan Fletcher](https://github.com/megfletch), MESM 2022

## Table of Contents
1. [Setup](#setup)

    1.a. [Load Libraries](#loadlibs)
    
    1.b. [Connect to postGIS database](#postgis)
    
2. [Identify Land Suitable for Turbine Placement](#placement)

    2.a.[Buildings](#buildings)
    
    2.b. [Airports](#airports)
    
    2.c. [Military](#military)
    
    2.d. [Nature Reserves, Parks, and Wetlands](#parks)
    
    2.e. [Highways and Railroads](highways)
    
    2.f. [Waterbodies](#waterbodies)
    
    2.g. [Power Lines](#powerlines)
    
    2.h. [Power Plants](#powerplants)
    
    2.i. [Wind Turbines](#windturbines)
    
    2.j. [Merge the Subqueries](#merge)
    
3. [Area of Data Polygons](#polygons)

4. [Number of Turbines Per Polygon](#turbines)

    4.a. [Scenario 1](#turbines_1)
    
    4.b [Scenario 2](#turbines_2)

5. [Total Energy Production](#energy)

<a id='setup'></a>
## 1. Setup

In this assignment, we will be taking spatial data from the state of Iowa and determining how much land could be utilized for wind turbine placement given specified siting constraints. From this analysis, we will then determine the overall amount of energy that could be produced under two different scenarios involving two different placement scenarios determined by distance from residential buldings (3H and 10 H, defined in the Parameters section). The setup for this assignment is broken down into five sections, linked in the table of contents above.  

First, we need to read in the various libraries necessary for performing the spatial analysis required throughout the assignment:

<a id='loadlibs'></a>
### 1.a. Load Libraries


```python
import sqlalchemy
import geopandas
import psycopg2
import pandas
import geopandas
import math
```

<a id='postgis'></a>
### 1.b. Connect to postGIS database:

Next, we need to connect to the postGIS database in order to pull in data from the the _osmiowa_ dataset which we will be analyzing in order to produce our final results. 


```python
pg_uri_template = 'postgresql+psycopg2://{user}:{pwd}@{host}/{db_name}'

db_uri = pg_uri_template.format(
    host = '128.111.89.111',
    user = 'eds223_students',
    pwd = 'eds223',
    db_name = 'osmiowa'
)
```


```python
db = sqlalchemy.create_engine(db_uri)
```

<a id='parameters'></a>
### 1.c Paramters:

Finally, to simplify the process of identifying suitable land for wind turbine placement using subqueries, we must set the parameters based on the siting constraints assigned to the various features being used within our analysis.


```python
airport_buff = 7500 #meters
h = 150 #meters
h3 = h * 3
h10 = h * 10
h2 = h * 2
d = 136 #meters
turbine_footprint = math.pi * (d*5)**2 #square meters
```

<a id='placement'></a>
## 2. Identify Land Suitable for Turbine Placement 

To identify suitable land, we run SQL queries which select all land unsuitable for windturbine placement and create suitable buffers around those sites set by wind turbine siting contraints. We then assign those queries to a variable to call in a later part of the notebook where we merge the subqueries and create a geodataframe of all the sites where wind turbines _cannot_ go. 

<a id='buildings'></a>
### Buildings

For the buildings queries, we run two queries under two different scenarios: one where the siting constraint specified turbines must be at least 3 times the turbine height in distance from residential buildings, and one where they must be 10 times the turbine height in distance. We also have a query for nonresidential buildings.

**Scenario 1:** Looking at residential buildings that would be at leasy 3H from turbines:


```python
sql_buildings_residential = """ SELECT osm_id, ST_BUFFER(way, {h}) as way
                                  FROM planet_osm_polygon 
                                  WHERE building in ('yes', 'residential', 'apartments', 'house', 'static_caravan', 'detached') 
                                  OR landuse = 'residential' 
                                  OR place = 'town'"""

# In order to properly create the buffer we need to format each parameter as follows 
sql_buildings_residential_1 = sql_buildings_residential.format(h=h3)
```

**Scenario 2:** Looking at residential buildings that would be at leasy 10H from turbines:


```python
#same query as above, but with updated buffer
sql_buildings_residential_2 = sql_buildings_residential.format(h=h10)
```


```python
sql_buildings_nonres = f"""SELECT osm_id, ST_BUFFER(way, {h3}) as way 
FROM planet_osm_polygon 
WHERE building not in ('yes', 'residential', 'apartments', 'house', 'static_caravan', 'detached') 
AND building IS NOT NULL"""
```

<a id='airports'></a>
### Airports


```python
sql_airports = f"""SELECT osm_id, ST_BUFFER(way, {airport_buff}) as way
FROM planet_osm_polygon
WHERE aeroway is not NULL"""
```

<a id='military'></a>
### Military

#### From the Military and Landuse columns:


```python
sql_military = f"""SELECT osm_id, way
FROM planet_osm_polygon 
WHERE military is not NULL
OR landuse = 'military'"""
```

<a id='parks'></a>
### Nature Reserves, Parks and Wetlands


```python
sql_nature = f"""SELECT osm_id, way
FROM planet_osm_polygon 
WHERE leisure in ('nature_reserve', 'park')
OR planet_osm_polygon.natural = 'wetland'"""
```

<a id='highways'></a>
### Highways and Railroads

#### From Railroads Column


```python
sql_railroads = f"""SELECT osm_id, ST_BUFFER(way, {h2}) as way
FROM planet_osm_line 
WHERE railway not in ('disused', 'abandoned')
OR highway in ('trunk', 'primary', 'secondary', 'motorway')"""
```

<a id='waterbodies'></a>
### Waterbodies

#### Rivers from the Waterway Column


```python
sql_rivers = f"""SELECT osm_id, ST_BUFFER(way, {h}) as way
FROM planet_osm_line 
WHERE waterway = 'river'"""
```

#### Lakes from the Water Column


```python
sql_lakes = f"""SELECT osm_id, way
FROM planet_osm_polygon 
WHERE water = 'lake'"""
```

<a id='powerlines'></a>
### Powerlines


```python
sql_power = f"""SELECT osm_id, ST_BUFFER(way, {h2}) as way
FROM planet_osm_line 
WHERE planet_osm_line.power IS NOT NULL"""
```

<a id='powerplants'></a>
### Power Plants and Other Power Equipment


```python
sql_powerplants = f"""SELECT osm_id, ST_BUFFER(way, {h}) as way
FROM planet_osm_polygon 
WHERE planet_osm_polygon.power IS NOT NULL"""
```

<a id='windturbines'></a>
### Wind Turbines


```python
sql_turbines = f"""SELECT osm_id, ST_BUFFER(way, 5 * {d}) as way
FROM planet_osm_point
WHERE "generator:source" = 'wind'"""
```

<a id='merge'></a>
### Merge the subqueries

Next, we use `SELECT` and `UNION SELECT` to select the geometries from all the subqueries above. We do this for both the 3H and 10H residential building siting scenarios, and then use the unioned queries to create geodataframes of all the space in Iowa where turbines _cannot_ be:


```python
# Create a union of the subqueries in order to calculate the total area unavailable for wind turbine placement building scenario 1
sql_scenario_1 = f"""{sql_buildings_residential_1}
                UNION
                {sql_airports}
                UNION
                {sql_military}
                UNION
                {sql_nature}
                UNION
                {sql_railroads}
                UNION
                {sql_rivers}
                UNION
                {sql_lakes}
                UNION
                {sql_power}
                UNION
                {sql_powerplants}
                UNION
                {sql_turbines}
                """

#create a geodataframe from the query union using the column 'way' as the geometry
scenario_1_df = geopandas.read_postgis(sql_scenario_1, con = db, geom_col = 'way')
```


```python
# Create a union of the subqueries in order to calculate the total area unavailable for wind turbine placement under building scenario 2
sql_scenario_2 = f"""{sql_buildings_residential_2}
                UNION
                {sql_airports}
                UNION
                {sql_military}
                UNION
                {sql_nature}
                UNION
                {sql_railroads}
                UNION
                {sql_rivers}
                UNION
                {sql_lakes}
                UNION
                {sql_power}
                UNION
                {sql_powerplants}
                UNION
                {sql_turbines}
                """

#create a geodataframe from the query union using the column 'way' as the geometry
scenario_2_df = geopandas.read_postgis(sql_scenario_2, con = db, geom_col = 'way')
```

<a id='polygons'></a>
## 3. Area of Data Polygons 

We know that the area of Iowa is 144,669.2 km^2. However, finding the area of each of the buffers we've created will give us areas much larger than Iowa itself. To correct this, we need to dissolve the overlapping areas. We can dissolve and solve for the are in one step for each scenario.


```python
# Total area under scenario 1
scenario_1_df.dissolve().area/1000/1000
```




    0    64197.937194
    dtype: float64




```python
# Total area under scenario 2
scenario_2_df.dissolve().area/1000/1000
```




    0    72131.583716
    dtype: float64



<a id='turbines'></a>
## 4. Total Number of Turbines per Polygon and Total Energy Production

Now we need to calculate the total number of wind turbines that could be placed in each suitable cell. This suitability is based on the parameters established using the subqueries from above as well as an additional dataset that looks at 10 km^2 polygons with associated average annual wind speeds, which is called using SQL in the next code chunk. Once we determine the suitable cells and then the total number of placeable turbines, we will be able to calculate the total energy production that all of the wind turbines could create.


```python
# Select for wind cells to determine the wind cells suitable for wind turbine placement
sql_wind_grid = """SELECT * FROM wind_cells_10000"""
wind_grid_df = geopandas.read_postgis(sql_wind_grid, con = db, geom_col = 'geom')
```

Next, we subtract the union of the siting constraints (for both scenarios from the wind cells, leaving only the cells that could accomodate new wind turbines using `overlay()`:

<a id='turbines_1'></a>
### Scenario 1


```python
wind_constraint_cells_1 = wind_grid_df.overlay(scenario_1_df, how ='difference')
```

    /Users/scoutleonard/opt/anaconda3/envs/eds223/lib/python3.8/site-packages/geopandas/geodataframe.py:2196: UserWarning: `keep_geom_type=True` in overlay resulted in 88 dropped geometries of different geometry types than df1 has. Set `keep_geom_type=False` to retain all geometries
      return geopandas.overlay(


Now, we are left with a dataframe of the areas that are suitable for wind turbine placement. 

Next we calculate total number of turbines per polygon and use that estimate and the wind per cell to predict the potential for wind energy in the two different scenarios: 


```python
#create area column based on geom column which contains area of the polygon in each cell based on the geom column
wind_constraint_cells_1["area"] = wind_constraint_cells_1["geom"].area
```


```python
#calculate the total number of turbines that could fit in the total area by dividing the total area by turbine footprint
wind_constraint_cells_1["turbine_no"] = wind_constraint_cells_1["area"]/turbine_footprint
```


```python
#calculate the energy per single turbine per cell based on the wind speed in that cell 
wind_constraint_cells_1["energy_per_turbine"] = (wind_constraint_cells_1["wind_speed"]*2.6)-5
```


```python
#calculate the total energy 
wind_constraint_cells_1["energy_per_cell"] = wind_constraint_cells_1["turbine_no"]*wind_constraint_cells_1["energy_per_turbine"]
```


```python
total_energy_scenario_1 = wind_constraint_cells_1['energy_per_cell'].sum()
```

<a id='turbines_2'></a>
### Scenario 2

We repeat the process of calculating total energy per cell in the available polygons, but for the second scenario now. Note the overlay using the second query:


```python
wind_constraint_cells_2 = wind_grid_df.overlay(scenario_2_df, how ='difference')
```


```python
wind_constraint_cells_2["area"] = wind_constraint_cells_2["geom"].area
```


```python
wind_constraint_cells_2["turbine_no"] = wind_constraint_cells_2["area"]/turbine_footprint
```


```python
wind_constraint_cells_2["energy_per_turbine"] = (wind_constraint_cells_2["wind_speed"]*2.6)-5
```


```python
wind_constraint_cells_2["energy_per_cell"] = wind_constraint_cells_2["turbine_no"]*wind_constraint_cells_2["energy_per_turbine"]
```


```python
total_energy_scenario_2 = wind_constraint_cells_2['energy_per_cell'].sum()
```

<a id='energy'></a>
## 5. Total Energy Production


```python
print('The total energy of all the turbines that could be placed in scenario 1 is', total_energy_scenario_1, 'GWh.')
```

    The total energy of all the turbines that could be placed in scenario 1 is 1065473.0533352676 GWh.



```python
print('The total energy of all the turbines that could be placed in scenario 2 is', total_energy_scenario_2, 'GWh.')
```

    The total energy of all the turbines that could be placed in scenario 2 is 967482.4765797912 GWh.



```python

```
