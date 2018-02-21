# Indexing/integration of records from multiple sources

Query-time deduplication is natively supported in Solr, and greatly increases the
flexibilty of the index without negative performance implications. 

To perform query-time deduplication, 

1. Choose a cluster id field (upon which to base deduplication). OCLC ids generally
work well for this purpose, though other schemes could absolutely be used.

2. If using a distributed Solr deployment, set the `uniqueKey` field to be a "!"-delimited
concatenation of the cluster id and the unique id for a given record. e.g., for a 
record with cluster id `1234` and unique id `56789`, the `uniqueKey` field (say,
`routing_id`) would be `1234!56789`. Used in conjunction with Solr's `compositeId` router,
this would cause all records with the same cluster id (prefix) to be routed to the same
node in the SolrCloud. This is essential for deduplication to work properly. Record the
"source" of each record in a field, to allow for prioritization of metadata sources.

3. Deduplicate the search domain at query time with [Solr "join"](https://wiki.apache.org/solr/Join)
filter queries. (to search over the full domain and deduplicate the *results*, one could use the 
[CollapsingQParser plugin](https://cwiki.apache.org/confluence/display/solr/Collapse+and+Expand+Results)).

4. Use `join` filter queries to define the search domain by defining an order of preference
for records in the same cluster, but different "record sources". Different `join` filter
queries can be used over the same index to define different orders of precedence for record
sources. e.g.:
```
fq=NOT({!join from=cluster_id to=cluster_id v=‘source:Penn’} AND source:(LC OR Hathi OR CRL))
AND NOT({!join from=cluster_id to=cluster_id v=‘source:LC’} AND source:(Hathi OR CRL))
AND NOT({!join from=cluster_id to=cluster_id v=‘source:Hathi’} AND source:CRL)
```

5. Facets that incorporate information from records excluded by the deduplication filter
query (e.g., access facets, location facets, record source facets) must be (re-)written
as `facet.query`s instead of `facet.field`s. 
```
facet.query={!join from=cluster_id to=cluster_id v=‘access:Online’}
facet.query={!join from=cluster_id to=cluster_id v=‘access:\’At the library\’’}
```

6. The `join` queries are fairly expensive, but are cached to great effect, and thus do not
adversely affect user queries. The one caveat there is that you *must* ensure that
any `join` queries that you plan to invoke are included among your `newSearcher` warming 
queries. `filterCache` is fairly granular, which is good because it means that various
atomic queries are calculated once and may be recombined efficiently via bitset intersection
(`BitDocSet`), but it also means that your `filterCache` must be sized to comfortably fit
all atomic queries that you expect to be used in the wild (plus room for normal `fq` params,
all values for `enum` method facets, etc. The `filterCache` is crucial!

7. Once a (deduplicated) window of results is determined, use the Solr `ExpandComponent` 
to return other records clustered with a particular result document. The `expand` component
is usually mentioned in conjunction with the `CollapsingQParser`, but it works just fine (and
is very useful) in a context with `join`-based domain deduplication (`expand.q=*:*`). N.b.,
pending resolution of [SOLR-7798](https://issues.apache.org/jira/browse/SOLR-7798), use of 
`ExpandComponent` in this manner will require special care (e.g., application of one of the
patches mentioned in that issue). As of time of writing, for solr v6.6.0, the
[perSegFacetCache branch of the upenn-libraries/solrplugins project](https://github.com/upenn-libraries/solrplugins/tree/perSegFacetCache) incorporates a patched version of `ExpandComponent`.
