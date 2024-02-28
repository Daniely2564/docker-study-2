- docker container top - process list in one container
- docker container inspect - details of one container config
- docker container stats - performance stats for all containers

## Networking

In order to check port forwarding you can use the command

```sh
docker container port <<docker-job-name>>
```

**Example output**

```
80/tcp -> 0.0.0.0:80
```

But if you want to find out the actual ip address, you can query from json

```sh
docker container inspect --format '{{ .NetworkSettings.IPAddress }}' <<docker-job>>
```

**Example output**

```
172.17.0.2
```

### Docker Networks: CLI Management

- Show networks `docker network ls`
- Inspect a network `docker network inspect`
- Create a network `docker network create --driver`
- Attach a network to container `docker network connect`
- Detach a network from container `docker network disconnect`

#### docker network ls

```
NETWORK ID     NAME      DRIVER    SCOPE
b74cf644fe41   bridge    bridge    local
58e37240879d   host      host      local
c6a45e500a51   none      null      local
```

Show all of the networks that have been created.

Network `bridge` here is the default Docker virtual network, which is NAT'ed behind the Host IP.

#### docker network inspect **network-name**

Let us run the `docker network inspect bridge`.

```
[
    {
        "Name": "bridge",
        "Id": "b74cf644fe410231f77b76a45a91530cfcaa0da786dd947f9ed8ec83a8b37625",
        "Created": "2023-06-21T02:27:44.603188814Z",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "172.17.0.0/16",
                    "Gateway": "172.17.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {
            "4e12b5d88fb3c1723ca934573e5f8c18b00813f4e1613c3c198798bb238a3c16": {
                "Name": "nginx",
                "EndpointID": "b404c0533450b6a3b3a5657106222731d51a4cfa8327f518b181d5b5576ce7a8",
                "MacAddress": "02:42:ac:11:00:02",
                "IPv4Address": "172.17.0.2/16",
                "IPv6Address": ""
            }
        },
        "Options": {
            "com.docker.network.bridge.default_bridge": "true",
            "com.docker.network.bridge.enable_icc": "true",
            "com.docker.network.bridge.enable_ip_masquerade": "true",
            "com.docker.network.bridge.host_binding_ipv4": "0.0.0.0",
            "com.docker.network.bridge.name": "docker0",
            "com.docker.network.driver.mtu": "1500"
        },
        "Labels": {}
    }
]
```

We can see `nginx` is attached to the `bridge` network under **Containers**.

```
 "Containers": {
    "4e12b5d88fb3c1723ca934573e5f8c18b00813f4e1613c3c198798bb238a3c16": {
        "Name": "nginx",
        "EndpointID": "b404c0533450b6a3b3a5657106222731d51a4cfa8327f518b181d5b5576ce7a8",
        "MacAddress": "02:42:ac:11:00:02",
        "IPv4Address": "172.17.0.2/16",
        "IPv6Address": ""
    }
},
```

If you take a look at IPAM config

```
"IPAM": {
    "Driver": "default",
    "Options": null,
    "Config": [
        {
            "Subnet": "172.17.0.0/16",
            "Gateway": "172.17.0.1"
        }
    ]
},
```

Subnet defaults to 172.17.0.1 and subnet has 16

**Network (host)**

Host network gains performance by skipping virtual networks but sacrifices security of container model. Prevents the security boundaries of the containerization protecting the interface of that container. But also in certain situtations, you can benefit performance.

**Network (none)**

Removes eth0 and only leaves you with localhost interface in container. Having an interface that is not attached to anything.

#### docker network create **network-name**

You can create your own network using the command `docker network create <network-name>`. Once you do, if you ls, you should see

```
NETWORK ID     NAME         DRIVER    SCOPE
b74cf644fe41   bridge       bridge    local
58e37240879d   host         host      local
07d9bb080f27   my_app_net   bridge    local
c6a45e500a51   none         null      local
```

You will ssee a new network named `my_app_net` has been created. You should also take a look at options to by using the --help command

