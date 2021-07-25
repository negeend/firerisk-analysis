# firerisk-analysis

## Introduction
The aim of this project is to gather and integrate various datasets to analyse how different factors in Greater Sydney, New South Wales (NSW) neighborhoods can affect the risk of bushfire.

## Data Description:
### StatisticalAreas.csv:
This csv file contains information regarding different areas around NSW; containing the area id, which is unique to each neighbourhood and is thus the primary key. The dataset also contains the names of the areas as well as the area id of the parent area, under the column “parent_area_id”

### Neighbourhoods.csv:
This csv file is central to this project as it contains all the information specific to each of the areas. With 322 area id’s, this file contains values regarding the different areas’ population, number of dwellings, number of businesses, median income and the average monthly rent.

### BusinessStats.csv:
This file is linked to the other data sets through the primary key being the area id, however, it also has the “number_of_businesses” column in common with Neighbourhoods.csv. The data contained within it is regarding the nature of the businesses within each area, where the businesses are categorised as one of the following: accommodation and food, retail trade, agriculture, forestry and fishing, health care and social assistance, public administration and safety, and transport, postal and warehousing.

### RFSNSW_BFPL.shp:
This is a geometry shape file which contains bush fire prone land vegetation spatial data from the NSW Rural Fire Service. Specifically, the geometries within the data set are single entity, point types which indicate the centres of bush fire prone areas around NSW. Along with the geometry objects, the file also provides information regarding the shape length and area of each bush fire prone area which is centred at each geometry point. Finally, a category score is given which provides an additional scale as to the fire risk of each area.

### SA2_2016_AUST.shp:
This is another shape file with Statistical Area 2 (SA2) data from the Australian Bureau of Statistics (ABS). This file can be linked to the other datasets by the “SA2_MAIN16” column which contains area id’s. It also allows us to connect each neighbourhood to a geometry Multipolygon object which represents the boundaries of the area.

### groundwater.json:
Thisl dataset contains geospatial data in the JSON format, pertaining to the groundwater types of different areas around NSW, VIC and QLD. The data was obtained from the Australian Government website as referenced in appendix E.

### Obtention and pre-processing of data:
The CSV files were all loaded into our Jupyter workspace through the use of the Python Pandas library, specifically the ‘read_csv()’ function. On the other hand, for the 2 Geometry shape files and our own additional data set, we made use of the GeoPandas library to parse and load in the data. Specifically, we used the ‘read_file()’ function which returns a GeoDataFrame object which then allows us to access the geometries of the file entries or the associated data attributes.
Further work was required on the spatial datasets so that they could be loaded into the tables created in PostGIS. We used the geoalchemy2 Python library to convert the Python shapely types that are used by GeoPandas into the Well-Known Text (WKT) format so that they are suitable for PostGIS. In the process of this conversion we also ensure that all the geometry objects are in MultiPolygon format. Subsequently, we remove the ‘geometry’ columns since the new ‘geom’ column is created with the elements being MultiPolygons of WKT format, and we want out GeoPandas DataFrame columns to match our schema.
Finally, in regard to the data cleaning process, first copies of all the datasets were created so that the original data can remain unaltered. We then removed the duplicate rows from each data set as well as null values. In regard to the groundwater shape file, we also removed any roes which did not lie in the jurisdiction of NSW.

## Database Description
To upload our datasets onto PostGIS so that we would be able to query our datasets through PostgreSQL, we first create schemas for the 3 given CSV files provided, the two spatial datasets, as well as our additional JSON file containing groundwater data. The UML diagram of these schemas can be seen in appendix A. The attributes and corresponding types of each schema could be determined by looking to the fields of each dataset. For the shape files we had to specify the ‘geom’ attributes as geometry objects of type MultiPolygon, with the Spatial Reference Identifier (SRID) as 4283 of the GDA94 geodetic coordinate system to store the geometric data into the Australian coordinate system.
We then use the Python Pandas ‘to_sql’ function to insert the data into our database, while ensuring to explicitly define our types so that the geoalchemy library is able to handle the
  
conversions. We use the ‘dtype’ parameter to facilitate this, in defining our spatial data as GeoaAlchemy’s type ‘Geometry’. All tables on PostgreSQL are named as follows; ‘rfsnsw_bfpl’, ‘sa2_2016_aust’, ‘statisticalareas’, ‘neighbourhoods’, ‘businessstats’, and ‘groundwater’ (see appendix A).
Subsequently, we created spatial indexes on ‘rfsnsw_bfpl’ and ‘sa2_2016_aust’ to reduce operating time as these tables are retrieved frequently throughout the project. We perform a similar spatial index on the schema created by executing a spatial join on these to datasets.

