# hometask

## Before I begin
First of all, the task is definitely too big to implement in code given the time constraints.  
Even not taking into account the requirement to produce high-quality code.  
As a result I decided to focus instead on describing the approach and architecture as thorough as possible.  
It is still too big to cover in a couple of days so I pinpointed those topics that you would deem most important in my opinion.

## General architecture
The solution is comprised of two regions as this a requirement.
Each region has the following entities running:

1. ALB to handle incoming requests, terminate HTTPS, run checks against the application and route requests to the application.
ALB balances across multiple AZs;
2. Applications that run as ECS tasks in a separate application subnets.
There is no requirement not to use managed services for application but if it is the application can be placed on EC2/ASG;
3. Patroni + PostgreSQL cluster comprised of two EC2 nodes running in private database subnets in two regions (can be three for higher availability);
4. Consul cluster running on EC2 in control plane private subnets and comprised of 5 nodes running in 3 different AZs.
Consul serves as DCS (voting place) for Patroni;
5. Route53 as DNS service or any other self-hosted solution;
6. Bastion host running in public network to manage infrastructure;
7. pgbackrest is used for database backups and this one runs on each database host.
pgbackrest service and repo are started on EC2 in separate backup private network to hold database backups and also maintaining copies in S3 bucket;
8. All subnets have security groups in place to restrict connections.
ALB is allowed to connect only to application subnets, application is allowed to connect only to the database nodes,
database nodes connect to the Consul nodes and pgbackrest node;
9. Database nodes also run postgres_export Prometheus agent. All nodes run Promtail agent to stream logs;
10. Prometheus and Alertmanager handle metrics and alarms;
11. Loki handles logs processing;
12. Grafana is used to visualize logs and metrics.

Disaster recovery is achieved by making use of Patroni standby cluster feature.  
In order to be ready for site failover it makes sense to observe the availability of some highly available cross-zone resource and ALB makes a perfect candidate for this.  
In case ALB fails resolving (it has static FQDN configured) this will be a sign of ALB failure.  
Route53 is able to do this natively so the service must be tied to FQDN that does not change but A/CNAME record will.  
CloudWatch can also trigger alarm on DNS failover and feed it to Lambda which will get to Patroni API and ask Patroni to promote standby leader.  
That way both PostgreSQL+Patroni and ALB will be ready to serve requests from the surviving site.

## Database management
Database operation is designed around PostgreSQL running in HA cluster with Patroni standby HA cluster running in another region.  
This solution allows to employ DR solution.  
Partoni uses Consul as DCS.  
Consul employs Consensus/Raft protocol to proivide consistency.  
The number of Consul nodes is chosen as Consul's best practice for optimum number of nodes for quorum and fault tolerance.  
I am sorry for not covering Patroni and Consul too much.  
You are aware I have been working with Patroni a lot recently so I am not telling about it much.  
I installed Consul to get some understanding of it.  
It is not too complex to start but I believe more challenging to manage.

## Database and schema design
I will create a database to host the schema:
```
> create database hello;
```
In PostgreSQL all users are able to connect to any database and issue non-privileged statements in public schema  
which was handy in 2000's but currently considered insecure so I am restricting this:
```
> revoke connect on database hello from public;
```
At this time I will connect to the newly created database and revoke public schema usage for the reason expressed above:
```
> revoke usage on schema public from public;
```
Then I will create schema role *without* login privilege and it will be solely used to store database objects.  
This is used to separate storage from operations:
```
> create role hello;
```
And the shema to hold database objects:
```
> create schema authorization hello;
```
Now to the database objects themselves:
```
> create table hello.birthdays (username varchar(30), birthday date, constraint pk_birthdays primary key (username) include (birthday));
```
And set the ownership of database objects to hello role:
```
> alter table hello.birthdays owner to hello;
> alter index hello.pk_birthdays owner to hello;
```
Bear in mind that I just created covering index with the include clause to optimize relation access.  
In case all the tuples in the relation are visible PostgreSQL won't need to visit the table and will 'cover' selects solely using index.  
Now create app_user role and grant privileges to it.  
Password is supplied pre-hashed but I am still not showing it.
```
> create role app_user encrypted password 'supply_password_here' login;
> grant select, insert, update on hello.birthdays to app_user;
> grant connect on database hello to app_user;
> grant usage on schema hello to app_user;
```

## Application
Application can be developed around basically any REST framework. I have a bit of experience with Flask and I know it can get the job done.  
A special care should be taken around user passing username in the request.  
This is in general an undesirable approach as the user in this case is given control over input data which can result in adverse behaviour.  
More specifically this gives an opportunity for malicious user to craft SQL injection attack on the database potentially leading to data destruction and compromise.  
Therefore username should be properly sanitized to ensure this does not happen.  
In addition the application should get to the user with meaningful errors in case birthday date or username is malformed.  
Application performs periodic checks towards Patroni API to:
1. Make sure that the database connection points to the leader and rewrites it accordingly;
2. Stops processing requests when leader is not available and signalling ALB over exposed health endpoint that the application cannot handle incoming connections.

