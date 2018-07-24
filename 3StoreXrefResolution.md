# Triplestore-backed cross-reference generation and indexing

## What and why?

For decades, catalogers have created and maintained graphs describing
relationships among representations of identical and related concepts,
in the form of name and subject authority files. In 1977 for name headings (NACO)
and 1983 for subject headings (SACO), disparate authority control efforts
were unified at national scope, in an early example of distributed metadata
maintenance. 

The normalized canonical headings maintained in authority control files (and the
rich access and context that these headings provide) constitute one of the most
valuable aspects of library catalog records. A naive approach to indexing these
headings (treating each heading simply as an ordered list of independent terms)
forfeits significant opportunities to leverage heading cross-references:
1. as "synonyms" (for the purpose of search/relevance)
2. as a non-invasive compatibility layer over inconsistent metadata
3. as intuitive, easily-navigable, user-controlled intersecting domain filters

Lucene/Solr (the de facto standard backend search stack) provides out-of-the-box
support (and/or near-support) for integrating enhanced heading metadata in useful
ways. This fact makes the opportunities missed by naive indexing of name and
subject headings all the more unfortunate.

## High-level approach

In order to better leverage name and subject headings in library metadata,
Penn embarked on a project to fully integrate name and subject authority
cross-reference information in our Solr-backed discovery interface.

The goal is not to use links to enhance the *results* of user queries (e.g.,
by linking from search results to external sources of information). Rather, the
goal is to use links to directly enhance the behavior of user queries, by building
linking information directly into the index (something akin to an advanced
index-time synonym filter backed by an arbitrarily large and complex thesaurus).

Authority cross-reference linking information is published by the Library of
Congress as dumps of RDF triples. One of the key benefits of this linked-data
approach is to support flexibility of data-transfer and schema. For extensibility,
flexibility, and forward compatibility, Penn has chosen to load these triples
into a triplestore (Apache Jena), and use them by natively querying (via SPARQL)
the graph that they represent.

## Triplestore as a backend service

In keeping with what we see as evolving recommended common practice for working
with linked data, the triplestore is not exposed as an endpoint open to arbitrary
user queries. Rather, arbitrary user queries are handled by a more opinionated
system specifically designed to efficiently handle such queries (in our case,
Solr/Lucene), and the triplestore is used as a backend service by that system.
There are a number of reasons to prefer this type of indirect triplestore use,
including:
1. The open-ended flexibility supported by arbitrary SPARQL is difficult to 
incorporate into an intuitive UI, and
2. Arbitrary SPARQL query access has the potential to be inefficient and costly in
terms of system resources (and user time, as measured by query response latency).

By using the triplestore on the backend, we treat linked data as a flexibile
compatibility layer, bridging the gap between systems (NACO/SACO authority files and
local discovery interface) that are designed to efficiently perform distinct tasks.

## Efficiency still matters!

Despite the fact that users don't query the triplestore in realtime, efficiency
is still important! For a full reindex of ~9 million records to complete in a
reasonable amount of time, we require that tens of millions of headings be
queried efficiently within the span of several hours. In addition to jumping
through a few hoops to accomodate Jena's somewhat-poorly-documented
idiosyncracies and expectations, some fundamental architectural decisions
have contributed to the performance/efficiency of Penn's system:

### Caching
A weighted caching strategy is employed that takes into account three factors:
1. last use of a cache entry
2. frequency of use of a cache entry, and
3. the relative cost of the underlying operation to derive the cached value

This weighted caching is particularly important over Jena, because certain queries
have proven to be consistently very fast, and others consistently very slow. The wide
disparity in response time is hard to explain or predict, and appears to be due
to low-level implementation details in Jena (or even peculiar to a particular
serialization of on-disk Jena data structures). A naive caching approach considering
only the first two factors (LRU and/or LFU) could easily (and initially did!) end up
pathologically purging the highest-latency cache entries. This is a problem when
some individual queries take up to 4 seconds to complete!

### Service-based architecture
To support better caching and deployment in a distributed environment, the
triplestore lives behind a webservice that manages a shared cache, and handles
lookup requests from clients over full-duplex, non-blocking, long-lived websocket
connections. Clients (which connect to the webservice) are configured per-node
in a distributed SolrCloud deployment; this enables Solr text analysis components
to invoke the lookup service RPC-style, as an abstraction directly in code.

### Parallelization of lookups at index-time
The lookup service is capable of being high-throughput, but even with caching
it is relatively high latency (especially since the actual lookup is done by a
service over the network). Solr indexing threads normally analyze fields
serially, but a serial approach is incompatible with the high latency associated
with any index-time external lookup. Some slight modifications to Solr update
request handling now support parallelization by field and document. As a result,
despite the high-latency for processing individual headings, the throughput of the
cross-reference generation component (for reasonably large update batches) is
high enough that the component has little if any effect on indexing speed.
