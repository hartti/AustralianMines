# AustralianMines
This is documentation for analyzing data about Australian mines

![](img/australian_mines.svg)

<details>
  <summary>Markup for the model</summary>
  
Markup for defining data model using the Arrows graph diagraming tool at http://www.apcjones.com/arrows/#
```
<ul class="graph-diagram-markup" data-internal-scale="1.45" data-external-scale="1">
  <li class="node" data-node-id="0" data-x="170.58362237338358" data-y="-373.75836181640625">
    <span class="caption">State</span><dl class="properties"><dt>name</dt><dd>String</dd><dt>shortName</dt><dd>String&lt;3&gt;</dd></dl></li>
  <li class="node" data-node-id="3" data-x="-390.9741100442819" data-y="-307.93059881802293">
    <span class="caption">Mine</span><dl class="properties"><dt>id</dt><dd>Int</dd><dt>name</dt><dd>String</dd><dt>status</dt><dd>String</dd><dt>location</dt><dd>Coordinates</dd></dl></li>
  <li class="node" data-node-id="4" data-x="374.19542089001914" data-y="50.68015052532327">
    <span class="caption">Commodity</span><dl class="properties"><dt>name</dt><dd>String</dd></dl></li>
  <li class="node" data-node-id="5" data-x="-428.21492688409234" data-y="94.45329468825771">
    <span class="caption">Company</span><dl class="properties"><dt>name</dt><dd>String</dd><dt>listedAt</dt><dd>String</dd><dt>HQInCountry</dt><dd>String</dd></dl></li>
  <li class="relationship" data-from="3" data-to="0">
    <span class="type">IS_LOCATED_IN</span>
  </li>
  <li class="relationship" data-from="5" data-to="3">
    <span class="type">OWNS</span><dl class="properties"><dt>percentage</dt><dd>Float</dd></dl></li>
  <li class="relationship" data-from="3" data-to="4">
    <span class="type">PRIMARY</span>
  </li>
  <li class="relationship" data-from="3" data-to="4">
    <span class="type">SECONDARY</span>
  </li>
</ul>
```
</details>

## Where to get the data

The mine and mineral data can be downloaded from AUSGIN Geoscience Portal at http://portal.geoscience.gov.au

The mine data can be found by choosing "Add Data" then under "Mines and Mining Activity" select "Mines (new version)", select only "Geoscience Australia" and download the data

The mineral data can be found by choosing "Add Data" then under "Mineral Occurrences and Resources", select "Mineral Occurrences (new version), select only "Geoscience Australia" and download the data.

Other datasets?
* commodity production data
* state boundary data
* details about the companies

## Steps to prepare the data for Neo4j

Currently the following steps are done for the data downloaded from AUSGIN using Trifacta (free online version). I will add other method for preparing the data for example using Python and Panda

### Steps needed:
* From both data sets, create a new id column by extracting 6-digit id from column "gml:id". This id is more reliable for joining the data sets than the mine names
* Join the two datasets to a single file using the id-key generated above. There are plenty of empty colums and duplicate columms, which can be dropped
* There are a lot rows which are not that interesting for mine analysis. Use column erl:status to drop such rows ("historic mine" and "mineral deposit") - this will remove roughly 3500 rows from the data set leaving a little over 500 rows
* There are multiple items on certain cells on the following colums ("erl:owner", "erl:commodity"). There are quotation marks around those cells. Drop those. Also change the item delimiter in those cells from "," to ";"
* There are both primary commodities and secondary products listed in column "erl:commodity". The secondary commodities are wrapped with parenthesis. Split the column in two using delimiter ";(". Name the first column to "PrimaryCommodities" and the second "SecondaryCommodities". Remove the trailing ")" from the column "SecondaryCommodities"

## Steps to import the CSV data to Neo4j

The following brute-force method of importing the data need polishing, but it gets the job done. I will work on improving the process.
The assumption is that you have either Neo4j desktop client installed or you are using some online storage to host your Neo4j database. The prerequisite is that you have a running, empty instance of Neo4j database, which also has APOC installed (for reverse geocoding).

