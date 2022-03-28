# R2RML: A Step-by-step Tutorial

[Christophe Debruyne ](http://christophedebruyne.be/)  
---
[ADAPT](https://www.adaptcentre.ie/), [Trinity College Dublin](https://www.tcd.ie/)  
[WISE](https://wise.vub.ac.be/), [Vrije Universiteit Brussel](https://www.vub.be/)

# 1 Introduction

The goal of this tutorial is to familiarize the reader with some core concepts of [R2RML](https://www.w3.org/TR/r2rml/). The R2RML engine we will use is [R2RML-F](https://github.com/chrdebru/r2rml), though we will not avail of any functionality outside of R2RML's specification. I assume that the reader has downloaded or installed an R2RML engine.

While R2RML was intended for relational databases, R2RML-F allows one to access CSV files as relational tables. This tutorial will use this feature so that a relational database will not be required for this tutorial. We use the [H2 Database Engine](https://www.h2database.com/html/main.html) to access CSV files as tables. This means that column names are capitalized (i.e., `emp` becomes `EMP`).

The CSV file we will transform into RDF is [weatherstations.csv](./files/weatherstations.csv). This CSV file was published by [Dublinked.ie with a CC BY 4.0 license](https://data.gov.ie/dataset/weather-stations/resource/602fc03a-bfb3-4501-b56b-09fb1564c9ed). This file contains information on weather stations; their names, locations, the agency responsible for that stations, and a URL to a page with the weather readings of that station. The CSV file contains the following rows (with the first being the header):

| Name | Weather_Reading | Agency | LAT | LONG |
|--|--|--|--|--|
| M50 Blanchardstown | ...| National Roads Authority |53.3704660326011 | -6.38085144711153 |
| M50 Dublin Airport | ... | National Roads Authority | 53.4096411069945 | -6.22759742761812 |
| Dublin Airport | ... | Met Éireann | 53.4215060785623 | -6.29784754004026 |

Note that the column "Weather_Reading" contains URLs in the file. We have omitted those for brevity. We will transform the contents of this CSV file into RDF using the [GeoSPARQL](https://www.opengeospatial.org/standards/geosparql). In GeoSPARQL, there is a distinction between features and geometries. Features are the "things" that one can represent on a map, and geometries are the way those things are represented using points, lines, polygons, etc. For each record in our CSV file, we thus have information on weather stations (the feature) and the longitude and latitude that constitute a point on a map (the geometry). Features and geometries are connected using the `geo:hasGeometry` predicate.

Before we start, we will first create the configuration file for our R2RML-F engine. The contents of that file, which we call `weather-config.properties` is as follows:

```
CSVFiles = weatherstations.csv
mappingFile = ./weather-mapping.ttl
outputFile = ./weather-output.ttl
format = TURTLE
```

This informs the engine that `weatherstations.csv` will be transformed into RDF using the mapping provided by `./weather-mapping.ttl`. The RDF will be formatted into TURTLE and written to `weather-output.ttl`.

We now create `weather-mapping.ttl` and add the following prefixes:

```turtle
@prefix rr: <http://www.w3.org/ns/r2rml#> .
@prefix fcc: <http://www.example.org/ont#> .
@prefix geo: <http://www.opengis.net/ont/geosparql#> .
@prefix rdfs: <http://www.w3.org/2000/01/rdf-schema#> .
@prefix geo2: <http://www.w3.org/2003/01/geo/wgs84_pos#> .
@prefix xsd: <http://www.w3.org/2001/XMLSchema#> .
```

The namespace `geo` is used for GeoSPARQL. The names spaces `rr`, `rdfs`, and `xsd` refer to the namespaces for R2RML, RDFS, and XSD respectively. We will use `geo2` for publishing the longitude and latitude in another vocabulary (next to GeoSPARQL). We furthermore assume the existence of an ontology prefixed with `fcc` for some concepts and relations.

# 2 A Triples Map for Weather Stations

## 2.1 Generating Resources for Weather Stations

We first create the triples map for weather stations (the feature). Every triples map needs a logical table and a subject map. The subject map is responsible for creating the subjects and any type declarations.

```turtle
# PREFIXES APPEAR HERE

<#WeatherStation>
  a rr:TriplesMap ;

  rr:logicalTable [ rr:tableName "WEATHERSTATIONS" ] ;

  rr:subjectMap [
    rr:template "http://data.example.org/ws/{NAME}" ;
    rr:class geo:Feature ;
    rr:class fcc:WeatherStation ;
  ] ;

  # PREDICATE OBJECT MAPS FOR <#WeatherStation> WILL APPEAR HERE
  .
```

For every row in the table, we use `NAME` to create the URI of the subject. We also declare that each subject is an instance of `geo:Feature` and `fcc:WeatherStation`.

We now execute the mapping with:

`$ java -jar r2rml/r2rml.jar weather-config.properties`

And six triples should be generated:

```turtle
<http://data.example.org/ws/Dublin%20Airport>
        a       <http://www.opengis.net/ont/geosparql#Feature> , <http://www.example.org/ont#WeatherStation> .

<http://data.example.org/ws/M50%20Dublin%20Airport>
        a       <http://www.opengis.net/ont/geosparql#Feature> , <http://www.example.org/ont#WeatherStation> .

<http://data.example.org/ws/M50%20Blanchardstown>
        a       <http://www.opengis.net/ont/geosparql#Feature> , <http://www.example.org/ont#WeatherStation> .
```

We will not provide the whole output for each step in this tutorial, but we will provide a snippet of the expected output instead.

## 2.2 Generating Labels for Weather Stations

We will now provide labels for Weather Stations. We know those labels are in English, so we can use that column for both the default label (i.e., with no language tag) and English labels. The predicate we will use is `rdfs:label`. Since we are going to use the same predicate for both labels, we only need to declare one Predicate Object Map with one predicate (for `rdfs:label`) and two object maps (one for the default label and one for the English label).

```turtle
  rr:predicateObjectMap [
    rr:predicate rdfs:label ;
    rr:objectMap [ rr:column "NAME" ] ;
    rr:objectMap [ rr:column "NAME" ; rr:language "en" ] ;
  ] ;
```

The resulting triples for one of the weather stations are as follows:

```turtle
<http://data.example.org/ws/Dublin%20Airport>
        <http://www.w3.org/2000/01/rdf-schema#label>
                "Dublin Airport" , "Dublin Airport"@en .
```

## 2.3 Generating the Coordinates with `geo2`

Now we will use the `geo2` namespace to publish the longitude and the latitude. They need to be published as `xsd:double`. If we do not provide any instructions, R2RML prescribes that literals should be generated. We extend the mapping with two predicate object maps.

```turtle
  rr:predicateObjectMap [
    rr:predicate geo2:lat ;
    rr:objectMap [ rr:column "LAT" ; rr:datatype xsd:double ] ;
  ] ;

  rr:predicateObjectMap [
    rr:predicate geo2:long ;
    rr:objectMap [ rr:column "LONG" ; rr:datatype xsd:double ] ;
  ] ;
```

These predicate object maps result in:

```turtle
<http://data.example.org/ws/Dublin%20Airport>
        <http://www.w3.org/2003/01/geo/wgs84_pos#lat>
                "53.4215060785623"^^<http://www.w3.org/2001/XMLSchema#double> ;
        <http://www.w3.org/2003/01/geo/wgs84_pos#long>
                "-6.29784754004026"^^<http://www.w3.org/2001/XMLSchema#double> .
```

You will notice that the engine will generate literals if you remove the `rr:datatype xsd:double` statements. I encourage you to try that.

## 2.4 Providing Weather Readings

We will now provide the weather readings (the URLs) as a resource with both `rdfs:seeAlso` and `fcc:withWeatherReading`. We can thus use one predicate object map with two predicates and one object map. The column `WEATHER_READING` contains a URL, but R2RML states that term maps with a `rr:column` generate literals. If we want to generate resources, however, we need to declare that in the mapping. The mapping looks as follows:

```turtle
  rr:predicateObjectMap [
  rr:predicate rdfs:seeAlso, fcc:withWeatherReading ;
    rr:objectMap [ rr:column "WEATHER_READING" ; rr:termType rr:IRI ] ;
  ] ;
```

This mapping results in:

```turtle
<http://data.example.org/ws/Dublin%20Airport>
        <http://www.w3.org/2000/01/rdf-schema#seeAlso>
                <http://www.met.ie/latest/reports.asp> ;
        <http://www.example.org/ont#withWeatherReading>
                <http://www.met.ie/latest/reports.asp> .
```

I encourage you to remove the `rr:termType rr:IRI` statement and to see what happens.

# 3 A Triples Map for Geometries

Before we can generate the relationships between features and geometries, we need to generate the geometries. In this section, we will create a triples map for geometries. In the next section, we will relate the two.

The triples map for geometries is pretty straightforward. It uses the same table and has a subject map that generates URIs for each geometry. The URIs are different as features are different from geometries. Each geometry is also declared to be an instance of `geo:Geometry`. Both longitude and latitude are used for the URI.

We have one predicate object map for the generation of [WKT](https://en.wikipedia.org/wiki/Well-known_text_representation_of_geometry) literals. Both longitude and latitude are used to fill in a template. R2RML prescribes that templates are used to generate IRIs. Here, however, we have to generate a literal (typed as `geo:wktLiteral`). The `rr:datatype` declaration will inform the engine that literals have to be produced as only literals can be data-typed.

```turtle
<#Geometries>
  a rr:TriplesMap ;

  rr:logicalTable [ rr:tableName "WEATHERSTATIONS" ] ;

  rr:subjectMap [
    rr:template "{NAME}" ;
    rr:class geo:Geometry
  ] ;

  rr:predicateObjectMap [
    rr:predicate geo:asWKT ;
      rr:objectMap [
        rr:template "POINT({LONG} {LAT})" ;
        rr:datatype geo:wktLiteral
      ] ;
  ] ;
  .
```

Geometries then look as follows:

```turtle
<http://data.example.org/geom/-6.22759742761812/53.4096411069945>
        a       <http://www.opengis.net/ont/geosparql#Geometry> ;
        <http://www.opengis.net/ont/geosparql#asWKT>
                "POINT(-6.22759742761812 53.4096411069945)"^^<http://www.opengis.net/ont/geosparql#wktLiteral> .

<http://data.example.org/geom/-6.29784754004026/53.4215060785623>
        a       <http://www.opengis.net/ont/geosparql#Geometry> ;
        <http://www.opengis.net/ont/geosparql#asWKT>
                "POINT(-6.29784754004026 53.4215060785623)"^^<http://www.opengis.net/ont/geosparql#wktLiteral> .

<http://data.example.org/geom/-6.38085144711153/53.3704660326011>
        a       <http://www.opengis.net/ont/geosparql#Geometry> ;
        <http://www.opengis.net/ont/geosparql#asWKT>
                "POINT(-6.38085144711153 53.3704660326011)"^^<http://www.opengis.net/ont/geosparql#wktLiteral> .
```

# 4 Tying it all together

Now that we have a triples map for geometries, we can create a predicate object map that relates features and geometries. It is true that in this simple case we can avail of a "simple" predicate object map, but we will now illustrate a predicate object map referring to another triples map with join conditions. The R2RML engine will thus create triples based on an SQL join. We extend the first triples map as follows:

```turtle
  rr:predicateObjectMap [
    rr:predicate geo:hasGeometry;
    rr:objectMap [
      rr:parentTriplesMap <#Geometries> ;
      rr:joinCondition [
        rr:child "NAME" ;
        rr:parent "NAME" ;
      ] ;
    ] ;
  ]
```

The two logical tables are joined using the column `NAME` resulting in:

```turtle
<http://data.example.org/ws/Dublin%20Airport>
        <http://www.opengis.net/ont/geosparql#hasGeometry>
                <http://data.example.org/geom/-6.29784754004026/53.4215060785623> .
```

# 5 Separating Longitude and Latitude in a Different Graph
*Note: the "solution" of this section are to be found in the files `weather-mapping-graph.ttl` and `weather-config-graph.properties`.*

Assuming we want to keep the statements using `geo2` in a separate named graph. This means we have to use an RDF serialization that supports graphs. Both predicate object maps and subject maps can be graph statements. Comprehending which triples end up in which graphs can be tricky, but we recommend reading R2RML's algorithm. In our case, separating those triples is straightforward.

First, we need to change our configuration file so that another serialization format is used. We also change the file extension of the output file to ensure that best practices are complied with. We will use [TriG](https://www.w3.org/TR/trig/) in this tutorial, which is an extension of TURTLE.

```
CSVFiles = weatherstations.csv
mappingFile = ./weather-mapping.ttl
outputFile = ./weather-output.trig
format = TRIG
```

Then we change the two predicate object maps of our weather triples map by adding a `rr:graph` statements.

```turtle
  rr:predicateObjectMap [
    rr:graph <http://data.example.org/graph/geo> ;
    rr:predicate geo2:lat ;
    rr:objectMap [
      rr:column "LAT" ;
      rr:datatype xsd:double ;
    ] ;
  ] ;

  rr:predicateObjectMap [
    rr:graph <http://data.example.org/graph/geo> ;
    rr:predicate geo2:long ;
    rr:objectMap [
      rr:column "LONG" ;
      rr:datatype xsd:double ;
    ] ;   
  ] ;
```

R2RML states that if the graph maps of both the subject map and the predicate object map are empty, then triples are written to the default graph; otherwise the triples are written to the union of both graph maps. Since the subject map has no graph maps (i.e., {}), the union is {} U {`http://data.example.org/graph/geo`}. In other words, those triples will appear in the named graph http://data.example.org/graph/geo and not in the default graph:

```turtle
# TRIPLES IN DEFAULT GRAPH OMITTED

<http://data.example.org/graph/geo> {
    <http://data.example.org/ws/Dublin%20Airport>
            <http://www.w3.org/2003/01/geo/wgs84_pos#lat>
                    "53.4215060785623"^^<http://www.w3.org/2001/XMLSchema#double> ;
            <http://www.w3.org/2003/01/geo/wgs84_pos#long>
                    "-6.29784754004026"^^<http://www.w3.org/2001/XMLSchema#double> .

    <http://data.example.org/ws/M50%20Dublin%20Airport>
            <http://www.w3.org/2003/01/geo/wgs84_pos#lat>
                    "53.4096411069945"^^<http://www.w3.org/2001/XMLSchema#double> ;
            <http://www.w3.org/2003/01/geo/wgs84_pos#long>
                    "-6.22759742761812"^^<http://www.w3.org/2001/XMLSchema#double> .

    <http://data.example.org/ws/M50%20Blanchardstown>
            <http://www.w3.org/2003/01/geo/wgs84_pos#lat>
                    "53.3704660326011"^^<http://www.w3.org/2001/XMLSchema#double> ;
            <http://www.w3.org/2003/01/geo/wgs84_pos#long>
                    "-6.38085144711153"^^<http://www.w3.org/2001/XMLSchema#double> .
}
```

# 6 Generating Geometries as Blank Nodes
*Note: the "solution" of this section are to be found in the files `weather-mapping-blank.ttl` and `weather-config-blank.properties`.*

Before we continue, it is important to note that blank nodes with the same identifier in different graphs are not considered representing the same resource. In other words, given a blank node with identifier *x* in *graph1* and a blank node with the same identifier *x* in a different *graph2*, then those two blank nodes do not represent the same thing. If you want to relate nodes across graphs, then you have to use IRIs (or avail of ontology axioms or rules to infer sameness).

Let's assume that we do not want to "publish" our geometries as resources with an IRI, but as blank nodes. In R2RML, this is easily achieved by stating that the `rr:termType` of the subject map is `rr:BlankNode`. We will need something that generates values to determine when the term map generates the same blank node. The values generated by the subject map are used for internal blank node identifiers. In R2RML-F, these values are kept in a dictionary. While the original template did generate unique URI strings, this is not necessary for blank nodes and we could choose a simpler representation (e.g., `{LONG}-{LAT}`).

We amend the subject map of the geometries triples map as follows:

```turtle
  rr:subjectMap [
    rr:template "{LONG}-{LAT}" ;
    rr:class geo:Geometry ;
    rr:termType rr:BlankNode ;
  ] ;
```

After running the R2RML engine, we see that it generates the following triples:

```turtle
<http://data.example.org/ws/Dublin%20Airport>
        <http://www.opengis.net/ont/geosparql#hasGeometry>
                [ a       <http://www.opengis.net/ont/geosparql#Geometry> ;
                  <http://www.opengis.net/ont/geosparql#asWKT>
                          "POINT(-6.29784754004026 53.4215060785623)"^^<http://www.opengis.net/ont/geosparql#wktLiteral>
                ] .
```

I encourage the reader to add a row to the CSV file with different values for `Name`, `Weather_Reading`, and `Agency`, but the values of `LAT` and `LONG` of one of the previous rows. Even though it does not make sense in the "real world", the reader will notice that two records will generate one blank node and all statements are merged. In other words, two weather stations will point to the same blank node. If the reader were to add a fifth row by copying the first and choosing different values for `LAT` and `LONG`, the reader will notice that this weather station will refer to two different geometries.

# 7 Some Final Considerations

In this tutorial, we covered a common occurrence in databases; the "grouping" of several concepts in a table. The city of an address, for instance, can appear as a column in an Address table. With R2RML, we can create mappings for projections.

# 8 Conclusions
With this step-by-step tutorial, I hope the reader has a better understanding of R2RML. I believe we have covered most of the R2RML W3C Recommendation and that the reader has everything they need to create and execute their own mapping. Feel free to contact me with feedback.

# License
This work is licensed under a [Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International License](https://creativecommons.org/licenses/by-nc-sa/4.0/).
