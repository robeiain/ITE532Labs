# Docker Swarm Tutorial


## Preparation
You need to have two machines/VMs on the same network running Docker.

This requires clear network communication, so ensure no firewall is active between your hosts and run the following commands on each.
1. `ufw disable`
2. `iptables -F`
3. `iptables -X`

## Creating the nodes and Swarm
1. Run `docker swarm init --listen-addr 10.0.0.20 --advertise-addr 10.0.0.20` where the ip address is the address of the machine you are running from.
2. This will provide you with a command you can copy to use on the other machine which will look like the following:

`docker swarm join --token SWMTKN-1-0i7ecku40oxilhuf7em8mvx5bzb02y2iczt4pg4pk974dlfmqu-95eujgsp1c1qs6fme753xqa9r 10.0.0.20:2377`
3. From the first machine, run `docker node ls` and it will show you status of your cluster:

`root@docker2:~# docker node ls
ID                            HOSTNAME   STATUS    AVAILABILITY   MANAGER STATUS   ENGINE VERSION
89hm11snya6gducfwadhoqn2h     docker1    Ready     Active                          20.10.21
uhz3opjuii09xl8ern9ip7d5p *   docker2    Ready     Active         Leader           20.10.21`
```

That last line will show you a list of all the nodes, something like this:

```
ID                           HOSTNAME  STATUS  AVAILABILITY  MANAGER STATUS
3cq6idpysa53n6a21nqe0924h    manager3  Ready   Active        Reachable
64swze471iu5silg83ls0bdip *  manager1  Ready   Active        Leader
7eljvvg0icxlw20od5f51oq8t    manager2  Ready   Active        Reachable
8awcmkj3sd9nv1pi77i6mdb1i    worker1   Ready   Active        
avu80ol573rzepx8ov80ygzxz    worker2   Ready   Active        
bxn1iivy8w7faeugpep76w50j    worker3   Ready   Active        
```
You can also find all your machines by running
```
$ docker-machine ls
NAME       ACTIVE   DRIVER       STATE     URL                         SWARM   DOCKER      ERRORS
manager1   -        virtualbox   Running   tcp://192.168.99.100:2376           v17.03.0-ce   
manager2   -        virtualbox   Running   tcp://192.168.99.101:2376           v17.03.0-ce 
manager3   -        virtualbox   Running   tcp://192.168.99.102:2376           v17.03.0-ce
worker1    -        virtualbox   Running   tcp://192.168.99.103:2376           v17.03.0-ce
worker2    -        virtualbox   Running   tcp://192.168.99.104:2376           v17.03.0-ce
worker3    -        virtualbox   Running   tcp://192.168.99.105:2376           v17.03.0-ce
```

The next step is to create a service and list out the services. This creates a single service called `web` that runs the latest nginx:
```
$ docker-machine ssh manager1 "docker service create -p 80:80 --name web nginx:latest"
$ docker-machine ssh manager1 "docker service ls"
ID            NAME  REPLICAS  IMAGE         COMMAND
2x4jsk6313az  web   1/1       nginx:latest  
```
Now open the machine's IP address in your browser. You can see above manager1 had an IP address of 192.168.99.100
![nginx in Chrome at 192.168.99.100](images/manager1-nginx.png)

You can actually load any of the node ip addresses and get the same result because of [Swarm Mode's Routing Mesh](https://docs.docker.com/engine/swarm/ingress/).
![nginx in Chrome at 192.168.99.100](images/manager2-nginx.png)

Next let's inspect the service
```
$ docker-machine ssh manager1 "docker service inspect web"
[
    {
        "ID": "2x4jsk6313azr6g1dwoi47z8u",
        "Version": {
            "Index": 104
        },
        "CreatedAt": "2016-08-23T22:43:23.573253682Z",
        "UpdatedAt": "2016-08-23T22:43:23.576157266Z",
        "Spec": {
            "Name": "web",
            "TaskTemplate": {
                "ContainerSpec": {
                    "Image": "nginx:latest"
                },
                "Resources": {
                    "Limits": {},
                    "Reservations": {}
                },
                "RestartPolicy": {
                    "Condition": "any",
                    "MaxAttempts": 0
                },
                "Placement": {}
            },
            "Mode": {
                "Replicated": {
                    "Replicas": 1
                }
            },
            "UpdateConfig": {
                "Parallelism": 1,
                "FailureAction": "pause"
            },
            "EndpointSpec": {
                "Mode": "vip",
                "Ports": [
                    {
                        "Protocol": "tcp",
                        "TargetPort": 80,
                        "PublishedPort": 80
                    }
                ]
            }
        },
        "Endpoint": {
            "Spec": {
                "Mode": "vip",
                "Ports": [
                    {
                        "Protocol": "tcp",
                        "TargetPort": 80,
                        "PublishedPort": 80
                    }
                ]
            },
            "Ports": [
                {
                    "Protocol": "tcp",
                    "TargetPort": 80,
                    "PublishedPort": 80
                }
            ],
            "VirtualIPs": [
                {
                    "NetworkID": "24r1loluvdohuzltspkwbhsc8",
                    "Addr": "10.255.0.9/16"
                }
            ]
        },
        "UpdateStatus": {
            "StartedAt": "0001-01-01T00:00:00Z",
            "CompletedAt": "0001-01-01T00:00:00Z"
        }
    }
]
```

That's lots of info! Now, let's scale the service:
```
$ docker-machine ssh manager1 "docker service scale web=15"
web scaled to 15
$ docker-machine ssh manager1 "docker service ls"
ID            NAME  REPLICAS  IMAGE         COMMAND
2x4jsk6313az  web   15/15     nginx:latest  
```
Docker has spread the 15 services evenly over all of the nodes
```
$ docker-machine ssh manager1 "docker service ps web"
ID                         NAME    IMAGE         NODE      DESIRED STATE  CURRENT STATE           ERROR
61wjx0zaovwtzywwbomnvjo4q  web.1   nginx:latest  worker3   Running        Running 13 minutes ago  
bkkujhpbtqab8fyhah06apvca  web.2   nginx:latest  manager1  Running        Running 2 minutes ago   
09zkslrkgrvbscv0vfqn2j5dw  web.3   nginx:latest  manager1  Running        Running 2 minutes ago   
4dlmy8k72eoza9t4yp9c9pq0w  web.4   nginx:latest  manager2  Running        Running 2 minutes ago   
6yqabr8kajx5em2auvfzvi8wi  web.5   nginx:latest  manager3  Running        Running 2 minutes ago   
21x7sn82883e7oymz57j75q4q  web.6   nginx:latest  manager2  Running        Running 2 minutes ago   
14555mvu3zee6aek4dwonxz3f  web.7   nginx:latest  worker1   Running        Running 2 minutes ago   
1q8imt07i564bm90at3r2w198  web.8   nginx:latest  manager1  Running        Running 2 minutes ago   
encwziari9h78ue32v5pjq9jv  web.9   nginx:latest  worker3   Running        Running 2 minutes ago   
aivwszsjhhpky43t3x7o8ezz9  web.10  nginx:latest  worker2   Running        Running 2 minutes ago   
457fsqomatl1lgd9qbz2dcqsb  web.11  nginx:latest  worker1   Running        Running 2 minutes ago   
7chhofuj4shhqdkwu67512h1b  web.12  nginx:latest  worker2   Running        Running 2 minutes ago   
7dynic159wyouch05fyiskrd0  web.13  nginx:latest  worker1   Running        Running 2 minutes ago   
7zg9eki4610maigr1xwrx7zqk  web.14  nginx:latest  manager3  Running        Running 2 minutes ago   
4z2c9j20gwsasosvj7mkzlyhc  web.15  nginx:latest  manager2  Running        Running 2 minutes ago   
```

You can also drain a particular node, that is remove all services from that node. The services will automatically be rescheduled on other nodes.
```
$ docker-machine ssh manager1 "docker node update --availability drain worker1"
worker1
$ docker-machine ssh manager1 "docker service ps web"
ID                         NAME        IMAGE         NODE      DESIRED STATE  CURRENT STATE           ERROR
61wjx0zaovwtzywwbomnvjo4q  web.1       nginx:latest  worker3   Running        Running 15 minutes ago  
bkkujhpbtqab8fyhah06apvca  web.2       nginx:latest  manager1  Running        Running 4 minutes ago   
09zkslrkgrvbscv0vfqn2j5dw  web.3       nginx:latest  manager1  Running        Running 4 minutes ago   
4dlmy8k72eoza9t4yp9c9pq0w  web.4       nginx:latest  manager2  Running        Running 4 minutes ago   
6yqabr8kajx5em2auvfzvi8wi  web.5       nginx:latest  manager3  Running        Running 4 minutes ago   
21x7sn82883e7oymz57j75q4q  web.6       nginx:latest  manager2  Running        Running 4 minutes ago   
8so0xi55kqimch2jojfdr13qk  web.7       nginx:latest  worker3   Running        Running 3 seconds ago   
14555mvu3zee6aek4dwonxz3f   \_ web.7   nginx:latest  worker1   Shutdown       Shutdown 4 seconds ago  
1q8imt07i564bm90at3r2w198  web.8       nginx:latest  manager1  Running        Running 4 minutes ago   
encwziari9h78ue32v5pjq9jv  web.9       nginx:latest  worker3   Running        Running 4 minutes ago   
aivwszsjhhpky43t3x7o8ezz9  web.10      nginx:latest  worker2   Running        Running 4 minutes ago   
738jlmoo6tvrkxxar4gbdogzf  web.11      nginx:latest  worker2   Running        Running 3 seconds ago   
457fsqomatl1lgd9qbz2dcqsb   \_ web.11  nginx:latest  worker1   Shutdown       Shutdown 3 seconds ago  
7chhofuj4shhqdkwu67512h1b  web.12      nginx:latest  worker2   Running        Running 4 minutes ago   
4h7zcsktbku7peh4o32mw4948  web.13      nginx:latest  manager3  Running        Running 3 seconds ago   
7dynic159wyouch05fyiskrd0   \_ web.13  nginx:latest  worker1   Shutdown       Shutdown 4 seconds ago  
7zg9eki4610maigr1xwrx7zqk  web.14      nginx:latest  manager3  Running        Running 4 minutes ago   
4z2c9j20gwsasosvj7mkzlyhc  web.15      nginx:latest  manager2  Running        Running 4 minutes ago   
```

You can check out the nodes and see that `worker1` is still active but drained.
```
$ docker-machine ssh manager1 "docker node ls"
ID                           HOSTNAME  STATUS  AVAILABILITY  MANAGER STATUS
3cq6idpysa53n6a21nqe0924h    manager3  Ready   Active        Reachable
64swze471iu5silg83ls0bdip *  manager1  Ready   Active        Leader
7eljvvg0icxlw20od5f51oq8t    manager2  Ready   Active        Reachable
8awcmkj3sd9nv1pi77i6mdb1i    worker1   Ready   Drain         
avu80ol573rzepx8ov80ygzxz    worker2   Ready   Active        
bxn1iivy8w7faeugpep76w50j    worker3   Ready   Active
```

You can also scale down the service
```
$ docker-machine ssh manager1 "docker service scale web=10"
web scaled to 10
$ docker-machine ssh manager1 "docker service ps web"
ID                         NAME        IMAGE         NODE      DESIRED STATE  CURRENT STATE            ERROR
61wjx0zaovwtzywwbomnvjo4q  web.1       nginx:latest  worker3   Running        Running 22 minutes ago   
bkkujhpbtqab8fyhah06apvca  web.2       nginx:latest  manager1  Shutdown       Shutdown 54 seconds ago  
09zkslrkgrvbscv0vfqn2j5dw  web.3       nginx:latest  manager1  Running        Running 11 minutes ago   
4dlmy8k72eoza9t4yp9c9pq0w  web.4       nginx:latest  manager2  Running        Running 11 minutes ago   
6yqabr8kajx5em2auvfzvi8wi  web.5       nginx:latest  manager3  Running        Running 11 minutes ago   
21x7sn82883e7oymz57j75q4q  web.6       nginx:latest  manager2  Running        Running 11 minutes ago   
8so0xi55kqimch2jojfdr13qk  web.7       nginx:latest  worker3   Running        Running 7 minutes ago    
14555mvu3zee6aek4dwonxz3f   \_ web.7   nginx:latest  worker1   Shutdown       Shutdown 7 minutes ago   
1q8imt07i564bm90at3r2w198  web.8       nginx:latest  manager1  Running        Running 11 minutes ago   
encwziari9h78ue32v5pjq9jv  web.9       nginx:latest  worker3   Shutdown       Shutdown 54 seconds ago  
aivwszsjhhpky43t3x7o8ezz9  web.10      nginx:latest  worker2   Shutdown       Shutdown 54 seconds ago  
738jlmoo6tvrkxxar4gbdogzf  web.11      nginx:latest  worker2   Running        Running 7 minutes ago    
457fsqomatl1lgd9qbz2dcqsb   \_ web.11  nginx:latest  worker1   Shutdown       Shutdown 7 minutes ago   
7chhofuj4shhqdkwu67512h1b  web.12      nginx:latest  worker2   Running        Running 11 minutes ago   
4h7zcsktbku7peh4o32mw4948  web.13      nginx:latest  manager3  Running        Running 7 minutes ago    
7dynic159wyouch05fyiskrd0   \_ web.13  nginx:latest  worker1   Shutdown       Shutdown 7 minutes ago   
7zg9eki4610maigr1xwrx7zqk  web.14      nginx:latest  manager3  Shutdown       Shutdown 54 seconds ago  
4z2c9j20gwsasosvj7mkzlyhc  web.15      nginx:latest  manager2  Shutdown       Shutdown 54 seconds ago  
```

Now bring `worker1` back online and show it's new availability
```
$ docker-machine ssh manager1 "docker node update --availability active worker1"
worker1
$ docker-machine ssh manager1 "docker node inspect worker1 --pretty"
ID:			8awcmkj3sd9nv1pi77i6mdb1i
Hostname:		worker1
Joined at:		2016-08-23 22:30:15.556517377 +0000 utc
Status:
 State:			Ready
 Availability:		Active
