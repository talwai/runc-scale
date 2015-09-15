### Treat this README as the spec of an unstarted project
See Tom Preston-Werner's [README-driven development](http://tom.preston-werner.com/2010/08/23/readme-driven-development.html) if you're curious why

`runc-scale` is a self-governing, single-node automation agent for the scale up/down of runC containers.
It uses Facebook's `osquery` for system monitoring and for evaluating scaling conditions

### Dependencies
- go 1.3+
- python 2.7+
- [osquery](https://github.com/facebook/osquery)

### Installation
```bash
$ go get github.com/talwai/runc-scale
$ cd $GOPATH/src/github.com/talwai/runc-scale
$ make
$ sudo make install
$ runc-scale --version
0.0.1
```

### Usage

#### Bootstrapping
First [install runc](https://github.com/opencontainers/runc/blob/master/README.md)
Then run:
```bash
$ runc-scale init
Aliasing runc in ~/.bash_profile
Done
```

To confirm success:
```bash
$ runc verify-scale
All systems operational
```

#### Add a runc-scale.yml
Here's an example config:
```yaml
- cluster_name: my_redis_cluster  # A unique identifier for your scaling cluster
  scale_metrics:
      - key : resident_set_size # An identifier for a single scaling metric, unique to this cluster
        query : "select sum(resident_size) from processes where name like 'redis%'" # An osquery query to execute
        target: "/home/redis_container" # Path to the runc image to target for this scaling operation.
                                                      # The path must contain a `config.json` compliant with the OCF-spec

        # Scaling conditions (simple booleans that are evaluated with the result of `query` as the LHS)
        up:
            condition: "> 10000000"
        down:
            evaluator: "< 50000"
```

#### Start a single-node runc cluster
Follow the instructions in the [runc README](https://github.com/opencontainers/runc/blob/master/README.md)
to setup a `rootfs` for a container and a valid `config.json`

The rest of this doc will assume these live under `/home/redis-container`:

Start a single-node cluster with
`$ runc start --id redis_container_1 --cluster my_redis_cluster --scale-config /path/to/runc-scale.yml`
The `--cluster` flag is crucial as it tells `runc-scale` which scaling config in `runc-scale.yml` to match your single-node cluster against


#### View Cluster Status
```bash
$ runc scale-status my_redis_cluster
1 container(s) running for target /home/redis_container:
 - redis_container_1
```

#### Trigger a scaling operation
Let's put our redis container under some heavy load so that it triggers the scale-up

```
TODO: Memory-intensive operations against redis_container_1
```

In a separate tab run :

```bash
tail -f /var/log/runc-scale/agent.log

### You may have to wait a second for the scaling action to trigger
...
### But eventually you should see
Executing runc scale up --target=/home/redis-container
Bringing up container redis_container_2 ...
Success!
```


#### Manually triggering a scaling operation
##### Up
```bash
$ runc scale up --target=/home/redis-container
Bringing up container redis_container_2 ...
Success!
$ runc scale-status my_redis_cluster
2 container(s) running for target /home/redis_container:
 - redis_container_1
 - redis_container_2
```

##### Down
```bash
$ runc scale down --target=/home/redis-container
Bringing down container redis_container_2 ...
Success!
$ runc scale-status my_redis_cluster
1 container(s) running for target /home/redis_container:
 - redis_container_1
```