```
$ docker network create --help

Usage:  docker network create [OPTIONS] NETWORK

Create a network

Options:
      --attachable           Enable manual container attachment
      --aux-address map      Auxiliary IPv4 or IPv6 addresses used by Network driver (default map[])
      --config-from string   The network from which to copy the configuration
      --config-only          Create a configuration only network
  -d, --driver string        Driver to manage the Network (default "bridge")
      --gateway strings      IPv4 or IPv6 Gateway for the master subnet
      --ingress              Create swarm routing-mesh network
      --internal             Restrict external access to the network
      --ip-range strings     Allocate container ip from a sub-range
      --ipam-driver string   IP Address Management Driver (default "default")
      --ipam-opt map         Set IPAM driver specific options (default map[])
      --ipv6                 Enable IPv6 networking
      --label list           Set metadata on a network
  -o, --opt map              Set driver specific options (default map[])
      --scope string         Control the network's scope
      --subnet strings       Subnet in CIDR format that represents a network segment
```

When you create a new docker container, you can attach your network using `--network` flag.

```
$ docker container run -d --name new_nginx --network my_app_net nginx
```

Once you run and run the command `docker network inspect my_app_net` you will see `new_nginx` has been added to the **Container**.

```
$ docker network inspect my_app_net

[
    {
        "Name": "my_app_net",
        "Id": "07d9bb080f27b369566838358e82b8fcdde382c51bf7e782f9e49853da7b1d76",
        "Created": "2023-06-21T02:40:11.345232366Z",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "172.18.0.0/16",
                    "Gateway": "172.18.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {
            "f5ed3b8925937674566c0eeaf5075143b20cadea63ef2d89c93cf79796dea468": {
                "Name": "new_nginx",
                "EndpointID": "b186166dc3e6d1b6545acf2df3504041d54f1bfb4f10ee8824fb948dd69b2a03",
                "MacAddress": "02:42:ac:12:00:02",
                "IPv4Address": "172.18.0.2/16",
                "IPv6Address": ""
            }
        },
        "Options": {},
        "Labels": {}
    }
]
```

#### docker network connect \<new-network-id> \<container-id>

Our networks

```
$ docker network ls

NETWORK ID     NAME         DRIVER    SCOPE
b74cf644fe41   bridge       bridge    local
58e37240879d   host         host      localb
07d9bb080f27   my_app_net   bridge    local
c6a45e500a51   none         null      local
```

Our containers

```
$ docker container ls

CONTAINER ID   IMAGE     COMMAND                  CREATED         STATUS          PORTS                NAMES
f5ed3b892593   nginx     "/docker-entrypoint.…"   3 minutes ago   Up 3 minutes    80/tcp               new_nginx
4e12b5d88fb3   nginx     "/docker-entrypoint.…"   6 hours ago     Up 15 minutes   0.0.0.0:80->80/tcp   nginx
```

Using the network id and container id, we can connect bridge network to the new_nginx

```
$ docker network connect b74cf644fe41 f5ed3b892593
```

Now if we inspect the container, we should see two networks attached

```
[
    {
            ...
            "Networks": {
                "bridge": {
                    "IPAMConfig": {},
                    "Links": null,
                    "Aliases": [],
                    "NetworkID": "b74cf644fe410231f77b76a45a91530cfcaa0da786dd947f9ed8ec83a8b37625",
                    "EndpointID": "62ecc3694d74d667f9e3ed2a3028897c227d969f6be65a2b57872b2831ae5f53",
                    "Gateway": "172.17.0.1",
                    "IPAddress": "172.17.0.3",
                    "IPPrefixLen": 16,
                    "IPv6Gateway": "",
                    "GlobalIPv6Address": "",
                    "GlobalIPv6PrefixLen": 0,
                    "MacAddress": "02:42:ac:11:00:03",
                    "DriverOpts": {}
                },
                "my_app_net": {
                    "IPAMConfig": null,
                    "Links": null,
                    "Aliases": [
                        "f5ed3b892593"
                    ],
                    "NetworkID": "07d9bb080f27b369566838358e82b8fcdde382c51bf7e782f9e49853da7b1d76",
                    "EndpointID": "b186166dc3e6d1b6545acf2df3504041d54f1bfb4f10ee8824fb948dd69b2a03",
                    "Gateway": "172.18.0.1",
                    "IPAddress": "172.18.0.2",
                    "IPPrefixLen": 16,
                    "IPv6Gateway": "",
                    "GlobalIPv6Address": "",
                    "GlobalIPv6PrefixLen": 0,
                    "MacAddress": "02:42:ac:12:00:02",
                    "DriverOpts": null
                }
            }
        }
    }
]
```

