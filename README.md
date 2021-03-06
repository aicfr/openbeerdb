# neo4j-openbeerdb

GraphQL backend : https://github.com/aicfr/graphql-neo4j-openbeerdb

Android application : https://github.com/Gerth01/graphql-android-openbeer

Inspired by https://neo4j.com/graphgist/beer-amp-breweries-graphgist

Powered by http://openbeerdb.com and https://neo4j.com

## Docker

```
docker run -d -p 7474:7474 -p 7687:7687 -v $(pwd)/neo4j/data:/data -e NEO4J_AUTH=neo4j/openbeerdb neo4j:3.4.0
```

* <http://localhost:7474>
  * User : neo4j
  * Password : openbeerdb

## Schema

![openbeerdb](http://www.plantuml.com/plantuml/png/XP3F2eCm38VlFaMF2le2Emp_PKuc4wemF8dM88AsKZk6Rx_gt8Q7iLCI-BvVIDn9LLBTXvw84NcDA9lQC7rTKLh4SOva7Ino8DJB8JHbfJhUseI9OK2kT2EnKjXuolhX1MS8Bjhi1TEku3OPPSVmZ_bouwmyqHYkOPcUpAdZ33VsJKfye9mNrATmWx3qyXqGw0sjs0W0MWRwYYYbnOxArDpZ1ydo8W7ZcxqY4GeccXMA84tI9MVTvY9lzF-U "openbeerdb")

* Beerer
  * beererID
  * beererName
  * location
  * description
  * website
  * picture

* Beer
  * beerID
  * beerName
  * description
  * abv : alcohol by volume
  * picture

* Brewery
  * breweryID
  * breweryName
  * address1
  * city
  * state
  * zipCode
  * country
  * phoneNumber
  * website
  * description

* Category
  * categoryID
  * categoryName

* Style
  * styleID
  * styleName

* Geocode
  * geocodeID
  * location (latitude, longitude)

## Create nodes, indexes and relationships

For cql script, view `cql\load_openbeerdb.cql`. Thanks @rvanbruggen !

### Create nodes

```sql
LOAD CSV WITH HEADERS FROM 'https://raw.githubusercontent.com/aicfr/neo4j-openbeerdb/master/beerers.csv' AS row
CREATE (:Beerer { beererID: toInteger(row.id), beererName: row.name, location: row.location, description: row.descript, website: row.website, picture: row.picture })

LOAD CSV WITH HEADERS FROM 'https://github.com/aicfr/neo4j-openbeerdb/raw/master/beers.csv' AS row
CREATE (:Beer { beerID: toInteger(row.id), beerName: row.name, description: row.descript, abv: toFloat(row.abv) })

LOAD CSV WITH HEADERS FROM 'https://github.com/aicfr/neo4j-openbeerdb/raw/master/breweries.csv' AS row
CREATE (:Brewery { breweryID: toInteger(row.id), breweryName: row.name, address1: row.address1, city: row.city, state: row.state, zipCode: row.code, country: row.country, phoneNumber: row.phone, website: row.website, description: row.descript })

LOAD CSV WITH HEADERS FROM 'https://github.com/aicfr/neo4j-openbeerdb/raw/master/categories.csv' AS row
CREATE (:Category { categoryID: toInteger(row.id), categoryName: row.cat_name })

LOAD CSV WITH HEADERS FROM 'https://github.com/aicfr/neo4j-openbeerdb/raw/master/styles.csv' AS row
CREATE (:Style { styleID: toInteger(row.id), styleName: row.style_name })

LOAD CSV WITH HEADERS FROM 'https://github.com/aicfr/neo4j-openbeerdb/raw/master/geocodes.csv' AS row
CREATE (:Geocode { geocodeID: toInteger(row.id), latitude: toFloat(row.latitude), longitude: toFloat(row.longitude) })
```

### Create indexes

```sql
CREATE INDEX ON :Beerer(beererID)
CREATE INDEX ON :Beer(beerID)
CREATE INDEX ON :Brewery(breweryID)
CREATE INDEX ON :Category(categoryID)
CREATE INDEX ON :Style(styleID)
CREATE INDEX ON :Geocode(geocodeID)
```

### Create relationships

```sql
MATCH (b1:Beerer {beererID: 1})
MATCH (b2:Beerer {beererID: 6})
CREATE (b1)-[i:IS_FRIEND_OF]->(b2)
SET i.since = '1997-04-22'

MATCH (b1:Beerer {beererID: 3})
MATCH (b2:Beerer {beererID: 6})
CREATE (b1)-[i:IS_FRIEND_OF]->(b2)
SET i.since = '2009-01-20'

MATCH (beerer:Beerer {beererID: 3})
MATCH (beer:Beer {beerID: 4265})
CREATE (beerer)-[r:RATED]->(beer)
SET r.rating = 5,r.comment = '',r.createdAt = timestamp()

MATCH (beerer:Beerer {beererID: 3})
MATCH (beer:Beer {beerID: 4265})
CREATE (beerer)-[c:CHECKED]->(beer)
SET c.location = 'Vineuil, FR',c.price = 4.5,c.createdAt = timestamp()

USING PERIODIC COMMIT
LOAD CSV WITH HEADERS FROM "https://github.com/aicfr/neo4j-openbeerdb/raw/master/beers.csv" AS row
MATCH (beer:Beer {beerID: toInteger(row.id)})
MATCH (brewery:Brewery {breweryID: toInteger(row.brewery_id)})
MERGE (beer)-[:BREWED_AT]->(brewery)

USING PERIODIC COMMIT
LOAD CSV WITH HEADERS FROM "https://github.com/aicfr/neo4j-openbeerdb/raw/master/beers.csv" AS row
MATCH (beer:Beer {beerID: toInteger(row.id)})
MATCH (category:Category {categoryID: toInteger(row.cat_id)})
MERGE (beer)-[:BEER_CATEGORY]->(category)

USING PERIODIC COMMIT
LOAD CSV WITH HEADERS FROM "https://github.com/aicfr/neo4j-openbeerdb/raw/master/beers.csv" AS row
MATCH (beer:Beer {beerID: toInteger(row.id)})
MATCH (style:Style {styleID: toInteger(row.style_id)})
MERGE (beer)-[:BEER_STYLE]->(style)

USING PERIODIC COMMIT
LOAD CSV WITH HEADERS FROM "https://github.com/aicfr/neo4j-openbeerdb/raw/master/geocodes.csv" AS row
MATCH (brewery:Brewery {breweryID: toInteger(row.brewery_id)})
MATCH (geocode:Geocode {geocodeID: toInteger(row.id)})
MERGE (brewery)-[:GEOLOCATED_AT]->(geocode)

// Neo4j 3.4 locations added from Geocode
MATCH (geocode:Geocode)
SET geocode.location = point({latitude: geocode.latitude, longitude: geocode.longitude});

CREATE INDEX ON :Geocode(location)
CREATE INDEX ON :Beer(beerName)
CREATE INDEX ON :Beerer(beererName)
CREATE INDEX ON :Brewery(breweryName)
CREATE INDEX ON :Category(categoryName)
CREATE INDEX ON :Style(styleName)
```

## Queries

For more queries, view `cql\query_openbeerdb.cql`. Thanks @rvanbruggen !

### Beers for a particular category

```sql
MATCH (category:Category {categoryName: "German Lager"}) <- [:BEER_CATEGORY]- (beer:Beer)
RETURN DISTINCT(beer.beerName) as beer
```

### Beers for a particular category and brewery location

```sql
MATCH (category:Category {categoryName: "British Ale"}) <- [:BEER_CATEGORY]- (beer:Beer) -[:BREWED_AT] -> (brewery:Brewery {country: "United Kingdom"})
RETURN DISTINCT(beer.beerName) as beer
```

### Location : Beers around seattle

```sql
WITH point({latitude: 47.608013, longitude: -122.335167}) AS seattle
MATCH path = (geocode:Geocode)--(br:Brewery)--(b:Beer)
WHERE distance(geocode.location, seattle) < 200000
RETURN path
```

## Resources

* <https://graphaware.com/neo4j/2013/10/11/neo4j-bidirectional-relationships.html>
* <https://gist.github.com/rvanbruggen/a9bde09c4a305261437347a8145e811b>
* <https://blog.bruggen.com/2018/06/exploring-new-datatypes-in-neo4j-34-and.html>