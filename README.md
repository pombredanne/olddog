# olddog

The repository contains the code that accompanies the blog post
[Teaching An Old Dog a new Trick](https://www.chriskamphuis.com/2019/03/06/teaching-an-old-dog-a-new-trick.html).

_olddog_ is build using maven:

`
mvn clean package appassembler:assemble
`

_olddog_ takes a Lucene index as input, this can be created by for example [Anserini](https://github.com/castorini/Anserini). 
For example, the Robust 04 collection can be indexed as explained on [this](https://github.com/castorini/Anserini/blob/master/docs/experiments-robust04.md) page.

After creating the index, the csv files representing the tables can be created issuing the following command:

`
nohup target/appassembler/bin/nl.ru.convert.Convert -index path/to/index -docs /tmp/docs.csv -dict /tmp/dict.csv -terms /tmp/terms.csv
`

This creates multiple files that represent the columns of the `docs`, `dict` and `terms` 
tables as described in the blog post. 

The column store relational database [MonetDB](https://www.monetdb.org) can load
these files using the `copy into` [command](https://www.monetdb.org/Documentation/Cookbooks/SQLrecipes/CSV_bulk_loads).

After this it is possible issue the query described in the post:

```sql
WITH qterms AS (SELECT termid, docid, df FROM terms                             
  WHERE termid IN (10575, 1285, 191)),                                          
  subscores AS (SELECT docs.docid, len, term_tf.termid,                         
  tf, df, (log((528155-df+0.5)/(df+0.5))*((tf*(1.2+1)/                          
  (tf+1.2*(1-0.75+0.75*(len/188.33)))))) AS subscore                            
  FROM (SELECT termid, docid, df AS tf FROM qterms) AS term_tf                  
  JOIN (SELECT docid FROM qterms                                                
    GROUP BY docid HAVING COUNT(distinct termid) = 3)                           
    AS cdocs ON term_tf.docid = cdocs.docid                                     
  JOIN docs ON term_tf.docid=docs.docid                                         
  JOIN dict ON term_tf.termid=dict.termid)                                      
SELECT scores.docid, score FROM (SELECT docid, sum(subscore) AS score           
  FROM subscores GROUP BY docid) AS scores JOIN docs ON                         
  scores.docid=docs.docid ORDER BY score DESC;
```