# Start 

``` bash
docker-compose up -d
```

# Create topic with default broker values (rf = 5 (3 + 2 observers), min.insync.replicas = 2)

``` bash
docker exec -it zookeeper-1 bash

kafka-topics --bootstrap-server broker-avivm281:19091 --create --topic test --partitions 3
kafka-topics --bootstrap-server broker-avivm281:19091 --topic test --describe
```

**Try to stop 2 followers and watch how observer became a follower present in the ISR list**

# Create topic where each partition is present on each broker (RF = 6 (4 + 2 observers), min.insync.replicas = 3)

Open a new terminal :

``` bash
docker exec -it broker-avivm281 bash

kafka-topics --bootstrap-server broker-avivm281:19091 --topic test2 --create --partitions 3 --replica-placement /etc/kafka/demo/replica-placement-full.json --config min.insync.replicas=3
kafka-topics --bootstrap-server broker-avivm281:19091 --topic test2 --describe

```

More documentations :
- https://docs.confluent.io/platform/current/multi-dc-deployments/multi-region.html#mrr-replica-placement
- https://docs.confluent.io/platform/current/tutorials/examples/multiregion/docs/multiregion.html#replica-placement