#### docker network connect \<new-network-id> \<container-id>

You can disconnect the same way you connect and you will no longer have the network.

The end goal here is that if you are running all your apps on the same docker network, you won't be able to protect them. We most of time expose more than what we need to.

### DNS

- Understand how DNS is the key to easy inter-container communications
- `--link` to enable DNS on default bridge network

#### Problem with IP Address's

- Static IP's and using IP's for talking to containers is an anti-pattern.

Solution

#### Docker DNS

##### DNS Default Names

Docker defaults the hostname to the container's name, but you can also set aliases

Let's inspect the `my_app_net` we created last time.

```
$ docker network inspect my_app_net

[{
    ...
     "Containers": {
        "f5ed3b8925937674566c0eeaf5075143b20cadea63ef2d89c93cf79796dea468": {
            "Name": "new_nginx",
            "EndpointID": "e93e302e09cb0c818bd84065afbda009f33e99e17efe4e073a1a8ea88f29f7d4",
            "MacAddress": "02:42:ac:12:00:02",
            "IPv4Address": "172.18.0.2/16",
            "IPv6Address": ""
        }
    },
    ...
}]
```

Now let's create a new container.

```
$ docker container run -d --name my_nginx --network my_app_net nginx
```

We can see network has two containers now.

```
$ docker network inspect my_app_net

[{
    ...
    "Containers": {
        "1cfa894099f3ddf486cffe86490a537fa58f2d491f039da94ec4f8e7b989b296": {
            "Name": "my_nginx",
            "EndpointID": "0f2243f3695757bbf2c1db248f2f6817f0ee9c4bdf8fbd27e2747a66aa866a07",
            "MacAddress": "02:42:ac:12:00:03",
            "IPv4Address": "172.18.0.3/16",
            "IPv6Address": ""
        },
        "f5ed3b8925937674566c0eeaf5075143b20cadea63ef2d89c93cf79796dea468": {
            "Name": "new_nginx",
            "EndpointID": "e93e302e09cb0c818bd84065afbda009f33e99e17efe4e073a1a8ea88f29f7d4",
            "MacAddress": "02:42:ac:12:00:02",
            "IPv4Address": "172.18.0.2/16",
            "IPv6Address": ""
        }
    },
    "Options": {},
    "Labels": {}
}
    ...
}]
```

Surprisingly, docker will automatically set up dns for ip address of containers you've created. So from my_nginx, I can ping to new_nginx and vice versa.

Let us execute a command to ping to the network of newly created `new_nginx`

```
$ docker container exec -it my_nginx ping new_nginx

PING new_nginx (172.18.0.2) 56(84) bytes of data.
64 bytes from new_nginx.my_app_net (172.18.0.2): icmp_seq=1 ttl=64 time=0.089 ms
64 bytes from new_nginx.my_app_net (172.18.0.2): icmp_seq=2 ttl=64 time=0.083 ms
64 bytes from new_nginx.my_app_net (172.18.0.2): icmp_seq=3 ttl=64 time=0.089 ms
64 bytes from new_nginx.my_app_net (172.18.0.2): icmp_seq=4 ttl=64 time=0.076 ms
64 bytes from new_nginx.my_app_net (172.18.0.2): icmp_seq=5 ttl=64 time=0.078 ms
64 bytes from new_nginx.my_app_net (172.18.0.2): icmp_seq=6 ttl=64 time=0.080 ms
64 bytes from new_nginx.my_app_net (172.18.0.2): icmp_seq=7 ttl=64 time=0.075 ms
```

Let's again look at networks

```
$ docker network ls

NETWORK ID     NAME         DRIVER    SCOPE
abe229e9c9be   bridge       bridge    local
58e37240879d   host         host      local
07d9bb080f27   my_app_net   bridge    local
c6a45e500a51   none         null      local
```

Unfortunately, the default network doesn't have the DNS built-in. So you can use `--link`.

```
$ docker container create --help

...
--link list                      Add link to another container
...
```

But it is much easier to create a new network and add containers to the network.

Compose. Automatically create new virtual network.
