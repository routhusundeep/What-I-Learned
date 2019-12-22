#+TITLE: Daily Notes Friday, 29/11/2019
** [[https://github.com/postgres/postgres/blob/master/src/backend/access/heap/README.HOT][HOT]]                                             :heap_only_tuple:postgres:
Effect on Index update due to a table update can be classified into cold and hot, cold, when an indexed column is modified and hot when not, for hot there is no necessity to update an index and this feature does exactly that.
This feature also facilitates vacuuming(garbage collection) at a single page granularity instead of a typical full table one which is very costly.
The README talks about various other advantages and a few disadvantages due to this approach, Its affect on other operations like concurrent index creation(helpful [[https://www.2ndquadrant.com/en/blog/create-index-concurrently/][link]]) etc.

Really puts a lot of things into perspective, how even some minute details of modern systems are riddled with innovation and complexity beyond imaginable.

** [[http://www.interdb.jp/pg/][Internals]]                                    :postgres:postgres_internals:
This site explains the internals of POSTGRES, goes into the deeper technical trenches to prove the point. 
I have read most of the site which has been very insightful, will go through [[http://www.interdb.jp/pg/pgsql03.html][query processing]] again tomorrow, since it is the piece which has not been most intuitive(probably due to the mathematics involved).
Will make more detailed note about this at a later date.
