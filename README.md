**Forked of https://github.com/beekeeper-studio/beekeeper-studio**

#### Problem description: 
https://github.com/beekeeper-studio/beekeeper-studio/issues/263

Basically running 3 select * from bigtable queries in different QueryTabs crashes the application running out of memory. I was disappointed.
I tried to fix my particular problem: MySQL and CSV. BROKE other stuff by accident and force, because I didn't really care about other databases/formats/INSERTS/UPDATES and so on.


#### Solution proposal 0: File based storage (maintainers solution)

- download the results to a File
- page through file

Pros:
- Easy
- Downloadable straight away

Cons:
- Can't sort columns without redownload
- bigtable results in 400mb per query alternation
- downloading takes ages


#### Solution proposal 1: Local pagination in Tabulator

- add pagination: "local" to Tabulator 

Pros: 
- Easy
- Seems to work for a while
Cons:
- Crashes nevertheless
- Download has to be rewritten

#### Solution proposal 2: Fake remote pagination over the query results

- hold query result in memory
- page through array columns
- display only about 1000 rows simuntaneously

Pros:
- Easy enough
- Works until multiple tabs are opened
Cons:
- All results are still in memory
- Crashes with multiple tabs
- Download has to be rewritten

#### Solution proposal 3: Real remote pagination

- rewrite the query and apply limit and offset
- send query to server

Pros:
- Medium effort
- Runs pretty stable
- Memory gets released
- Blazing fast
- Works with 800000 rows for me

Cons:
- Need to be careful with original query syntax 
- ex: what if the original query is SELECT * FROM bigtable LIMIT 10 <- applying another limit with pagination will result in an syntax error
- Would have to parse the query to get this right
- Parsers seem immature
- Download has to be rewritten

#### Solution proposal 4: Real remote pagination with wrapped queries (that's what's in the package)

- offload work to the server
- don't touch original query
- wrap queries
- why wrappers? distinct does funny things, FOUND_ROWS is getting deprecated and we want to circumwent a query parser
- get count of results first: wrap into count(*)
SELECT count(*) FROM ( originalquery ) countres
- apply pagination / ordering
SELECT * FROM ( originalquery ) limitres ORDER BY order LIMIT limit OFFSET offset

Pros:
- Offloads work server side
- Runs pretty stable
- Memory gets released
- Medium fast
- Works with 800000 rows for me
- Column sorting works quite well

Cons:
- Can still have pitfalls with the syntax (didn't test much besides simple selects)
- Server load because of wrapped queries
- Growing latency with growing table size
- Download has to be rewritten
- Is somewhat inefficient on the server side, induces latency
- Download has to be rewritten

#### Sidenotes:

- To not store all results in memory on download again, I resorted to the original mysql2 package to support streaming the query asynchronously to a FileStreamWriter (sqlectron doesn't support this out of the box afaik). 
On big tables, this might take some time. You get notified by noty when it's ready meanwhile you can use the interface without being locked out.