## Fire Risk Score
Formula used for fire_risk score:

1 / 1 + e^(−(𝑧_𝑝𝑜𝑝𝑢𝑙𝑎𝑡𝑖𝑜𝑛_𝑑𝑒𝑛𝑠𝑖𝑡𝑦 + 𝑧_𝑑𝑤𝑒𝑙𝑙𝑖𝑛𝑔_𝑑𝑒𝑛𝑠𝑖𝑡𝑦 + 𝑧_𝑏𝑢𝑠𝑖𝑛𝑒𝑠𝑠_𝑑𝑒𝑛𝑠𝑖𝑡𝑦 + 𝑧_𝑏𝑓𝑝𝑙_𝑑𝑒𝑛𝑠𝑖𝑡𝑦 − 𝑧_𝑎𝑠𝑠𝑖𝑠𝑡𝑖𝑣𝑒_𝑠𝑒𝑟𝑣𝑖𝑐𝑒_𝑑𝑒𝑛𝑠𝑖𝑡𝑦 − 𝑧_𝑔𝑟𝑜𝑢𝑛𝑑𝑤𝑎𝑡𝑒𝑟_𝑑𝑒𝑛𝑠𝑖𝑡𝑦))


In calculating the z scores used to retrieve this fire risk score we made several intermediate calculations which we will now discuss. Firstly, we executed a spatial join between ‘sa2_2016_aust’ and ‘neighbourhoods’, then joined the result with ‘rfsnsw_bfpl’, and inserted it into a schema to create a new table called ‘neighbourhoods_sa2_rfs’. However, since there are multiple bfpl points in each neighbourhood, we grouped by ‘area_id’ and summed up ‘SHAPE_AREA’ within each area. Nonetheless, as bfpl points in one area can be categorized differently, we created a scaling system called ‘category_scaled’ and set its values according to the following system:
● Category 1: category_scaled = 2
● Category 2: category_scaled = 1
● Category 3: category_scaled = 1.5

Seeing as category 1 has the highest risk of bushfire, followed by category 2 and finally, category 3.

Next, densities of different factors in each area were calculated and set in columns, which were then updated into ‘neighbourhoods_sa2_rfs2’. However, the overall bfpl density was calculated by dividing the sum of ‘SHAPE_AREA’ by ‘land_area’ and then multiplying by ‘category_scaled’. Finally, we took the average of all bfpl densities within a neighbourhood, and grouped them by ‘area_id’ to create a ‘bfpl_density’ column. Additionally, for the assistive service density, we took account of health care and social assistance businesses, public administration and safety businesses. Eventually, we obtained ‘population_density’, ‘dwelling_density’, ‘business_density’, ‘bfpl_density’, and ‘assistive_service_density’.

Subsequently, we executed a spatial join between ‘neighbourhoods_sa2_rfs2’ and ‘groundwater2’ and inserted it into a new table ‘neighbourhoodssa2rfs_groundwater’. For the groundwater score, it was executed similarly as bfpl was since one neighbourhood can intersect with more than one groundwater area. A scaling system for the level of groundwater, ‘sdl_type_scaled’, was created to adjust the groundwater density with the following system:
● sdl_type = Groundwater: sdl_type_scaled = 2
● sdl_type = Deep Groundwater: sdl_type_scaled = 1.5
● sdl_type = null: sdl_type_scaled = 1

We also calculate the overlapping percentage of the groundwater area intersecting with each neighbourhood area. The groundwater score was then calculated by taking the overlapping percentage multiplied by ‘sdl_type_scaled’. Eventually, we summed up the groundwater score and grouped by ‘area_id’ to find ‘groundwater_density’, and updated it into the final table ‘neighbourhoods_sa2_rfs3’.

Finally, we calculated z-score for each factor density, and integrated them into the formula to find the fire risk score of each neighbourhood.
Results: Overall, Figure 1 shows that the North-West neighbourhoods have the highest fire risk score of 0.8 – 1.0, followed by the South-West of NSW with the score ranges from 0.4 to 0.8. In contrast, the Eastern area scores lowest, 0.0 – 0.4. This substantial discrepancy may be because the vegetation is distributed densely in areas with higher risk, while assistive services are readily accessible in urban neighbourhoods.
