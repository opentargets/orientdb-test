# OrientDB - Proof of Concept

This repo builds a dockerised instance of OrientDB containing platform data stored as a graph.

## Query benchmarks

The intention is to test some of the more complex queries used in the platform and benchmark them.

Some examples of such queries are:

- Get all genes that are associated with a specific disease, such that they have a RNA expression level above some threshold in one or more tissues
- Get the aggregated counts, per RNA expression level and per tissue, of all genes that are associated with a specific disease
- Get all genes that are associated with a specific disease or are associated with a child of the specific disease (ie. indirect associations)

Other queries that are not currently used in the platform, but lend themselves to a graph structure, might also be explored.

## Setup instructions

Clone this repo.

Download and prepare the data for OrientDB:

```
bash prepare-data.sh
```

Run OrientDB in a docker container (sharing the processed data):

```
docker run -v $PWD/data:/etl/data -v$PWD/loaders:/etl/loaders -e ORIENTDB_ROOT_PASSWORD=root -p 2424:2424 -p 2480:2480 -d --name otorient orientdb
```

Attach to the container and run the ETL steps (it is important that vertices are before edges):

```
docker exec -it otorient /bin/sh
$ cd bin
$ oetl.sh /etl/loaders/vertex-gene.json
$ oetl.sh /etl/loaders/vertex-efo.json
$ oetl.sh /etl/loaders/vertex-tissue.json
$ oetl.sh /etl/loaders/edge-efo-efo.json
$ oetl.sh /etl/loaders/edge-gene-efo.json
$ oetl.sh /etl/loaders/edge-gene-tissue.json
```

You should now be able to browse the database `opentargets` in OrientDB Studio at http://localhost:2480. Username and password are both `root`.

## Queries to try

```
MATCH {class:Gene, as: g, where: (symbol = "BRAF")}
	.inE("IsExpressedIn") {as: e, where: (rnaLevel > 4)}
    .outV()
    {as: t}
    RETURN g.symbol, t.name

MATCH {class:Gene, as: g}
	.inE("IsExpressedIn") {as: e, where: (rnaLevel > 7)}
    .outV()
    {as: t}
    RETURN g.symbol, t.name

MATCH {class:Disease, as: d, where: (id = "EFO_0003778")}
	.outE("IsAssociatedWith") {as: e, where: (score > 0)}
    .inV() {as: g}
    RETURN g.symbol, d.name

MATCH {class:Disease, as: d, where: (id = "EFO_0003778")}
	.outE("IsAssociatedWith") {as: d2g, where: (score > 0)}
    .inV() {as: g}
    .inE("IsExpressedIn") {as: g2t, where: (rnaLevel > 1)}
    .outV()
    {as: t}
    RETURN d.name, d2g.score, g.symbol, g2t.rnaLevel, t.name

MATCH {class:Disease, as: d, where: (id = "Orphanet_848")}
	.in("IsChildOf") {as: d2, while: ($depth < 2)}
	.outE("IsAssociatedWith") {as: e, where: (score > 0)}
    .inV() {as: g}
    RETURN DISTINCT g.symbol
```

## Using the console

As an alternative to running commands in OrientDB Studio, you can use the CLI (from `/orientdb/bin` directory). This also allows you to export the database.

```
$ console.sh
orientdb> connect plocal:/orientdb/databases/opentargets admin admin
orientdb {db=opentargets}> MATCH {class:Gene, as: g, where: (symbol = "BRAF")}.inE("IsExpressedIn") {as: e, where: (rnaLevel > 4)}.outV(){as: t} RETURN g.symbol, t.name
orientdb {db=opentargets}> EXPORT DATABASE /etl/data/opentargets.dump
```
