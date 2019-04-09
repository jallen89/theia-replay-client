Theia DIFT Module
================

The Theia DIFT Module was developed to provide a simpler method for TA2 teams
to communicate with Theia's internal replay server.  Theia’s replay server
relies on recording the system’s original execution, which allows it to essentially 
replay the execution at a later time. Unlike approaches for whole-system record-and-replay
in the past, Theia's replay system has the capability of dynamically instrumenting 
the replay of the execution, which allows Theia to repeatedly apply fine-grained analysis
techniques such as taint analysis on top of the original execution. This allows
Theia to provide a refined version of the provenance graph that only contains
the true dependencies and does not suffer from dependency explosion. 


Engagement 5 Notes:
=======

#### E5 Sample Query Data:

We have provided sample Theia CDMv20 data in topic `ta1-theia-e5-replay-data-1`, which TA2 teams can 
use to become familiar with our `theia-client` and replay server before the
engagement. The topic contains records generated using the `ripe-prov-tests` and some background noise. 
Each team can use their respective  replay server to make queries on the data in this topic. 


#### Example Query

The command below runs a sample forward query on the file `/home/darpa/ripe-prov-tests/Linux/README.md` 
and the query result will show it leaks information to `/home/darpa/ripe-prov-tests/Linux/out.tmp`.
```
theia-client forward-query 0100d00f-980d-1a00-0000-0000730dad5c 81286ee1-d2eb-53d8-aeab-db2728f9fb0f
```

#### Known Issues

* Before each replay, Theia will do a system-call level backward or forward
  analysis. If the `hops` parameter is set to a high value (greater than 5),
  then it may result in very long replay time.


Prerequisties
=====

pip install -r requirements.txt


Setup
=====

Modify `theia-client.cfg`, 

```
[metadata]
# The team name.
team=theia
# address and port number of theia replay server.
replay_server_ip=127.0.0.1
```

Set the team name to the correct value. We will provide the replay server ip
for each TA2 once they are setup on BBN.


Overview
========
* DIFT Client Interface 
* DIFT REST Interface



DIFT Client Interface
=====================

The DIFT Client Interface provides a CLI interface, which has three main capabilities:

* Sending forward, backward, and point-to-point queries to Theia's replay
   server. 
* Provide the status of a query.
* Cancel an outstanding query.

Additionally, there are several advantages to using the CLI utility, since it
maintains unique IDs for each topic and manages connecting to the RESTful API
endpoints.


### Usage

To get a list of commands.

```
$ ./theia-client --help
Usage: theia-client [OPTIONS] COMMAND [ARGS]...

Options:
  --help  Show this message and exit.

  Commands:
    backward-query        Sends a request to make a backward query to...
    forward-query         Sends a request to make a forward query to...
    point-to-point-query  Sends a request to make a point-to-point...
    show                  Show all queries made.
    status                Sends a request for a status update for a...
```

### Get information about a specific command.

```
./theia-client backward-query --help
Usage: theia-client backward-query [OPTIONS] UUID HOST_UUID

  Sends a request to make a backward query to Theia's replay system.

  Arguments: uuid - The UUID of the CDM record in which the query will be
  applied.

  host-uuid - The UUID of the Host.

  start-ts - The start timestamp. Event records with timestamps prior to the
  start timestamp will not be involved in the query (default: 0).

  end-ts - The end timestamp. Event records with timestamps after the end
  timestamp will not be involved in the query (default: 9999999999999999999).

  hops - The analysis will span across 'hops' nodes from the target node
  (default 2) NOTE: (It is recommended to use <= 5 hops).

  timeout - A timeout for the query in minutes (default 30 minutes).

  Returns:     Returns a receipt for this query along with the query's ID.

Options:
  --start-ts INTEGER
  --end-ts INTEGER
  --hops INTEGER
  --timeout INTEGER
  --help              Show this message and exit.
```

Examples
=====
CLI Interface examples

#### Sending a backward query 

```
$ theia-client backward-query 0100d00f-3c1e-1400-0000-00001ac8d85b 440c00a0-22c6-b07f-0000-000000000050 1537403124236956107 1537403125239814679 4 
```


#### Receive the status of a query's progress.
Internally, the `theia-client` utility will maintain the query ID, which can be
used to access information related to the query at a later point. 

```shell
$ theia-client status theia20
$ Query Status: "Finished."
```

The status of a query can have the following states:

* Initialized -- A new query has been created.
* Replaying -- The query is in the process of being replayed.
* Reachability Analsys -- The server is running reachability analysis.
* Finished -- The query is finished, and the Kafka topic contains the
  Provenance records. 
* Timeout -- The query did was timed out.


#### List all queries made
You can locate the query ID for all queries using the `show` command:

```
$ theia-client show
{'_id': 'Theia1', 'status': 'Finished.', 'uuid': '440c00a0-22c6-b07f-0000-000000000050', 'hops': u'4', 'query_type': 'backward', 'start': '1537403124236956107', 'end': '1537403125239814679',}
{'_id': 'Theia2', 'status': 'Finished.', 'uuid': '850b6f01-0000-0000-0000-000000000021', 'hops': u'3', 'query_type': 'forward', 'start': '1537403124236034502', 'end': '1537403124236069253'}
```


Provenance Tag CDM Records
====

For each query, a unique Kafka topic will be created and  the topic will 
match the query id. For example, if the query id is theia20, then the kafka
topic would also have the name theia20.



DIFT REST Interface
====================

It is recommended to use the `theia-client` CLI utility instead of connecting 
directly to the API endpoints. However, for teams that wish to do so, we provide
documentation below: __The CLI should only be used for E4.__

Endpoint: queries
---

### POST /queries

Create a new query, which will be placed in the replay queue.

* Content-Type: "application/json"
* Accept: "application/json"

Include the following parameters in the body:

* `_id`  A unique ID for the query.

* `query_type` - The query type (backward, forward, point-to-point).
* `uuid`   The UUID of the CDM record in which the query will be applied.
* `start`  The start timestamp. Event records with timestamps prior to the 
* `end`  Event records with timestamps after the end timestamp will not be involved in the query.
* `hops` The analysis will span across 'hops' nodes from the target node.
* `timeout ` (optional) The timeout for a query to finish. The default value is 30
* `uuid_end` (optional) This is an optional parameter, that is only meaningful for point-to-point queries.  

``` 
{ 
    '_id': 'theiaT209',
    'query_type': 'backward', 
    'uuid': '440c00a0-22c6-b07f-0000-000000000050',
    'start': '1537403124236956107', 
    'end': '1537403125239814679', 
    'hops': '4' 
}
```

### GET /queries/{query-id}

Get the status of a query.


### DELETE /queries/{query-id}

Cancel a query in process.


[ci-docs]: https://git.tc.bbn.com/carter/theia-ci-docs/blob/master/README.md
