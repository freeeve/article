## 1 Getting started with Neo4j and Cypher

This tutorial will introduce the Neo4j graph database and the Cypher query language, while building an access control list (ACL) system. Fine-grained ACL systems that deal with membership and inherited permissions over hierarchies of groups are one of the pain points that you deal with in traditional SQL databases. It's also often a subsystem that can be pulled out of the main codebase, even if you use SQL or other datastores for everything else, so it's a good way to get started using a graph. We'll start with our ACL in an SQL database (postgres), and migrate it to Neo4j using Cypher.

This is by no means the only use case for picking a graph database. To get other ideas, see the [neo4j use case](http://www.neotechnology.com/neo4j-use-cases/) page, or the diverse use cases submitted as part of the [graph gist challenge](https://github.com/neo4j-contrib/graphgist/wiki).

### 1.1 Installing Neo4j for development

Head to [http://neo4j.com/download/](http://neo4j.com/download/) and click on the link to download. You'll need to have Java 7 installed as well. On Mac or Linux, untar the download to the folder of your choice, and then run `bin/neo4j start` from the folder where you put it. Windows users will receive an installer package, and you can run the service from the dashboard that starts up after installing. Once you're done, you should be able to visit [http://localhost:7474/](http://localhost:7474/) to test your server.

## 2 Fundamental building blocks of Neo4j

Neo4j is a graph database, adopting a labeled property graph model. In Neo4j terminology, vertices are called nodes, and edges are called relationships. 

### 2.1 Nodes

- \- Nodes are typically used to represent entities (or complex value types).

- \- Nodes can have properties, which are key/value pairs. Values can be primitives or collections of primitives.

- \- Nodes can have zero or more relationships connecting them to other nodes.

### 2.2 Relationships

- \- Relationships are used to represent the relationships between nodes; to provide context to the nodes. 

- \- Relationships must have a start and end node, thus relationships must have a direction. Direction can be ignored at query time, so the fact that direction is there does not mean it must be used. 

- \- Relationships must have a relationship type.

- \- Relationships can have properties (key/value pairs. values can be primitives or collections of primitives). 


### 2.3 Properties

- \- Nodes and relationships can have properties (key/value pairs. values can be primitives or collections of primitives.)

- \- Properties can quantify relationships.

### 2.4 Labels

- \- Nodes can have zero or more labels. 

- \- Labels can represent roles, categories or types.

- \- Labels are used to define indexes and constraints.

## 3 A quick Cypher introduction

Cypher is a pattern-oriented, declarative query language; a mix of SQL and graph traversal patterns. If you know SQL well, you'll probably quickly see the parallels. This is just a brief introduction to get you started &mdash; if you want more complete documentation, see the [refcard](http://docs.neo4j.org/refcard/2.1/) and the [manual](http://docs.neo4j.org/chunked/stable/cypher-query-lang.html). Note that much of Cypher is case-insensitive, like SQL. Notable exceptions to this rule include identifiers, labels, property keys, and relationship types.

### 3.1 Cypher patterns

Patterns are the fundamental traversal description of Cypher. Designed after ASCII art representing nodes as circles and relationships as arrows, came the `(ident)-->(ident2)` pattern, where `ident` and `ident2` are identifiers. Relationship identifiers are specified within square brackets, with an optional type after a colon, like `(u)-[r:HAS_ACCESS]->(a)`. Labels are specified similarly to relationship types, following a colon: `(u:User)-->(a:Asset)`. To be able to specify more complex patterns that don't fit in a straight line, you can comma separate patterns; the simplest of these complex patterns is a triangle:` (a)-->(b)-->(c), (c)-->(a)`. Patterns can be used not only while querying, but also while creating new nodes and relationships.

### 3.2 Cypher clauses

#### 3.2.1 MATCH / WHERE

Find a pattern in the graph, filtering out results with predicates in the `WHERE`. This query will search for nodes with relationships pointing to nodes that have no relationships:

<!--code lang=sql linenums=true-->

    MATCH (n)-->(m)
    WHERE NOT (m)-->()

#### 3.2.2 OPTIONAL MATCH / WHERE

Find a pattern in the graph using existing identifiers as starting points. The predicates in `WHERE` are only executed against new data matched in the `OPTIONAL MATCH`. This query will search for all nodes, then it will optionally find all of the connecting nodes and relationships. If there aren't any for a particular `n `with an` r.strength > 0.5`, it will return null for `r` and `m,` while still returning` n`:

<!--code lang=sql linenums=true-->

    MATCH (n)
    OPTIONAL MATCH (n)-[r]-(m)
    WHERE r.strength > 0.5
    RETURN n, r, m

#### 3.2.3 RETURN / ORDER BY / SKIP / LIMIT

`RETURN` is like SELECT in SQL. You list all of the expressions and fields you want to return, along with optional aliases. This returns the first three users sorted by name, and `3^2` (9):

<!--code lang=sql linenums=true-->

    RETURN 3^2 as nine, u.name as userName 
    ORDER BY userName
    LIMIT 3

#### 3.2.4 WITH / ORDER BY / SKIP / LIMIT

Similar to `RETURN`, except that the query can continue after the `WITH`. This is handy in a lot of situations where you need to do a query, and then use the results to do another query. This query gets all of the people and their friends, counts them, and then gets all of the friends of friends:

<!--code lang=sql linenums=true-->

    MATCH (n:Person)-[:FRIEND]-(f)
    WITH count(f) as c, n
    MATCH (n)-[:FRIEND]-()-[:FRIEND]-(fof)
    RETURN n, c, fof

#### 3.2.5 CREATE

`CREATE` will create all new parts in a pattern. This query will create a new `:Bar` node, and a `:FOO` relationship connecting out from the n node that has node id 1:

<!--code lang=sql linenums=true-->

    MATCH (n)
    WHERE id(n) = 1
    CREATE (n)-[:FOO]->(b:Bar)
    RETURN n, b

Note that, if we run this query more than once, we'll end up with more than one `:Bar` node connecting to our n node. Each time we'll get a new b returned out of the query.

#### 3.2.6 MERGE

`MERGE` will `MATCH` or `CREATE` (if it doesn't exist). 

<!--code lang=sql linenums=true-->

    MATCH (n)
    WHERE id(n) = 1
    MERGE (n)-[:FOO]->(b:Bar)
    RETURN n, b

If we run this query more than once, we'll end up with just one node connecting to our `n` node. This is because `MERGE` will match the pattern instead of creating it, once it is created, so the `b` node returned will always be the same.

### 3.3 Cypher expressions and scalar/collection functions

Cypher has a rich set of expressions for math, strings, collection, comparisons, etc., as you would expect from a query language. For example, you can get the length of a collection using the `length()` function, or compare equality using the `=` expression.

### 3.4 Cypher aggregation functions

Cypher has built-in aggregation functions like `count()`, `sum()`, `avg()`, `min()`, and more. When you use these, the rest of the results asked for are grouped implicitly. In SQL you would need to explicitly define which fields to GROUP BY; in Cypher, it groups them implicitly. 

This query finds the count of outbound relationships  for each node. If the `RETURN n, count(1)` were changed to be just `RETURN count(1)`, it would give the total count of outbound relationships for all nodes, because of the implicit grouping:

<!--code lang=sql linenums=true-->

    MATCH (n)-->(m)
    RETURN n, count(1)

### 3.5 Indexes and Constraints

The Neo4j 2.0-style indexes and constraints are defined with Cypher, and if you're going to be using `MERGE` to uniquely create nodes, then it is important to add a unique constraint on the unique field, especially if you have more than just a few to add. Ideally, if you are going to need to query on a particular property on a node, you will have an index or unique constraint on it, so that you can do a fast lookup rather than scanning the entire graph or labeled subset. 

Here are two examples. One creates a constraint (with a backing index), and the other an index:

<!--code lang=sql linenums=true-->

    CREATE CONSTRAINT on (u:User) ASSERT u.id IS UNIQUE;
    CREATE INDEX on :User(name);

## 4 What does ACL look like in SQL-land?

Armed with some Cypher, let's get back to the task at hand. You'll probably have users, groups, a many-to-many join table of user_groups, and then a many-to-many join table of group_groups, or something of the sort. You'll also have artifacts or assets, which we'll keep in a single level table for simplicity &mdash; although this sort of thing is often hierarchical as well, like a folder structure where the contents inherit the permissions of the folder &mdash; and refer to them by an id. 

Let's simplify it even further, and keep it down to a single permission indicating "has access," rather than multiple permissions for different roles. Here's a simplified ERD showing what we're going to start with.

![ACL ERD](http://i.imgur.com/id4AloN.png)

It doesn't look so bad, but consider the queries we'll need to run in order to decide whether a particular user has access to a particular asset. They might be directly connected via user_asset_access, or they might be in a group that has access via group_asset_access. Or they may even be in a group that's part of a larger group that has access via group_asset_access, and so on... 

Some SQL engines even have a hierarchical index optimization, to handle this a little better &mdash; these are still somewhat limited, though. The hierarchical nature of the model makes it ideal to use a graph database. 

Here's a quick script [(SQL Fiddle)](http://sqlfiddle.com/#!15/032bf) to create the schema in postgres, to follow along:

<!--code lang=sql linenums=true-->

    CREATE TABLE USERS (id SERIAL, name varchar(50));
    CREATE TABLE GROUPS (id SERIAL, name varchar(50));
    CREATE TABLE USER_GROUPS (user_id integer, group_id integer);
    CREATE TABLE GROUP_GROUPS (parent_group_id integer, group_id integer);
    CREATE TABLE USER_ASSET_ACCESS(user_id integer, asset_id integer);
    CREATE TABLE GROUP_ASSET_ACCESS(group_id integer, asset_id integer);
    CREATE TABLE ASSETS (id SERIAL, uri varchar(1000));

    INSERT INTO USERS (name) values('neo');
    INSERT INTO USERS (name) values('morpheus');
    INSERT INTO USERS (name) values('trinity');
    INSERT INTO USERS (name) values('cypher');
    INSERT INTO USERS (name) values('smith');

    INSERT INTO GROUPS (name) values('agents');
    INSERT INTO GROUPS (name) values('matrix');
    INSERT INTO GROUPS (name) values('crew');
    INSERT INTO GROUPS (name) values('the_one');

    INSERT INTO USER_GROUPS values(1, 3);
    INSERT INTO USER_GROUPS values(1, 4);
    INSERT INTO USER_GROUPS values(2, 3);
    INSERT INTO USER_GROUPS values(3, 3);
    INSERT INTO USER_GROUPS values(4, 3);
    INSERT INTO USER_GROUPS values(5, 1);

    INSERT INTO GROUP_GROUPS values(2, 1);

    INSERT INTO ASSETS (uri) values('/there/is/no/spoon');
    INSERT INTO ASSETS (uri) values('/the/red/pill');
    INSERT INTO ASSETS (uri) values('/mainframe');
    INSERT INTO ASSETS (uri) values('/deja/vu');

    INSERT INTO USER_ASSET_ACCESS values(1, 1);

    INSERT INTO GROUP_ASSET_ACCESS values(2, 3);
    INSERT INTO GROUP_ASSET_ACCESS values(3, 2);
    INSERT INTO GROUP_ASSET_ACCESS values(4, 2);

You end up writing long queries like this [(SQL Fiddle)](http://sqlfiddle.com/#!15/032bf/10) in order to determine whether someone has access to a particular asset. Here, if any counts returned are >0, they have access:

<!--code lang=sql linenums=true-->

    SELECT count(1)
    FROM users u, user_asset_access uaa, assets a
    WHERE u.id = uaa.user_id 
      AND uaa.asset_id = a.id
      AND a.uri = '/mainframe'
      AND u.name = 'smith'
    UNION ALL
    SELECT count(1)
    FROM users u, user_groups ug, groups g, group_asset_access gaa, assets a
    WHERE u.id = ug.user_id
      AND g.id = ug.group_id
      AND gaa.asset_id = a.id
      AND gaa.group_id = g.id
      AND a.uri = '/mainframe'
      AND u.name = 'smith'
    UNION ALL
    SELECT count(1)
    FROM users u, user_groups ug, groups g, groups pg, group_groups gg, group_asset_access gaa, assets a
    WHERE u.id = ug.user_id
      AND g.id = ug.group_id
      AND gg.parent_group_id = pg.id
      AND gg.group_id = g.id
      AND gaa.asset_id = a.id
      AND gaa.group_id = pg.id
      AND a.uri = '/mainframe'
      AND u.name = 'smith'

One of the downsides is that this query has a fixed depth that you need to continue to increase with more `UNION`s, depending on how deep your group hierarchy can be. If there's no theoretical limit, you'll just need to do it programmatically and create more than one query.

## 5 Migrating data to Neo4j

With the 2.1 addition of `LOAD CSV` in Cypher, this step got a lot easier for datasets that aren't too large. Note: If you're looking at importing many millions of records, you should probably check out the [batch-import](https://github.com/jexp/batch-import) tool, or build your own [BatchInserter](http://docs.neo4j.org/chunked/stable/javadocs/org/neo4j/unsafe/batchinsert/BatchInserter.html) from the unsafe package in the Java API of Neo4j. 

So, let's construct our export and import queries to load our data from a SQL database to Neo4j. In postgres, there is a convenient CSV export function called COPY:

<!--code lang=sql linenums=true-->

    wfreeman=# COPY users TO '/tmp/users.csv' WITH CSV;
    COPY 5
    wfreeman=# COPY assets TO '/tmp/assets.csv' WITH CSV;
    COPY 4
    wfreeman=# COPY groups TO '/tmp/groups.csv' WITH CSV;
    COPY 4
    wfreeman=# COPY user_groups TO '/tmp/user_groups.csv' WITH CSV;
    COPY 6
    wfreeman=# COPY group_groups TO '/tmp/group_groups.csv' WITH CSV;
    COPY 1
    wfreeman=# COPY group_asset_access TO '/tmp/group_asset_access.csv' WITH CSV;
    COPY 3
    wfreeman=# COPY user_asset_access TO '/tmp/user_asset_access.csv' WITH CSV;
    COPY 1

Most SQL databases provide a similar functionality, but the end result will be CSV-encoded dumps of the tables that we can use to direct the data into Neo4j. We'll need to use `LOAD CSV` along with the `MERGE` clause to bring the data in.

First we'll make some unique constraints (which come with indexes), as well as some indexes for our non-unique data:

<!--code lang=sql linenums=true-->

    CREATE CONSTRAINT ON (u:User) ASSERT u.id IS UNIQUE;
    CREATE CONSTRAINT ON (g:Group) ASSERT g.id IS UNIQUE;
    CREATE CONSTRAINT ON (a:Asset) ASSERT a.id IS UNIQUE;

    CREATE INDEX ON :User(name);
    CREATE INDEX ON :Group(name);
    CREATE INDEX ON :Asset(uri);

Let's start with the simple parts, individual entity creation. We'll use the ids as "unique identifiers" for the entities:

<!--code lang=sql linenums=true-->

    LOAD CSV FROM 'file:///tmp/users.csv' as line
    MERGE (:User {id:toInt(line[0]), name:line[1]});

    LOAD CSV FROM 'file:///tmp/groups.csv' as line
    MERGE (:Group {id:toInt(line[0]), name:line[1]});

    LOAD CSV FROM 'file:///tmp/assets.csv' as line
    MERGE (:Asset {id:toInt(line[0]), uri:line[1]});

Now we can connect the entities with relationships using the join table CSV exports:

<!--code lang=sql linenums=true-->

    LOAD CSV FROM 'file:///tmp/user_asset_access.csv' as line
    MATCH (u:User {id:toInt(line[0])}), (a:Asset {id:toInt(line[1])})
    MERGE (u)-[:HAS_ACCESS]->(a);

    LOAD CSV FROM 'file:///tmp/group_asset_access.csv' as line
    MATCH (g:Group {id:toInt(line[0])}), (a:Asset {id:toInt(line[1])})
    MERGE (g)-[:HAS_ACCESS]->(a);

    LOAD CSV FROM 'file:///tmp/user_groups.csv' as line
    MATCH (u:User {id:toInt(line[0])}), (g:Group {id:toInt(line[1])})
    MERGE (u)-[:IS_MEMBER]->(g);

    LOAD CSV FROM 'file:///tmp/group_groups.csv' as line
    MATCH (p:Group {id:toInt(line[0])}), (g:Group {id:toInt(line[1])})
    MERGE (g)-[:IS_MEMBER]->(p);

## 6 Running some queries

Once all of the data is imported, we can run some queries in the browser.

<!--code lang=sql linenums=true-->

    MATCH n RETURN n;

![image alt text](http://i.imgur.com/LXt6h4C.png)

Let's see whether Neo has access to `/the/red/pill`, using the `shortestPath` function to optimize the search (note the difference in complexity between this query and the SQL that accomplished a similar result):

<!--code lang=sql linenums=true-->

    MATCH shortestPath((neo:User {name:'neo'})-[:HAS_ACCESS|IS_MEMBER*]->(a:Asset {uri:'/the/red/pill'}))
    RETURN count(*) > 0 as hasAccess

![image alt text](http://i.imgur.com/LBNjDLC.png)

Who has access to `/the/red/pill`?

<!--code lang=sql linenums=true-->

    MATCH shortestPath((u:User)-[:HAS_ACCESS|IS_MEMBER*]->(a:Asset {uri:'/the/red/pill'}))
    RETURN u.name

![image alt text](http://i.imgur.com/yodolz5.png)

## 7 A closer look at our Cypher for ACL

### 7.1 Loading the data

<!--code lang=sql linenums=true-->

    LOAD CSV FROM 'file:///tmp/group_asset_access.csv' as line
    MATCH (g:Group {id:toInt(line[0])}), (a:Asset {id:toInt(line[1])})
    MERGE (g)-[:HAS_ACCESS]->(a);

`LOAD CSV` gives us a result for each line of the CSV, a collection of strings. In order to use those strings as integers (like the ids), we can use the conversion function `toInt`. Cypher will do an index lookup on the group id and the asset id in the `MATCH`, and then `MERGE` the `:HAS_ACCESS `relationship (create if it doesn't exist).

### 7.2 Checking for access

<!--code lang=sql linenums=true-->

    MATCH shortestPath((neo:User {name:'neo'})-[:HAS_ACCESS|IS_MEMBER*]->(a:Asset {uri:'/the/red/pill'}))
    RETURN count(*) > 0 as hasAccess

This query does a lot of things at once in the first `MATCH`. It can actually be broken out to make the lines shorter, if desired:

<!--code lang=sql linenums=true-->

    MATCH shortestPath((neo:User)-[:HAS_ACCESS|IS_MEMBER*]->(a:Asset))
    WHERE neo.name = 'neo' AND a.uri = '/the/red/pill'
    RETURN count(*) > 0 as hasAccess

The above two queries are exactly the same, and you can decide whether you prefer the inline property matching syntax, or the `WHERE` property matching syntax. Let's start by looking at the pattern and `WHERE` filters.

<!--code lang=sql linenums=true-->

    … (neo:User)-[:HAS_ACCESS|IS_MEMBER*]->(a:Asset) …
    WHERE neo.name = 'neo' AND a.uri = '/the/red/pill'

We have `:User(name)` and `:Asset(uri)` indexes, so Cypher can look those up quickly via their respective indexes.

Once it has those starting points, it will use the `shortestPath()` matcher to find a single path that matches, following relationships of either` :HAS_ACCESS` or `:IS_MEMBER` (which is enough to determine whether `Neo` has access to `/the/red/pill`).

### 7.3 Who has access

<!--code lang=sql linenums=true-->

    MATCH shortestPath((u:User)-[:HAS_ACCESS|IS_MEMBER*]->(a:Asset {uri:'/the/red/pill'}))
    RETURN u.name

In this case, the query is almost the same, except we start with the `:Asset(uri)` lookup, and then calculate the shortest path for all `:User`-labeled nodes, note that the `:User` has no name specified in the pattern. The names that are returned are the ones that have a path with either `:HAS_ACCESS` or `:IS_MEMBER`, so it's able to traverse the group hierarchy as well as the access hierarchy.

## 8 Conclusions

This is a simplified example of an ACL, but it should be apparent that the complexity of the queries, and the exact transfer of the data model to the graph structure are a great fit for Neo4j. There are many other [use cases](http://www.neotechnology.com/neo4j-use-cases/) &mdash; the graph isn't just for social data! 

If you have any more questions or need help getting started with Neo4j and Cypher, be sure to book me for an AirPair!