1. Copy your csv-file to the Import-directory of your database and name the file "output.csv"
2. Create the mines
```
LOAD CSV WITH HEADERS FROM 'file:///output.csv' AS row
WITH row
MERGE (m:Mine {name: row["erl:name"], id: row["mineId"], status: row["erl:status"], location: row["erl:shape"], geology: row["erl:geologicHistory"]})
```
3. Create the primary production commodities and the relationships
```
LOAD CSV WITH HEADERS FROM 'file:///output.csv' AS row
WITH row, SPLIT(row.PrimaryCommodities, ";") AS primaries
UNWIND primaries AS primary
MATCH (m:Mine) WHERE m.id = row.mineId
FOREACH (ignoreMe in CASE WHEN exists(row.PrimaryCommodities) THEN [1] ELSE [] END | MERGE (c:Commodity {name:primary}) MERGE (m)-[:PRIMARY]->(c))
```
4. Create the secondary production commoditions and the relationships
```
LOAD CSV WITH HEADERS FROM 'file:///output.csv' AS row
WITH row, SPLIT(row.PrimaryCommodities, ";") AS primaries
UNWIND primaries AS primary
MATCH (m:Mine) WHERE m.id = row.mineId
FOREACH (ignoreMe in CASE WHEN exists(row.PrimaryCommodities) THEN [1] ELSE [] END | MERGE (c:Commodity {name:primary}) MERGE (m)-[:PRIMARY]->(c))
```
5. Create the companies and ownership relations
```
LOAD CSV WITH HEADERS FROM 'file:///output.csv' AS row
WITH row, SPLIT(row["erl:owner"] , ";") AS owners
UNWIND owners AS owner
MATCH (m:Mine) WHERE m.id = row.mineId
FOREACH (ignoreMe in CASE WHEN exists(row["erl:owner"]) THEN [1] ELSE [] END | MERGE (c:Company {name:owner}) MERGE (c)-[:OWNS]->(m))
```
6. The previous three statements created a Commodity and Company node with no name. Delete those entities (and the relationships they have)
```
MATCH (c:Commodity) WHERE c.name = "" DETACH DELETE c
MATCH (c:Company) WHERE c.name = "" DETACH DELETE c
```
7. Fix issues which need to handle earlier (latitude longitude)
```
match (m:Mine) set m.longitude = toFloat(replace(SPLIT(m.location," ")[1],"(",""))
match (m:Mine) set m.latitude = toFloat(replace(SPLIT(m.location," ")[2],")",""))
```
Fix Indexing

8. Add state level geo coding with reverse geocoding

This section is slightly problematic as the free reverse geocoding service (Open Street Map) used by Neo4j by default does not provide accurate or consistent enough information against the given latitude and longitude information. The Google Geocoding service works much better (giving very consistenly responses in the form "<optional street address,> <City StateAbbreviation ZIP>, Australia", but there is a cost involved and also the set-up requires configuration both for Neo4j as well as for Google services. There are also handful of results (5) which produce non-consistent location descriptions. These can be fixed manually.
```
CALL apoc.spatial.reverseGeocode(-31.740950444496217,140.6623645096) YIELD description

// this query takes very long time with Open Street Map as there is by default a significant time interval between requests
MATCH (m:Mine)
WITH m
CALL apoc.spatial.reverseGeocode(m.latitude, m.longitude) YIELD description
SET m.locationString = description

```

match (m:Mine) with m, substring(m.locationString, 0, size(m.locationString)-11) as testStr
with m, right(testStr, 4) as zip, trim(substring(testStr, size(testStr)-8, 3)) as state
match (s:State) where s.id = state
merge (m)-[:LOCATED_IN]->(s)


## Sample queries with the data

See the current schema of the data
```
call db.schema.visualization
```

Return all mines which do not have any owners listed
```
MATCH (m:Mine) WHERE NOT (m)-[]-(:Company) RETURN m
```

Return a list of mines with more than one owner (ordered by owner count - desconding)
```
MATCH (:Company)-[r]->(m:Mine)
WITH m, count(r) as Owner_count
WHERE Owner_count > 1
RETURN m.name, Owner_count
ORDER BY Owner_count DESC
```

Return a list of companies which own more than one mine (and the count of owned mines) ordered by descencing count of owned mindes 
```
MATCH (c:Company)-[r]->(:Mine)
WITH c, count(r) as Mine_count
WHERE Mine_count > 1
RETURN c.name, Mine_count
ORDER BY Mine_count DESC
```

Return a list of minerals / commodities produced by "Cameco Corporation"
```
MATCH (c:Company)-[]->(:Mine)-[]->(m:Commodity)
WHERE c.name = "Cameco Corporation"
RETURN DISTINCT m.name
ORDER BY m.name
```

Return operating mines producing Uranium with location state and owners
```
match (m:Mine)-[r1:LOCATED_IN]->(s:State), (m)-[]->(c:Commodity), (co:Company)-[r2]-(m)
where c.name = "Uranium" and m.status = "operating mine"
return m,r1,r2,s,co
```

Return mines mining gold located in NSW and QLD with owners
```
match (m:Mine)-[r1:LOCATED_IN]->(s:State), (m)-[]->(c:Commodity), (co:Company)-[r2]-(m)
where s.id IN ["QLD","NSW"] and c.name = "Gold"
return m,r1,r2,s,co
```
