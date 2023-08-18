# neo4j-for-diseases
This file contains the updated commands from Sixing Huang's [Neo4j for diseases](https://towardsdatascience.com/neo4j-for-diseases-959dffb5b479) article

---

The docker container can by pulled from https://hub.docker.com/repository/docker/dantkachov/neo4j-csv/general
Pull the container by running:
```bash
docker pull dantkachov/neo4j-csv:0.1
```
Running the docker container:
```bash
docker run --name neo4j-csv-container -p 7474:7474 -p 7687:7687 -d -e NEO4J_AUTH=neo4j/password neo4j-csv
```

# Section 1
Load everything:
```cypher
LOAD CSV WITH HEADERS FROM 'file:///disease.csv' AS row MERGE (n:disease {name: row.name, ko: row.ko, description: row.description, disease_category:row.disease_category});
LOAD CSV WITH HEADERS FROM 'file:///drug.csv' AS row MERGE (n:drug {name: row.name, ko: row.ko});
LOAD CSV WITH HEADERS FROM 'file:///pathogen.csv' AS row MERGE (n:pathogen {name: row.name, ko: row.ko, taxonomy: row.taxonomy});

CREATE CONSTRAINT FOR (n:disease) REQUIRE n.ko IS UNIQUE;
CREATE CONSTRAINT FOR (n:drug) REQUIRE n.ko IS UNIQUE;
CREATE CONSTRAINT FOR (n:pathogen) REQUIRE n.ko IS UNIQUE;

LOAD CSV WITH HEADERS FROM 'file:///drug_disease.csv' AS row MERGE (n1:drug {ko: row.from}) MERGE (n2:disease {ko: row.to}) MERGE (n1)-[r:treats]->(n2);
LOAD CSV WITH HEADERS FROM 'file:///pathogen_disease.csv' AS row MERGE (n1:pathogen {ko: row.from}) MERGE (n2:disease {ko: row.to}) MERGE (n1)-[r:causes]->(n2);
```

Return the entire graph:
```cypher 
MATCH (n)-[r]->(m)
RETURN n, r, m
```

# Section 2: Get Overviews
Get the three types of nodes:
```cypher
MATCH (p:pathogen) RETURN COUNT(DISTINCT p)
MATCH (dr:drug) RETURN COUNT(DISTINCT dr)
MATCH (di:disease) RETURN COUNT(DISTINCT di)
```
Another way to get the entire graph:
```cypher
MATCH p=(n:disease) <-[]-() RETURN p;
```
Get the top 10 disease categories in the dataset. Will not double count specific diseases:
```cypher
MATCH (d:disease) RETURN d.disease_category, COUNT(d.disease_category) as count ORDER BY count DESC LIMIT 10;
```
Show the number of infectious diseases in the database:
```cypher
MATCH (v:pathogen) RETURN split(v.taxonomy, "; ")[0] as domain, COUNT(split(v.taxonomy, "; ")[0]) as count ORDER BY count DESC;
```
Get the number of drugs against infectious diseases:
```cypher
MATCH (dr:drug) -[]->(di:disease) 
WHERE di.disease_category CONTAINS 'infectious disease'
RETURN COUNT(DISTINCT(dr));
```
# Section 3: Search for multipurpose drugs
Get the top ten most versatile medicines:
```cypher
MATCH (dr:drug) -[r:treats]->(di:disease) RETURN dr.name as drug, COUNT(r) as count ORDER BY count DESC LIMIT 10;
```
Get the top 10 drugs and the diseases they treat:
```cypher
MATCH (a:drug)-[r:treats]->(b:disease)
WITH a, COUNT(r) AS indicationCount
     ORDER BY indicationCount DESC LIMIT 10
     MATCH (a:drug)-[r:treats]->(b:disease)
     RETURN a,r,b
```
See details about drugs (prednisolone):
```cypher
MATCH (dr:drug) -[]->(di:disease) WHERE dr.name="Prednisolone sodium phosphate" RETURN  di.disease_category, count(di.disease_category) as count ORDER BY count DESC LIMIT 10;
```
``` cypher
MATCH (dr:drug) -[]->(di:disease) WHERE dr.name="Prednisolone sodium phosphate" AND (di.disease_category = "Lung disease" or di.disease_category CONTAINS 'infectious disease') RETURN di.name, di.disease_category ORDER BY di.disease_category;
```

# Section 4: Details about some pathogens
Count how many diseases a pathogen can cause:
```cypher
MATCH (p:pathogen) -[r:causes]->(di:disease) RETURN p.name, COUNT(r) as count ORDER BY count DESC LIMIT 10;
```
Find what HPV causes: (original has 16 and 18, this includes all)
```cypher
MATCH a=(p:pathogen) -[r:causes]->(di:disease) WHERE p.name CONTAINS "papillomavirus" RETURN a;
```
Same as above but for Hydroxychloroquine
```cypher
MATCH a=(dr:drug) -[] ->() WHERE dr.name CONTAINS "Hydroxychloroquine" RETURN a
```

# Section 5: Discover highly connect nodes and communities with graph algorithms
Create graph projection:
```cypher
CALL gds.graph.project.cypher(
	'disease-graph',
	'MATCH (n) RETURN id(n) AS id',
	'MATCH (n)--(m) RETURN id(n) AS source, id(m) AS target'
)
```
Form disease communities wit Louvain Algorithm:
```cypher
CALL gds.louvain.stats('disease-graph')
YIELD communityCount
```
Check the largest communities from the louvian algorithm
```cypher
CALL gds.louvain.stream('disease-graph')
YIELD nodeId, communityId, intermediateCommunityIds
RETURN  communityId, COUNT(communityId) as count
ORDER BY count DESC LIMIT 10;
```
Check members in the community above:
```cypher
CALL gds.louvain.stream('disease-graph')
YIELD nodeId, communityId, intermediateCommunityIds
WHERE communityId = 3421
RETURN  gds.util.asNode(nodeId).name AS name, communityId
```
Find the most connected nodes with PageRank:
```cypher
CALL gds.pageRank.stream('disease-graph')
YIELD nodeId, score
RETURN gds.util.asNode(nodeId).name AS name, score
ORDER BY score DESC, name ASC LIMIT 12;
```
Find number of drugs that treats each dieases:
```cypher
CALL gds.pageRank.stream('disease-graph')
YIELD nodeId, score
WITH COLLECT(DISTINCT(gds.util.asNode(nodeId))) AS di_list, score
ORDER BY score DESC 
MATCH (dr:drug) -[r:treats]-> (di:disease)
WHERE di in di_list
RETURN di.name as disease_name, score AS pagerank_score, COUNT(DISTINCT(r)) AS drug LIMIT 12;
// could also augment return line to include "COLLECT(dr.name) as drug_names" to get the names of all drugs
```