Upon receiving PUT request the application issues MERGE SQL statement:

```
> merge into hello.birthdays trgt using (values (:1, :2)) as src(username, birthday) on trgt.username = src.username
> when matched then update set birthday = src.birthday
> when not matched then insert (username, birthday) values (src.username, src.birthday);
```

If username is found in the table, birthday will be updated and a new username will be created otherwise.  
For GET a select query will be used.  
I will demonstrate the ability of covering index and the overall flow with the following example  
(sorry this is lengthy but it wasn't intentional and turned out be an exciting experiment that I decided to share):

```
hello=> merge into hello.birthdays trgt using (values ('a', current_date)) as src(username, birthday) on trgt.username = src.username
hello-> when matched then update set birthday = src.birthday
hello-> when not matched then insert (username, birthday) values (src.username, src.birthday);
MERGE 1
hello=> select * from hello.birthdays;
 username |  birthday
----------+------------
 a        | 2025-09-28
(1 row)
hello=> explain analyze select birthday from hello.birthdays where username='a';
                                                         QUERY PLAN
-----------------------------------------------------------------------------------------------------------------------------
 Index Only Scan using pk_birthdays on birthdays  (cost=0.15..8.17 rows=1 width=4) (actual time=0.039..0.040 rows=1 loops=1)
   Index Cond: (username = 'a'::text)
   Heap Fetches: 1
 Planning Time: 0.062 ms
 Execution Time: 0.057 ms
(5 rows)
```
As you can see the index is used but I still have heap fetches which is because the visibility map is not updated for this relation.  
Let's switch to the superuser and vacuum it:
```
hello=# vacuum hello.birthdays;
```
Vaccum command also updates visibility map. Let's rerun select once again:
```
hello=> explain analyze select birthday from hello.birthdays where username='a';
                                            QUERY PLAN
---------------------------------------------------------------------------------------------------
 Seq Scan on birthdays  (cost=0.00..1.01 rows=1 width=4) (actual time=0.006..0.007 rows=1 loops=1)
   Filter: ((username)::text = 'a'::text)
 Planning Time: 0.257 ms
 Execution Time: 0.039 ms
(4 rows)
```
Now the index does not work at all.  
The reason is simple -- this is a very tiny table.  
Let's create a million of tuples:
```
hello=> INSERT INTO hello.birthdays(username, birthday)
        SELECT substring(md5(random()::text) from 1 for 10) || '_' || gs::text AS username,
        DATE '1950-01-01' + (random() * (DATE '2010-12-31' - DATE '1950-01-01'))::int AS birthday
        FROM generate_series(1, 1000000) AS gs;
INSERT 0 1000000
hello=> select count(*) from hello.birthdays;
  count
---------
 1000003
(1 row)
```
Now vacuum and analyze once again:
```
hello=# vacuum analyze hello.birthdays;
hello=> explain analyze select birthday from hello.birthdays where username='a';
                                                         QUERY PLAN
-----------------------------------------------------------------------------------------------------------------------------
 Index Only Scan using pk_birthdays on birthdays  (cost=0.42..4.44 rows=1 width=4) (actual time=0.041..0.043 rows=1 loops=1)
   Index Cond: (username = 'a'::text)
   Heap Fetches: 0
 Planning Time: 0.784 ms
 Execution Time: 0.073 ms
(5 rows)
```
Now we can see at last that covering index works as expected which demonstrates it's power and shows the importance of running vacuum/analyze.  
Worth noting that the execution time merely doubled while the number of rows changed drastically.

## Observability
I already described the idea behind site failovers and what kind of observability it needs.  
postgres_exporter will deliver lots of metrics.  
The most important to look closely are: availability of services, connections reaching max_connections, as well as other static parameters such as the number of prepared transactions.  
Also WAL replication lag, replication slot stats and filesystem space usage.  
Top-5 waits and hot objects are also important for tuning.
This system is typical OLTP so it should not have long running transactions and blocking locks occuring too long so this should be observed as well.

## Room for improvement
1. GET requests resulting in selects can be routed to replica(s) to offload primary;
2. Secret management can be employed in the infrastructure using Vault;
3. Connections utilize mTLS and certificate rotation solution is introduced;
4. Users can make mistakes entering their birthdays. They might notice this and will want to fix it but I believe the number of possible fixes should not exceed one;
5. Add authentication so that users can be identified and be restricted to only their username in the system.