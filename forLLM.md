Description:
Create a intermediary for accessing shotgrid / FPT data.
Accessing shotgrid directly requires a secure handshake with cryptographic authentication. This can slow everything down.
By creating an intermediary, the local machines can talk using standard http and be faster. It can also listen to shotgrid and do cache invalidation.
Finally if you want to change DBs later, this is the only point of contact that needs to be updated to switch.

A:
We already have SG cache for this so we should upgrade/rewrite SG cache for this.
I have designed an approach for upgrading the SG cache without hosting a DB, while can easily migrate to one in the future.
I definitely need Fred’s thoughts on my design, and if we end up deciding to host a DB, I believe Fred can set up the DB pretty well.
But I want to implement this intermediary service myself.

B:
A caching service doesn’t have to have a DB, you could cache in memory. This would still be an upgrade from the existing disk-based shotgun_cache, because you can subscribe to the shotgrid events and use those for cache invalidation. As long as you only need one instance of the server, then you could basically stop there design wise.

If after benchmarking, the load on the service warrants 2 or more instances then you have 3 options to choose from:

Accept a 50% miss rate as each server has its own private cache
Add a shared key/value store such as memcached or redis to hold the cache values. This would allow each instance of the service to return a cached value for anything that has been previously requested
Introduce a shared database. This option is a little bit trickier than the key/value store option because you need to introduce a schema for your cached values. You probably only want to choose this option if you either have other data you want to store alongside the cached data or if you intend to cache more data than can fit in memory as generally key/value store need to keep at least the entire keyset in memory.

A:
Currently, we use an SG cache for shot data, e.g., /pipeline/tools/shotgun_cache/blitz.json. It works 99% of the time, so I’d prefer not to change it for now.
The SG cache I mentioned above is for publish data (Mega Publishes). Since we need relational searching here, I think Redis is not an option in this case?

A shared database(#3 option) seems most suitable for our “current” resources and needs. But we may have fewer resoures in the future, and given our company size, daily access times, a database server might be overkill?

Just in case, I’ve thought of 2 more plans that minimize or zero hosting requirements:

FastAPI service as a single writer for a local SQLite DB (WAL mode). Clients query publish data through this service. Use a separate SQLite or LMDB as a task queue for retries after failures. Create hourly snapshots on NFS, though I’m unsure about the snapshot format. Parquet might be better than SQLite for concurrent file reads on NFS. If the service goes down, clients can fall back to reading the snapshot. DuckDB can convert SQLite to Parquet and merge multiple files into one table when reading.
Use append-only log to simulate the single writer with Parquet files on NFS. Use DuckDB + Parquet for query. Split Parquet files into smaller chunks if they grow too large. Writes won’t be real-time, but should be acceptable.
To me the fastapi approach is not bad because it doesn’t require Podman or Docker knowledge, and it also offers portability, though I’m not sure how much portability actually matters.
The append-only log approach seems simplest to me, just need to make sure the log can handles the queue job well.
Does these make sense?

B:
I think you may have over thought the problem a bit. While the underlying data in shotgrid is relational, a cache need not model the data in the same fashion. For top level objects such as Projects, Shots, and Assets, it’s sufficient for the cache to grab all of them and answer the queries for object by id. For more complicated queries, from the cache’s point of view, you can treat that query as an opaque key and store the response from shotgrid without understanding the model. For both of those scenarios a FastAPI or Flask server in front of Redis is sufficient. The service can also register with the Shotgrid Events system to invalidate cached items when data is updated or inserted on shotgrid’s side.

As for sqlite as a storage backend, ideally you want to avoid any database reading and writing to NFS because a db is sensitive to latency and is very particular about file locks. Neither of those are NFS’s virtues to say the least.

An append only log is pretty overkill for a caching project, as you’ll end up paying a lot of storage for short lived data.

Anyway, ideally you want to keep things as simple as possible within the constraints of the problem.

A:
Thanks for the explanation, I didn’t know Redis could be used here. That would be great.

It’s no longer important, but I initially considered using Parquet for snapshots on NFS because it seems like it’s not an embedded DB and supports concurrent access over NFS and it can do SQL via DuckDB.
But I haven’t used Parquet before.

B:
Parquet is usually used for ETL or MapReduce workflows where you have many writers each crunching some numbers and building their own parquet files during the mapping phase of the calculation, then during the reduce phase, the parquet files will be queried against to compute the final summaries.