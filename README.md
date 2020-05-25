# kubesql

kubesql is a tool to use sql to query the resources of kubernetes.

The resources of kubernetes such as nodes, pods and so on are handled as tables.

For example, all pods are easily to list from apiserver. But the number of pods on each node is not easy to caculate.

```
presto:kubesql> select nodename, count(*) as pod_count from pods group by nodename;
   nodename    | pod_count 
---------------+-----------
 10.111.11.118 |         8 
(1 row)

Query 20200513_094558_00006_7cycm, FINISHED, 1 node
Splits: 49 total, 49 done (100.00%)
0:00 [8 rows, 0B] [48 rows/s, 0B/s]
```

# deploy

## docker

In case the kubeconfig file is located at `/root/.kube/config`. Just run and enjoy.

```
docker run -it -d --name kubesql -v /root/.kube/config:/home/presto/config xuxinkun/kubesql:latest
docker exec -it kubesql presto --server localhost:8080 --catalog kubesql --schema kubesql
```

## kubernetes

TODO

# architecture

kubesql makes use of **presto** to execute the sql. 

There are three main modules in kubesql:

![kubesql-arc](https://xuxinkun.github.io/img/kubesql/kubesql.png)


-kubesql-watcher: monitor k8s api pod and node changes. And convert the structured data of pod and node into relational data.
-kubecache: used to cache pod and node data.
-kubesql-connector: as presto connector, accept calls from presto, query column information and corresponding data through kubecache, and return to presto about column and data information.

Since all data is cached in memory, there is almost no disk requirement. But it also needs to provide larger memory according to the size of the cluster.

## tables and columns

Taking pod data as an example, the main data in the pod is divided into three parts, metadata, spec, and status.

The more difficult parts of metadata are label and annotation. I flatten the label map, each key is used as a column.

```
labels:
    app: mysql
    owner: xxx
```

I use labels as the prefix, and put the keys in the labels as the column names. Thus two pieces of data are obtained:


``` 
labels.app: mysql
labels.owner: xxx
```

For pod A there is an app label but pod B does not have the label, then for pod B, the value of this column labels.app is null.


Similar annotations are handled similarly. Thus, annotations can be used to filter pods.

For spec, the biggest difficulty lies in the handling of containers. Because there may be several containers in a pod, I will directly use containers as a new table. At the same time, a uid column is added to the containers table to indicate which pod this row of data comes from. The fields in the containers are also added to the containers table. The more important resources in containers such as request and limit, I directly use requests. as a prefix, and spell resource as a column name. For example, `requests.cpu`,` requests.memory`, etc. Here, the CPU is processed as a double type and the unit is a core. For example, 100m will be converted to 0.1. The memory is bigint and the unit is B.

For status, conditions and containerStatus are more difficult to deal with. The conditions are a list, but the type of each condition is different. So I use type as a prefix to generate column names for conditon. such as:

``` 
  conditions:
  - lastProbeTime: null
    lastTransitionTime: 2020-04-22T09:03:10Z
    status: "True"
    type: Ready
  - lastProbeTime: null
    lastTransitionTime: 2020-04-22T09:03:10Z
    status: "True"
    type: ContainersReady
```

Then in the pod table, I can get these columns:

|                Column                 |   Type    | Extra | Comment |
|---------------------------------------|-----------|-------|---------|
| containersready.lastprobetime         | timestamp |       |         |
| containersready.lasttransitiontime    | timestamp |       |         |
| containersready.message               | varchar   |       |         |
| containersready.reason                | varchar   |       |         |
| containersready.status                | varchar   |       |         |
| ready.lastprobetime                   | timestamp |       |         |
| ready.lasttransitiontime              | timestamp |       |         |
| ready.message                         | varchar   |       |         |
| ready.reason                          | varchar   |       |         |
| ready.status                          | varchar   |       |         |

In this way, I can filter pods of type ready and status True in condition by "ready.status" = "True".

Since containerStatus corresponds one-to-one with containers, I merge containerStatus into the containers table, and correspond one-to-one with the container name.

## example

```
[root@localhost kubesql]# docker exec -it kubesql presto --server localhost:8080 --catalog kubesql --schema kubesql
presto:kubesql> show tables;
   Table    
------------
 containers 
 nodes      
 pods       
(3 rows)

Query 20200513_094219_00003_7cycm, FINISHED, 1 node
Splits: 19 total, 19 done (100.00%)
0:01 [3 rows, 70B] [2 rows/s, 54B/s]
```


For example, you want to query the CPU resources of each pod (requests and limits).


```
presto:kubesql> select pods.namespace,pods.name,sum("requests.cpu") as "requests.cpu" ,sum("limits.cpu") as "limits.cpu" from pods,containers where pods.uid = containers.uid group by pods.namespace,pods.name
     namespace     |                 name                 | requests.cpu | limits.cpu 
-------------------+--------------------------------------+--------------+------------
 rongqi-test-01    | rongqi-test-01-202005151652391759    |          0.8 |        8.0 
 ljq-nopassword-18 | ljq-nopassword-18-202005211645264618 |          0.1 |        1.0 
```

Another example is that I want to query the remaining CPUs on each node.

```
presto:kubesql> select nodes.name, nodes."allocatable.cpu" - podnodecpu."requests.cpu" from nodes, (select pods.nodename,sum("requests.cpu") as "requests.cpu" from pods,containers where pods.uid = containers.uid group by pods.nodename) as podnodecpu where nodes.name = podnodecpu.nodename;
    name     |       _col1        
-------------+--------------------
 10.11.12.29 | 50.918000000000006 
 10.11.12.30 |             58.788 
 10.11.12.32 | 57.303000000000004 
 10.11.12.34 |  33.33799999999999 
 10.11.12.33 | 43.022999999999996 
```

Another example is to query all pods created after 2020-05-12.

```
presto:kube> select name, namespace,creationTimestamp from pods where creationTimestamp > date('2020-05-12') order by creationTimestamp desc;
                         name                         |        namespace        |    creationTimestamp    
------------------------------------------------------+-------------------------+-------------------------
 kube-api-webhook-controller-manager-7fd78ddd75-sf5j6 | kube-api-webhook-system | 2020-05-13 07:56:27.000 
```

You can also query based on tags. The appid of all tags is springboot, and pods that have not been scheduled successfully. And counting.


The label appid will have a column in the pods table with the column name "labels.appid". Use this column as a condition to delete pods.

```
presto:kubesql> select namespace,name,phase from pods where phase = 'Pending' and "labels.appid" = 'springboot';
     namespace      |     name     |  phase  
--------------------+--------------+---------
 springboot-test-rd | v6ynsy3f73jn | Pending 
 springboot-test-rd | mu4zktenmttp | Pending 
 springboot-test-rd | n0yvpxxyvk4u | Pending 
 springboot-test-rd | dd2mh6ovkjll | Pending 
 springboot-test-rd | hd7b0ffuqrjo | Pending
 
 presto:kubesql> select count(*) from pods where phase = 'Pending' and "labels.appid" = 'springboot';
  _col0 
 -------
      5 
```

# plan

At present, there are only pods and nodes resources. Compared with the huge resources of k8s, it is only the tip of the iceberg. But increasing each resource requires adding a considerable amount of code. I'm also thinking about how to use openapi's swagger description to automatically generate code.


The deployment is now using docker to deploy, and the deployment method of kubernetes will be added soon, which will be more convenient.


At the same time, I am thinking that in the future, each worker in Presto will be responsible for a cluster cache. Such a presto cluster can query all k8s cluster information. This function also needs to be redesigned and considered.