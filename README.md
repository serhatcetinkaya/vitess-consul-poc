# vitess-consul-poc

This repo has resources to create a proof of concept showing that when Consul leader node is unreachable, Vitess returns errors to queries. 

Resources are intended to be used in kubernetes, tests are done using minikube:

After cloning this repo, create a minikube cluster and a namespace:

```bash
minikube start -p vitess
kubectl create ns cloud
```

Deployment order:

- Apply `consul.yaml` file, it will create a consul client pod and 3 server nodes in cluster. Once it is ready you can get URL for consul UI from minikube and test access (http://192.168.64.6:32754 is the test URL for this example):
```
kubectl apply -f consul.yaml
minikube service list -pvitess
|-------------|---------------|---------------------------|-----|
|  NAMESPACE  |     NAME      |        TARGET PORT        | URL |
|-------------|---------------|---------------------------|-----|
| cloud       | consul-client | No node port              |     |
| cloud       | consul-server | No node port              |     |
| cloud       | consul-ui     | http://192.168.64.6:32754 |     |
| default     | kubernetes    | No node port              |     |
| kube-system | kube-dns      | No node port              |     |
|-------------|---------------|---------------------------|-----|
```

- Apply `vtctld.yaml` file to deploy vtctld pod. It has a nginx sidecar to proxy consul requests (localhost:8500) to consul client:

```
kubectl apply -f vtctld.yaml
```

- SSH into vtctld container and run following commands:

```
kubectl exec --namespace cloud --stdin --tty $VTCTLD_POD_NAME -- /bin/sh
/vt/bin/vtctlclient -server localhost:15999 AddCellInfo -server_address localhost:8500 -root vitess/us_east_1 us_east_1
/vt/bin/vtctlclient -server localhost:15999 CreateKeyspace peak-test
/vt/bin/vtctlclient -server localhost:15999 CreateShard peak-test/0
```

- Apply `mysql.yaml` file to create vttablet and mysql services (it takes 20-30 seconds for mysql to be up and running, so vttablet container might get restarted 1-2 times until then):

```bash
kubectl apply -f mysql.yaml
```

- SSH into vtctld container to init shard master:

```bash
kubectl exec --namespace cloud --stdin --tty $VTCTLD_POD_NAME -- /bin/sh
/vt/bin/vtctlclient -server localhost:15999 InitShardMaster -force peak-test/0 us_east_1-1126369102
```

- Apply `vtgate.yaml` to create vtgate service:

```bash
kubectl apply -f vtgate.yaml
```

- SSH into vtctld container one last time to do a planned reparent shard:

```bash
kubectl exec --namespace cloud --stdin --tty $VTCTLD_POD_NAME -- /bin/sh
/vt/bin/vtctlclient -server localhost:15999 PlannedReparentShard -keyspace_shard=peak-test/0 -new_master=us_east_1-1126369102
```

At this step after making sure every pod is up and running we port-forward `:3306` from vtgate pod to our local and test connection:

```bash
kubectl port-forward pod/$VTGATE_POD_NAME 3306:3306 --namespace cloud
Forwarding from 127.0.0.1:3306 -> 3306
Forwarding from [::1]:3306 -> 3306
```

```bash
➜  ~ mysql -h127.0.0.1 -uroot -ppassw0rd
mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 2
Server version: 5.7.29-32 Percona Server (GPL), Release 32, Revision 56bce88

Copyright (c) 2000, 2020, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql>
```

Now the setup is complete, from the consul UI, check the leader and note it down somewhere (the leader will have a star on it in `Nodes` page). You can use ACL token `6252a23b-6726-49a0-9e9b-0850f1321726` to login/authenticate consul. We will use `sysbench` utility from our local to create a read-write load on vitess.

Prepare:

```bash
sysbench --db-driver=mysql --mysql-user=root --mysql_password=passw0rd --mysql-db=peak-test --mysql-host=127.0.0.1 --mysql-port=3306 --tables=1 --table-size=500 --threads=1 --time=0 --events=50 --rate=10 --report-interval=1 oltp_read_write.lua --db-ps-mode=disable prepare
```

Run:

```bash
sysbench --db-driver=mysql --mysql-user=root --mysql_password=passw0rd --mysql-db=peak-test --mysql-host=127.0.0.1 --mysql-port=3306 --tables=1 --table-size=500 --threads=1 --time=0 --events=50 --rate=10 --report-interval=1 oltp_read_write.lua --db-ps-mode=disable run
```

Test should finish without any errors in couple of seconds. After confirming this, run another test, but just after the new test starts kill the leader consul pod to force leader election:

```bash
kubectl delete node $LEADER_CONSUL_POD --namespace cloud
```

After consul cluster starts new leader election and becomes unavailable for a short period of time, sysbench will get an error similar to following:

```text
[ 16s ] thds: 1 tps: 0.00 qps: 0.00 (r/w/o: 0.00/0.00/0.00) lat (ms,95%): 0.00 err/s: 0.00 reconn/s: 0.00
[ 16s ] queue length: 142, concurrency: 1
FATAL: mysql_drv_query() returned error 1105 (vtgate: http://vtgate-69d69486f4-4lm8f:8080/: paramsAnyShard: keyspace peak-test fetch error: Unexpected response code: 500) for query 'SELECT c FROM sbtest1 WHERE id=271'
FATAL: `thread_run' function failed: ./oltp_common.lua:426: SQL error, errno = 1105, state = 'HY000': vtgate: http://vtgate-69d69486f4-4lm8f:8080/: paramsAnyShard: keyspace peak-test fetch error: Unexpected response code: 500
```
