# metadata-proxy


`metadata-proxy` is a metadata server support the following features:

* **self** semantic support. metadata-proxy keep a mapping of IP and metadata, client direct request "/self", will get the metadata of this node. mapping settings is store to backend (etcd).
* metadata backend support [etcd](https://github.com/coreos/etcd) (TODO support more).
* support metadata local cache, so it can be used as a proxy to reducing the request pressure of backend (etcd).
* api out format support json/yaml/text,and is metadata/developer friendly data structure.
* support as [confd](https://github.com/kelseyhightower/confd) backend(TODO).

[中文文档](README_zh.md)

## Building

Go 1.6 is required to build confd, which uses the new vendor directory.

```
$ mkdir -p $GOPATH/src/github.com/yunify
$ git clone https://github.com/yunify/metadata-proxy.git $GOPATH/src/github.com/yunify/metadata-proxy
$ cd $GOPATH/src/github.com/yunify/metadata-proxy
$ ./build
```

You should now have metadata-proxy in your `bin/` directory:

```
$ ls bin/
metadata-proxy
```

## Manage API

Manage API default port is 127.0.0.1:8112

* /v1/mapping[/{nodePath}] manage metadata and ip mapping
    * GET show mapping config.
    * POST create or replace mapping config. 
    * PUT create or merge update mapping config.
    * DELETE delete mapping config.
* /v1/resync resync metadata and mapping from backend. method: POST

## Getting Started

* start etcd

```
etcd
```

* start metadata-proxy

```
bin/metadata-proxy --backend etcdv3 --nodes 127.0.0.1:2379 --log_level debug --listen :8080 --xff true
```

* set etcd version

```
export ETCDCTL_API=3
```

* fill data to etcd

```
for i in `seq 1 5`; 
do  
    etcdctl put /nodes/$i/name node$i; 
    etcdctl put /nodes/$i/ip 192.168.1.$i;
done
```

* show data

by etcdctl

```
etcdctl get / --prefix

/nodes/1/ip
192.168.1.1
/nodes/1/name
node1
/nodes/2/ip
192.168.1.2
/nodes/2/name
node2
/nodes/3/ip
192.168.1.3
/nodes/3/name
node3
/nodes/4/ip
192.168.1.4
/nodes/4/name
node4
/nodes/5/ip
192.168.1.5
/nodes/5/name
node5

```


by metadata-proxy text output

```
curl http://127.0.0.1:8080/

nodes/
```

by metadata-proxy json output

```
curl -H "Accept: application/json" http://127.0.0.1:8080/

{
    "nodes": {
        "1": {
            "ip": "192.168.1.1",
            "name": "node1"
        },
        "2": {
            "ip": "192.168.1.2",
            "name": "node2"
        },
        "3": {
            "ip": "192.168.1.3",
            "name": "node3"
        },
        "4": {
            "ip": "192.168.1.4",
            "name": "node4"
        },
        "5": {
            "ip": "192.168.1.5",
            "name": "node5"
        }
    }
}
```

by metadata-proxy yaml output

```
curl -H "Accept: application/yaml" http://127.0.0.1:8080/

nodes:
  "1":
    ip: 192.168.1.1
    name: node1
  "2":
    ip: 192.168.1.2
    name: node2
  "3":
    ip: 192.168.1.3
    name: node3
  "4":
    ip: 192.168.1.4
    name: node4
  "5":
    ip: 192.168.1.5
    name: node5
```

mapping create

```

curl -H "Content-Type: application/json" -X POST http://127.0.0.1:8112/v1/mapping -d '{"192.168.1.1":{"node":"/nodes/1"}}'

OK
```

show mapping

```

curl -H "Accept: application/json" http://127.0.0.1:8112/v1/mapping

{"192.168.1.1":{"node":"/nodes/1"}}
```

self request

```
curl -H "X-Forwarded-For: 192.168.1.1" http://127.0.0.1:8080/

nodes/
self/

url -H "Accept: application/json" -H "X-Forwarded-For: 192.168.1.1" http://127.0.0.1:8080/self/node


{"ip":"192.168.1.1","name":"node1"}

```

update mapping

```
curl -H "Content-Type: application/json" -X PUT http://127.0.0.1:8112/v1/mapping/192.168.1.1 -d '{"nodes":"/nodes"}'

OK

curl -H "Accept: application/json" http://127.0.0.1:8112/v1/mapping

{"192.168.1.1":{"node":"/nodes/1","nodes":"/nodes"}}


curl -H "Content-Type: application/json" -X PUT http://127.0.0.1:8112/v1/mapping/192.168.1.1/node2 -d '"/nodes/2"'

OK


curl -H "Accept: application/json" http://127.0.0.1:8112/v1/mapping

{"192.168.1.1":{"node":"/nodes/1","node2":"/nodes/2","nodes":"/nodes"}}
```

delete mapping

```
curl -X DELETE http://127.0.0.1:8112/v1/mapping/192.168.1.1/node2

OK

curl -H "Accept: application/json" http://127.0.0.1:8112/v1/mapping

{"192.168.1.1":{"node":"/nodes/1","nodes":"/nodes"}}


curl -X DELETE http://127.0.0.1:8112/v1/mapping/192.168.1.1

OK

curl -H "Accept: application/json" http://127.0.0.1:8112/v1/mapping

{}
```
