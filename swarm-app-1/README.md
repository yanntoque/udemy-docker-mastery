# Assignment: Create A Multi-Service Multi-Node Web App

## Goal: create networks, volumes, and services for a web-based "cats vs. dogs" voting app.

- See architecture.png in this directory for a basic diagram of how the 5 services will work
- All images are on Docker Hub, so you should use editor to craft your commands locally, then paste them into swarm shell (at least that's how I'd do it)
- a `backend` and `frontend` overlay network are needed. Nothing different about them other then that backend will help protect database from the voting web app. (similar to how a VLAN setup might be in traditional architecture)

# Creating the backend and frontend networks
```
root@node1:~# docker network create --driver overlay backend
pe928wlbcpmbsc9jm1p2ylmc7
root@node1:~# docker network create --driver overlay frontend
lka27drtj6bz0r94q3jwhjszo
```
- The database server should use a named volume for preserving data. Use the new `--mount` format to do this: `--mount type=volume,source=db-data,target=/var/lib/postgresql/data`

### Services (names below should be service names)
- vote
    - dockersamples/examplevotingapp_vote:before
    - web front end for users to vote dog/cat
    - ideally published on TCP 80. Container listens on 80
    - on frontend network
    - 2+ replicas of this container

```
docker service create --name vote --network frontend -p 80:80 --replicas 3 dockersamples/examplevotingapp_vote:before

root@node1:~# docker service create --name vote --network frontend -p 80:80 --replicas 3 dockersamples/examplevotingapp_vote:before 
nzxvi72chi9b9xuchfs76vbmi
overall progress: 3 out of 3 tasks 
1/3: running   [==================================================>] 
2/3: running   [==================================================>] 
3/3: running   [==================================================>] 
verify: Service converged 
root@node1:~# docker service ls
ID                  NAME                MODE                REPLICAS            IMAGE                                        PORTS
nzxvi72chi9b        vote                replicated          3/3                 dockersamples/examplevotingapp_vote:before   *:80->80/tcp

```
- redis
    - redis:3.2
    - key/value storage for incoming votes
    - no public ports
    - on frontend network
    - 1 replica NOTE VIDEO SAYS TWO BUT ONLY ONE NEEDED

```
root@node1:~# docker service create --name redis --network frontend --replicas 1 redis:3.2
bbvejrbqaoi427ace5k5sayqe
overall progress: 1 out of 1 tasks 
1/1: running   [==================================================>] 
verify: Service converged 
root@node1:~# docker service ps redis
ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE            ERROR               PORTS
t7yslmh9bjye        redis.1             redis:3.2           node3               Running             Running 17 seconds ago                       
```

- worker
    - dockersamples/examplevotingapp_worker
    - backend processor of redis and storing results in postgres
    - no public ports
    - on frontend and backend networks
    - 1 replica

```
root@node1:~# docker service create --name worker --network backend --network frontend --replicas 1 dockersamples/examplevotingapp_worker
bun2j1gob89f8cxe8b0d0m55n
overall progress: 1 out of 1 tasks 
1/1: running   [==================================================>] 
verify: Service converged 

root@node1:~# docker service ps worker 
ID                  NAME                IMAGE                                          NODE                DESIRED STATE       CURRENT STATE            ERROR               PORTS
l6a3uwni2hk2        worker.1            dockersamples/examplevotingapp_worker:latest   node2               Running             Running 28 seconds ago           
```

- db
    - postgres:9.4
    - one named volume needed, pointing to /var/lib/postgresql/data
    - on backend network
    - 1 replica

```
root@node1:~# docker service create --name db --network backend --mount type=volume,source=db-data,target=/var/lib/postgresql/data --replicas 1 postgres:9.4
rvdklewdi1aze67fungbsdd8n
overall progress: 1 out of 1 tasks 
1/1: running   [==================================================>] 
verify: Service converged 

root@node1:~# docker service ps db 
ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE           ERROR               PORTS
ncb1j6w86kgq        db.1                postgres:9.4        node1               Running             Running 2 minutes ago   
```
- result
    - dockersamples/examplevotingapp_result:before
    - web app that shows results
    - runs on high port since just for admins (lets imagine)
    - so run on a high port of your choosing (I choose 5001), container listens on 80
    - on backend network
    - 1 replica

```
root@node1:~# docker service create --name result -p 5001:80 --network backend --replicas 1  dockersamples/examplevotingapp_result:before
45g2i4o1ze32tefr3kutdew3f
overall progress: 1 out of 1 tasks 
1/1: running   [==================================================>] 
verify: Service converged 

root@node1:~# docker service ps result 
ID                  NAME                IMAGE                                          NODE                DESIRED STATE       CURRENT STATE            ERROR               PORTS
kaasf5vizxs2        result.1            dockersamples/examplevotingapp_result:before   node1               Running             Running 27 seconds ago                       
```


# List of all services and nodes where they are

```
root@node1:~# docker service ls
ID                  NAME                MODE                REPLICAS            IMAGE                                          PORTS
rvdklewdi1az        db                  replicated          1/1                 postgres:9.4                                   
bbvejrbqaoi4        redis               replicated          1/1                 redis:3.2                                      
45g2i4o1ze32        result              replicated          1/1                 dockersamples/examplevotingapp_result:before   *:5001->80/tcp
nzxvi72chi9b        vote                replicated          3/3                 dockersamples/examplevotingapp_vote:before     *:80->80/tcp
bun2j1gob89f        worker              replicated          1/1                 dockersamples/examplevotingapp_worker:latest   
root@node1:~# docker service ps db
ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE           ERROR               PORTS
ncb1j6w86kgq        db.1                postgres:9.4        node1               Running             Running 8 minutes ago                       
root@node1:~# docker service ps redis
ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE            ERROR               PORTS
t7yslmh9bjye        redis.1             redis:3.2           node3               Running             Running 42 minutes ago                       
root@node1:~# docker service ps result 
ID                  NAME                IMAGE                                          NODE                DESIRED STATE       CURRENT STATE           ERROR               PORTS
kaasf5vizxs2        result.1            dockersamples/examplevotingapp_result:before   node1               Running             Running 3 minutes ago                       
root@node1:~# docker service ps vote 
ID                  NAME                IMAGE                                        NODE                DESIRED STATE       CURRENT STATE               ERROR               PORTS
p6hmy8rsn97d        vote.1              dockersamples/examplevotingapp_vote:before   node2               Running             Running about an hour ago                       
ir4s1y5jea06        vote.2              dockersamples/examplevotingapp_vote:before   node3               Running             Running about an hour ago                       
olqe79nehu9j        vote.3              dockersamples/examplevotingapp_vote:before   node1               Running             Running about an hour ago                       
root@node1:~# docker service ps worker 
ID                  NAME                IMAGE                                          NODE                DESIRED STATE       CURRENT STATE           ERROR               PORTS
l6a3uwni2hk2        worker.1            dockersamples/examplevotingapp_worker:latest   node2               Running             Running 7 minutes ago                       
```