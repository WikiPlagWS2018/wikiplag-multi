# Wikipedia Importer

_Requires **serious** computing power!_

* parses a given [Wikipedia XML-Dump](https://dumps.wikimedia.org/)
* stores the Wikipedia articles in a Cassandra database
* creates an inverse index of n-gram hashes

## step 1: XML-Dump
Download the latest XML-Wikipedia Dump (or parts of it for test) from [here.](https://dumps.wikimedia.org/dewiki/latest/dewiki-latest-pages-articles-multistream.xml.bz2)

Use the bzip2 tool to unpack it:

```bash
bzip2 --decompress --keep dewiki-latest-pages-articles-multistream.xml.bz2
```
This will create a .xml file with the same name (without the bzip2 extension) and also keep the original .bz2 file.

## step 2: Cassandra (local) setup
Start `cqlsh` and create the required keyspace (=database) with all the tables. On ubuntu it goes like this `/usr/bin/sqlsh` if u installed it from a [debian package.](http://cassandra.apache.org/download/)

```cql
CREATE KEYSPACE wiki WITH replication = {'class': 'SimpleStrategy', 'replication_factor': '1'} ; 
```

Articles table:
```cql
CREATE TABLE articles(
 		 docid int PRIMARY KEY,
 		 title text,
 		 wikitext text
		 );
```

Tokens table: 
```cql
CREATE TABLE tokens(
		  docid int PRIMARY KEY,
		  tokens frozen<List<text>>
		  ) WITH COMPACT STORAGE;
```


Inverse-index table: 
```cql
CREATE TABLE ii(
		  ngram_hash bigint,
		  docid int,
		  occurrences frozen<list<int>>,
		  PRIMARY KEY(ngram_hash, docid)
		  ) WITH COMPACT STORAGE;
```

## step 3: build
```bash
sbt "project wikipediaImporter" clean assembly
```

## step 4: start

```bash
spark2-submit $SPARK wiki_importer_indexer.jar $OPTIONS   
```

```
$OPTIONS are: 
Following options are mandatory:
•	-dh database host
•	-dp database port
•	-du database user
•	-dpw database password
•	-dn database name e.g. "wiki"
•	
•	-at the name of the wikipedia-articles table
•	-tt the name of the tokenized table
• -it the name of the inverse-index table

Following options determine how the application will behave
•	-e parse wiki XML file and write the articles to Cassandra 
•	-t tokenize and normalize the parsed articles 
•	-i build an inverse index for hashesh of n-grams of a given n
```
e.g.
```bash
wiki_importer.jar -e /home/user/downloads/wiki-articles.xml -dh 127.0.0.1 -dp 9042 -du user -dpw password -dn wiki_dec -at articles -it ii -tt tokens 
```

sample $SPARK parameters for the different options(depend a lot on your system): 
```bash
-e  --master yarn --executor-memory 20G --driver-memory 6G
-t  --master yarn --executor-memory 4G --driver-memory 2G --num-executors 8 --executor-cores 5
-i  --master yarn --executor-memory 5G --driver-memory 3G --num-executors 3 --executor-cores 5
```


