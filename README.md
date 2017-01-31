# You Can (Not) Recomend 1.11 #

![logo.png](logo.png)

# About
Recommender system engine on NodeJS.

Uses PostgreSQL database as storage of users, items and ratings.

Made for MyAnimeList data scheme, but can be modified to use on any database with explicit ratings 
(see example with MovieLens data).

Uses explicit matrix factorization algorithms: ALS (primary, multi-threaded) and SGD (single-threaded, obsolete).

Matrix operations are accelerated by using C++ BLAS/LAPACK libraries binded to nodejs (see my forks of [nblas](https://github.com/ukrbublik/nblas-plus), [vectorious](https://github.com/ukrbublik/vectorious-plus)).

With ALS algorithm matrix factorization task is parallelized, runs in separate NodeJS worker processes. 
(By default 1 worker because BLAS/LAPACK is multi-threaded itself and utilizes all CPU threads.) 

Can be split to several PCs using simple implemented clustering with [IPC sockets](https://github.com/ukrbublik/quick-tcp-socket).

During training user/item factors matricies data are stored in [shared memory](https://github.com/ukrbublik/shm-typed-array) to be accessible by worker processes.

There is also option to store them in files (if RAM is low or data is too big to fit in RAM). 

Persistent storage of user/item factors - files.


# Usage
### Test on MovieLens data:
- Create PostgreSQL db, import schema from `data/db-schema.sql`
- Set options in `config/config-*.js`
- Import data (100k or 1m)
```bash
npm install
dbType=ml node index.js import_ml 1m
```
- Start
```bash
dbType=ml node index.js start
```
- Open in browser `http://localhost:8004/train`
- See progress at stdout
- After train complete - open in browser `http://localhost:8004/userid/<uid>/recommend`

### Test on MyAnimeList data:
- Create PostgreSQL db, import schema from `data/db-schema.sql`
- Set options in `config/config-*.js`
- See my [malscan](http://github.com/ukrbublik/malscan) project to import data (can take much time)
- Start
```bash
npm install
dbType=mal node index.js start
```
- Open in browser `http://localhost:8004/train`
- See progress at stdout
- To test recommendations open in browser `http://localhost:8004/user/<login in mal>/recommend`


# Using clustering

### master node
- Set options in `config/config-base.js`, `config/config-master.js`
  - `emf.clusterServerPort` (default: 7101) (port to listen in cluster)
- Start
```bash
$ dbType=mal configPrefix=master node index.js start
Listening cluster on port 7101
API listening on port 8004
... after slave started
Cluster node #s1 registered:  { port: 7104, host: 'localhost' }
... after gather
Let cluster slaves to connect with each other (full-mesh)...
Connected to cluster node #s1
Cluster is gathered: 1 slaves
```

### slave node
- Set options in `config/config-base.js`, `config/config-slave.js`
  - `emf.clusterMasterPort` (default: 7101) (should be equal to master's `emf.clusterServerPort`)
  - `emf.clusterServerPort` (default: 7104) (port to listen in cluster)
- Start
```bash
$ dbType=mal configPrefix=slave node index.js start
Listening cluster on port 7104
Connected to Lord
Lord registered me as #s1
API listening on port 8011
... after gather
Cluster node #m client connected
```

### API
- Open in browser `http://localhost:8004/gather` to let all nodes in cluster know each other (full-mesh).
  This step is not necessary, will be automatically performed on train start.
- Now can train as usual. 
  See progress at stdout on master and slave nodes.
  If some slave node will be disconnected, master will detect and handle it.


# Options
See `EmfBase.DefaultOptions`

todo... describe here


# Performance
Soft: 

- Ubuntu 16.04 LTS
- nodejs v6.9.2

Hard:

- PC#1: Core i5-6500, 16GB DRR4-2133, SSD Samsung 850
- Laptop#1: Lenovo Z570 - Core i5-2450M, 8GB DDR2, HDD 5400rpm (connected as slave with Fast Ethernet)

### MovieLens 1m @ PC#1
- 6040 users, 3883 items, 1M ratings
- 100 factors, 85/10/5% split
- Times per iteration: 2x 3.2s for U/I factors
- RMSE: ~0.842 (normalized 0.168) (after 10 iters)

### MAL @ PC#1
- 1.75M users with lists (2.13M without), 12.7K items, 121M ratings
- 100 factors, 85/10/5% split
- Prepare: first time (splitting all ratings to sets) too long - 1h:5m (todo: how to optimize?)
- Times per iteration: 2x 660s for U/I factors, 3x 25s for RMSE
- RMSE: 1.17 (normalized 0.117) (after 3 iters)

todo... do more iters

### MAL @ PC#1 + Laptop#1 (cluster)
todo...


# Thoughts
You can:

- use matrix factorization as base algo
- use item-to-item recommendations for items rated by user (todo)
- use kNN as secondary algo (use users - nearest neighbours) (todo)
- use social info (friends, clubs) (todo)

You can not:

- include in recommendations items that user already plans to watch or watched but not rated
- repeat items of same mediafranchise (todo)