Platform:
 Operating System:	linux
 Architecture:		x86_64
Resources:
 CPUs:			1
 Memory:		995.9 MiB
Plugins:
  Network:		bridge, host, null, overlay
  Volume:		local
Engine Version:		17.03.0-ce
Engine Labels:
 - provider = virtualbox
 ```

Now let's take the manager1 node, the leader, out of the Swarm
```
$ docker-machine ssh manager1 "docker swarm leave --force"
Node left the swarm.
```

Wait about 30 seconds just to be sure. The Swarm still functions, but must elect a new leader. This happens automatically.
```
$ docker-machine ssh manager2 "docker node ls"
ID                           HOSTNAME  STATUS  AVAILABILITY  MANAGER STATUS
3cq6idpysa53n6a21nqe0924h    manager3  Ready   Active        Reachable
64swze471iu5silg83ls0bdip    manager1  Down    Active        Unreachable
7eljvvg0icxlw20od5f51oq8t *  manager2  Ready   Active        Leader
8awcmkj3sd9nv1pi77i6mdb1i    worker1   Ready   Active        
avu80ol573rzepx8ov80ygzxz    worker2   Ready   Active        
bxn1iivy8w7faeugpep76w50j    worker3   Ready   Active
```
You see that `manager1` is Down and Unreachable and `manager2` has been elected leader. It's also easy to remove a service:
```
$ docker-machine ssh manager2 "docker service rm web"
web
```

## Cleanup
There's also a [bash script](https://github.com/ManoMarks/labs/blob/master/swarm-mode/beginner-tutorial/swarm-node-vbox-teardown.sh) that will clean up your machine by removing all the Docker Machines.

```
$ ./swarm-node-vbox-teardown.sh
Stopping "manager3"...
Stopping "manager2"...
Stopping "worker1"...
Stopping "manager1"...
Stopping "worker3"...
Stopping "worker2"...
Machine "manager3" was stopped.
Machine "manager1" was stopped.
Machine "manager2" was stopped.
Machine "worker2" was stopped.
Machine "worker1" was stopped.
Machine "worker3" was stopped.
About to remove worker1, worker2, worker3, manager1, manager2, manager3
Are you sure? (y/n): y
Successfully removed worker1
Successfully removed worker2
Successfully removed worker3
Successfully removed manager1
Successfully removed manager2
Successfully removed manager3
```  

## Next steps
Check out the documentation on [Docker Swarm Mode](https://docs.docker.com/engine/swarm/) for more information.
