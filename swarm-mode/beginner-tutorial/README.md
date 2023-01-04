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

```
root@docker2:~# docker node ls
ID                            HOSTNAME   STATUS    AVAILABILITY   MANAGER STATUS   ENGINE VERSION
89hm11snya6gducfwadhoqn2h     docker1    Ready     Active                          20.10.21
uhz3opjuii09xl8ern9ip7d5p *   docker2    Ready     Active         Leader           20.10.21
```

4. The next step is to create a service and list out the services. This creates a single service called `web` that runs the latest nginx:
```
$ docker service create -p 80:80 --name web nginx:latest
$ docker service ls
ID            NAME  REPLICAS  IMAGE         COMMAND
2x4jsk6313az  web   1/1       nginx:latest  
```
Now open the machine's IP address in your browser. You can see above manager1 had an IP address of 192.168.99.100
![nginx in Chrome at 192.168.99.100](images/manager1-nginx.png)

You can actually load any of the node ip addresses and get the same result because of [Swarm Mode's Routing Mesh](https://docs.docker.com/engine/swarm/ingress/).
![nginx in Chrome at 192.168.99.100](images/manager2-nginx.png)

Next let's inspect the service
```
$ docker service inspect web
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
$ docker service scale web=15
web scaled to 15
$ docker service ls
ID            NAME  REPLICAS  IMAGE         COMMAND
2x4jsk6313az  web   15/15     nginx:latest  
```
Docker has spread the 15 services evenly over all of the nodes
```
$ docker service ps web
ID             NAME      IMAGE          NODE      DESIRED STATE   CURRENT STATE            ERROR     PORTS
i5x472eny11n   web.1     nginx:latest   docker2   Running         Running 10 minutes ago             
nyb5yrdp4c72   web.2     nginx:latest   docker1   Running         Running 16 seconds ago             
qd9mhec8594t   web.3     nginx:latest   docker2   Running         Running 37 seconds ago             
vbtkh09x9e15   web.4     nginx:latest   docker1   Running         Running 14 seconds ago             
v9v833kbwpyy   web.5     nginx:latest   docker2   Running         Running 40 seconds ago             
jmvril7o0sjy   web.6     nginx:latest   docker1   Running         Running 15 seconds ago             
ya4rjbiqduor   web.7     nginx:latest   docker1   Running         Running 15 seconds ago             
ycwl952syned   web.8     nginx:latest   docker1   Running         Running 19 seconds ago             
bmzup7ex4lhd   web.9     nginx:latest   docker2   Running         Running 37 seconds ago             
qrth6o0c620l   web.10    nginx:latest   docker2   Running         Running 39 seconds ago             
o9jc3hmwpgw4   web.11    nginx:latest   docker1   Running         Running 14 seconds ago             
lmdzwk8kg7ih   web.12    nginx:latest   docker2   Running         Running 38 seconds ago             
f6er19lbhe8e   web.13    nginx:latest   docker1   Running         Running 15 seconds ago             
pmk8kdifj211   web.14    nginx:latest   docker1   Running         Running 16 seconds ago             
iuh3bet55ko3   web.15    nginx:latest   docker2   Running         Running 38 seconds ago    
```

You can also drain a particular node, that is remove all services from that node. The services will automatically be rescheduled on other nodes.
```
$ docker node update --availability drain docker1
docker1
$ docker service ps web
ID             NAME         IMAGE          NODE      DESIRED STATE   CURRENT STATE                    ERROR     PORTS
i5x472eny11n   web.1        nginx:latest   docker2   Running         Running 11 minutes ago                     
mzom8b3qk1jh   web.2        nginx:latest   docker2   Running         Running 1 second ago                       
nyb5yrdp4c72    \_ web.2    nginx:latest   docker1   Shutdown        Shutdown 13 seconds ago                    
qd9mhec8594t   web.3        nginx:latest   docker2   Running         Running about a minute ago                 
i6zw74qmdbiq   web.4        nginx:latest   docker2   Running         Running 4 seconds ago                      
vbtkh09x9e15    \_ web.4    nginx:latest   docker1   Shutdown        Shutdown 14 seconds ago                    
v9v833kbwpyy   web.5        nginx:latest   docker2   Running         Running about a minute ago                 
zomt10exdaiw   web.6        nginx:latest   docker2   Running         Running 1 second ago                       
jmvril7o0sjy    \_ web.6    nginx:latest   docker1   Shutdown        Shutdown 13 seconds ago                    
ymx02b2ba72z   web.7        nginx:latest   docker2   Running         Running less than a second ago             
ya4rjbiqduor    \_ web.7    nginx:latest   docker1   Shutdown        Shutdown 13 seconds ago                    
642vjmyrdqm5   web.8        nginx:latest   docker2   Running         Running less than a second ago             
ycwl952syned    \_ web.8    nginx:latest   docker1   Shutdown        Shutdown 12 seconds ago                    
bmzup7ex4lhd   web.9        nginx:latest   docker2   Running         Running about a minute ago                 
qrth6o0c620l   web.10       nginx:latest   docker2   Running         Running about a minute ago                 
awz9m9og5mck   web.11       nginx:latest   docker2   Running         Running less than a second ago             
o9jc3hmwpgw4    \_ web.11   nginx:latest   docker1   Shutdown        Shutdown 13 seconds ago                    
lmdzwk8kg7ih   web.12       nginx:latest   docker2   Running         Running about a minute ago                 
yocsjxorx40t   web.13       nginx:latest   docker2   Running         Running 2 seconds ago                      
f6er19lbhe8e    \_ web.13   nginx:latest   docker1   Shutdown        Shutdown 13 seconds ago                    
gofssv1kxqj7   web.14       nginx:latest   docker2   Running         Running 2 seconds ago                      
pmk8kdifj211    \_ web.14   nginx:latest   docker1   Shutdown        Shutdown 14 seconds ago                    
iuh3bet55ko3   web.15       nginx:latest   docker2   Running         Running about a minute ago
```

You can check out the nodes and see that `docker1` is still active but drained.
```
$ docker node ls
ID                            HOSTNAME   STATUS    AVAILABILITY   MANAGER STATUS   ENGINE VERSION
89hm11snya6gducfwadhoqn2h     docker1    Ready     Drain                           20.10.21
uhz3opjuii09xl8ern9ip7d5p *   docker2    Ready     Active         Leader           20.10.21

```

You can also scale down the service
```
$ docker service scale web=10
web scaled to 10
$ docker service ps web
ID             NAME        IMAGE          NODE      DESIRED STATE   CURRENT STATE                ERROR     PORTS
i5x472eny11n   web.1       nginx:latest   docker2   Running         Running 13 minutes ago                 
mzom8b3qk1jh   web.2       nginx:latest   docker2   Running         Running about a minute ago             
nyb5yrdp4c72    \_ web.2   nginx:latest   docker1   Shutdown        Shutdown 2 minutes ago                 
qd9mhec8594t   web.3       nginx:latest   docker2   Running         Running 3 minutes ago                  
i6zw74qmdbiq   web.4       nginx:latest   docker2   Running         Running about a minute ago             
vbtkh09x9e15    \_ web.4   nginx:latest   docker1   Shutdown        Shutdown 2 minutes ago                 
v9v833kbwpyy   web.5       nginx:latest   docker2   Running         Running 3 minutes ago                  
zomt10exdaiw   web.6       nginx:latest   docker2   Running         Running about a minute ago             
jmvril7o0sjy    \_ web.6   nginx:latest   docker1   Shutdown        Shutdown 2 minutes ago                 
ymx02b2ba72z   web.7       nginx:latest   docker2   Running         Running about a minute ago             
ya4rjbiqduor    \_ web.7   nginx:latest   docker1   Shutdown        Shutdown 2 minutes ago                 
642vjmyrdqm5   web.8       nginx:latest   docker2   Running         Running about a minute ago             
ycwl952syned    \_ web.8   nginx:latest   docker1   Shutdown        Shutdown 2 minutes ago                 
bmzup7ex4lhd   web.9       nginx:latest   docker2   Running         Running 3 minutes ago                  
qrth6o0c620l   web.10      nginx:latest   docker2   Running         Running 3 minutes ago                  
o9jc3hmwpgw4   web.11      nginx:latest   docker1   Shutdown        Shutdown 2 minutes ago                 
f6er19lbhe8e   web.13      nginx:latest   docker1   Shutdown        Shutdown 2 minutes ago                 
pmk8kdifj211   web.14      nginx:latest   docker1   Shutdown        Shutdown 2 minutes ago   
```

Now bring `docker1` back online and show it's new availability
```
$ docker node update --availability active docker1
docker1
$ docker node inspect docker1 --pretty
ID:			89hm11snya6gducfwadhoqn2h
Hostname:              	docker1
Joined at:             	2022-12-21 11:13:50.199877539 +0000 utc
Status:
 State:			Ready
 Availability:         	Active
 Address:		10.0.0.37
Platform:
 Operating System:	linux
 Architecture:		x86_64
Resources:
 CPUs:			2
 Memory:		964.2MiB
Plugins:
 Log:		awslogs, fluentd, gcplogs, gelf, journald, json-file, local, logentries, splunk, syslog
 Network:		bridge, host, ipvlan, macvlan, null, overlay
 Volume:		local
Engine Version:		20.10.21
TLS Info:
 TrustRoot:
-----BEGIN CERTIFICATE-----

 ```
Let's promote docker1:
```
$ docker node promote docker1
```
Now let's take the manager1 node, the leader, out of the Swarm
```
It's also easy to remove a service:
```
$ docker service rm web
web
```
## Next steps
Check out the documentation on [Docker Swarm Mode](https://docs.docker.com/engine/swarm/) for more information.
