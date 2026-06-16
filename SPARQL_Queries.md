# SPARQL Queries for Railway Flood-Twin Ontology

These 8 queries demonstrate how to extract data from the `RailwayFloodTwin_with_individuals.owl` ontology. You can test these in Stardog Free, Apache Jena Fuseki, or the Protégé SPARQL plugin.

### Prefixes
Run these prefixes before executing the queries below:
```sparql
PREFIX rdf: <http://www.w3.org/1999/02/22-rdf-syntax-ns#>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
PREFIX rft: <https://w3id.org/def/RailwayFloodTwin#>
PREFIX xsd: <http://www.w3.org/2001/XMLSchema#>
```

---

### 1. Retrieve all Infrastructure Assets and their Specific Types
Returns a list of all assets and what specific class they belong to (e.g., TrackSegment, Culvert, Embankment).

```sparql
SELECT ?assetId ?assetType
WHERE {
  ?asset rdf:type ?type .
  ?asset rft:assetId ?assetId .
  ?type rdfs:label ?assetType .
  
  # Filter out the owl:NamedIndividual type to get the specific class
  FILTER (?type != owl:NamedIndividual)
}
ORDER BY ?assetId
```

---

### 2. Find Risk Thresholds for a Specific Asset (e.g., "Buse_0")
Gets the YELLOW, ORANGE, and RED elevation thresholds in metres NGF for a specific asset.

```sparql
SELECT ?assetId ?yellowZ ?orangeZ ?redZ
WHERE {
  ?asset rft:assetId "Buse_0" .
  ?asset rft:hasRiskThreshold ?threshold .
  ?threshold rft:yellowZ ?yellowZ .
  ?threshold rft:orangeZ ?orangeZ .
  ?threshold rft:redZ ?redZ .
}
```

---

### 3. Find Assets Adjacent to "Voie_0"
Finds all embankments, ditches, and culverts that are structurally adjacent to track segment "Voie_0".

```sparql
SELECT ?adjacentAssetId
WHERE {
  ?voie rft:assetId "Voie_0" .
  
  {
    ?adjacentAsset rft:isAdjacentTo ?voie .
  } UNION {
    ?adjacentAsset rft:adjacentToTrack ?voie .
  }
  
  ?adjacentAsset rft:assetId ?adjacentAssetId .
}
```

---

### 4. Find the Current RAMS Alert Verdicts
Uses the n-ary `AlertVerdict` relation to find the current operational directive for any assets currently under a warning.

```sparql
SELECT ?assetId ?wseValue ?statusName ?directiveName
WHERE {
  ?verdict rdf:type rft:AlertVerdict .
  
  ?verdict rft:verdictForAsset ?asset .
  ?asset rft:assetId ?assetId .
  
  ?verdict rft:verdictHasWSE ?wse .
  ?wse rft:wseValue ?wseValue .
  
  ?verdict rft:verdictHasStatus ?status .
  ?status rdfs:label ?statusName .
  
  ?verdict rft:verdictHasDirective ?directive .
  ?directive rdfs:label ?directiveName .
}
```

---

### 5. Hydrology: Retrieve the SWI and Runoff Coefficient for a Timestep
Finds the SWI value, its source rainfall intensity, and the resulting runoff coefficient.

```sparql
SELECT ?swiValue ?intensity ?runoffValue
WHERE {
  ?swi rdf:type rft:SoilWaterIndex .
  ?swi rft:swiValue ?swiValue .
  
  ?swi rft:computedFromRainfall ?rain .
  ?rain rft:intensityMmH ?intensity .
  
  ?swi rft:derivesRunoff ?runoff .
  ?runoff rft:runoffValue ?runoffValue .
}
```

---

### 6. Count Assets by Type
Groups the assets by their specific class type and counts how many of each exist in the sample data.

```sparql
SELECT ?assetType (COUNT(?asset) AS ?count)
WHERE {
  ?asset rdf:type ?type .
  ?type rdfs:label ?assetType .
  
  # Only match subclasses of InfrastructureAsset
  ?type rdfs:subClassOf* rft:InfrastructureAsset .
  FILTER (?type != owl:NamedIndividual && ?type != rft:InfrastructureAsset)
}
GROUP BY ?assetType
ORDER BY DESC(?count)
```

---

### 7. Find Sensor Observations
Retrieves all sensors and the timestamps of their observations.

```sparql
SELECT ?sensorName ?intensity ?timestamp
WHERE {
  ?sensor rdf:type rft:RainGauge .
  # Get local name from URI for display
  BIND(REPLACE(STR(?sensor), "^.*#", "") AS ?sensorName)
  
  ?sensor rft:madeObservation ?obs .
  ?obs rft:intensityMmH ?intensity .
  ?obs rft:timestamp ?timestamp .
}
```

---

### 8. Find Drainage Assets at Risk of Overflow (WSE > YELLOW threshold)
Checks if the water surface elevation (WSE) exceeds the YELLOW drainage capacity threshold for any culvert or ditch.

```sparql
SELECT ?assetId ?wseValue ?yellowZ
WHERE {
  # Find the WSE
  ?wse rdf:type rft:WaterSurfaceElevation .
  ?wse rft:wseValue ?wseValue .
  
  # Find the threshold
  ?verdict rft:verdictHasWSE ?wse .
  ?verdict rft:verdictForAsset ?asset .
  ?asset rft:assetId ?assetId .
  
  ?asset rft:hasRiskThreshold ?threshold .
  ?threshold rft:yellowZ ?yellowZ .
  
  # Compare
  FILTER (?wseValue > ?yellowZ)
}
