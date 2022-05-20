# Initial setup cluster using Automatic Observer Promotion

``` bash
docker-compose up -d
```

## Create topic with default broker values (rf = 5 (3 + 2 observers), min.insync.replicas = 2)

``` bash
docker exec -it zookeeper-1 bash

kafka-topics --bootstrap-server broker-avivm281:19091 --list
kafka-topics --bootstrap-server broker-avivm281:19091 --create --topic test --partitions 3
kafka-topics --bootstrap-server broker-avivm281:19091 --topic test --describe
```

**Try to stop 2 followers and watch how observer became a follower present in the ISR list**

## Create topic where each partition is present on each broker (RF = 6 (4 + 2 observers), min.insync.replicas = 3)

Open a new terminal :

``` bash
docker exec -it broker-avivm281 bash

kafka-topics --bootstrap-server broker-avivm281:19091 --topic test2 --create --partitions 3 --replica-placement /etc/kafka/demo/replica-placement-full.json --config min.insync.replicas=3
kafka-topics --bootstrap-server broker-avivm281:19091 --topic test2 --describe

```

More documentations :
- https://docs.confluent.io/platform/current/multi-dc-deployments/multi-region.html#mrr-replica-placement
- https://docs.confluent.io/platform/current/tutorials/examples/multiregion/docs/multiregion.html#replica-placement


Stop your cluster

``` bash
docker-compose down -v
```

# Upgrading cluster

## Upgrade cluster without constraint placement and apply a new constraint with kafka-configs (not works)

Start the kafka stack

``` bash
docker-compose -f docker-compose-not-placement.yml up -d
```

Connect to broker-avivm281 

``` bash
docker exec -it broker-avivm281 bash
```

Create topics and describe it

```
$ kafka-topics --bootstrap-server broker-avivm281:19091 --create --topic test --partitions 3 --replication-factor 3

>> Created topic test.

$ kafka-topics --bootstrap-server broker-avivm281:19091 --topic test --describe

>> Topic: test     TopicId: JPBEenqYSpeC2jizo1Xacg PartitionCount: 3       ReplicationFactor: 3    Configs: min.insync.replicas=2
        Topic: test     Partition: 0    Leader: 4       Replicas: 4,1,2 Isr: 4,1,2      Offline: 
        Topic: test     Partition: 1    Leader: 2       Replicas: 2,4,3 Isr: 2,4,3      Offline: 
        Topic: test     Partition: 2    Leader: 3       Replicas: 3,2,1 Isr: 3,2,1      Offline: 
```

Uncomment all lines on `docker-compose-not-placement.yml` file

Restart your cluster

``` bash
docker-compose -f docker-compose-not-placement.yml up -d
```

Wait some seconds that cluster is UP and all partitions on test topic is rebalanced

Connect to broker-avivm281 

``` bash
docker exec -it broker-avivm281 bash
```

Describe test topics and check if assigment list looks good

``` bash
$ kafka-topics --bootstrap-server broker-avivm281:19091 --topic test --describe

>> Topic: test     TopicId: JPBEenqYSpeC2jizo1Xacg PartitionCount: 3       ReplicationFactor: 3    Configs: min.insync.replicas=2
        Topic: test     Partition: 0    Leader: 4       Replicas: 4,1,2 Isr: 4,1,2      Offline: 
        Topic: test     Partition: 1    Leader: 4       Replicas: 2,4,3 Isr: 4,2,3      Offline: 
        Topic: test     Partition: 2    Leader: 3       Replicas: 3,2,1 Isr: 3,2,1      Offline: 
```

Alter `test` configuration topic using a constraint placement

``` bash
$ kafka-configs --bootstrap-server broker-avivm281:19091 --topic test --replica-placement /etc/kafka/demo/replica-placement.json --alter

>> Completed updating config for topic test.

$ kafka-topics --bootstrap-server broker-avivm281:19091 --topic test --describe

>> Topic: test     TopicId: JPBEenqYSpeC2jizo1Xacg PartitionCount: 3       ReplicationFactor: 3    Configs: min.insync.replicas=2,confluent.placement.constraints={"version":2,"replicas":[{"count":2,"constraints":{"rack":"mel"}},{"count":1,"constraints":{"rack":"euhq"}}],"observers":[{"count":1,"constraints":{"rack":"mel-aop"}},{"count":1,"constraints":{"rack":"euhq-aop"}}]}
        Topic: test     Partition: 0    Leader: 4       Replicas: 4,1,2 Isr: 4,1,2      Offline: 
        Topic: test     Partition: 1    Leader: 4       Replicas: 2,4,3 Isr: 4,2,3      Offline: 
        Topic: test     Partition: 2    Leader: 3       Replicas: 3,2,1 Isr: 3,2,1      Offline: 
```

**No works, new constraint placement is not applied**

Stop your cluster

``` bash
docker-compose -f docker-compose-not-placement.yml down -v
rm -rf kafka_data/
```

## Create a topic with replica assignment and manually alter it after with kafka-configs (not works)

Start the kafka stack

``` bash
docker-compose -f docker-compose-manually-placement.yml up -d
```

Connect to broker-avivm281 

``` bash
docker exec -it broker-avivm281 bash
```

Create topic with a specified replica assigment list (just broker 1, 2, 3. Btw brokers 5,6 still preserved for the observers later).

``` bash
$ kafka-topics --bootstrap-server broker-avivm281:19091 --create --topic test --replica-assignment 1:2:3,2:1:3,3:1:2

>> Created topic test.

$ kafka-topics --bootstrap-server broker-avivm281:19091 --topic test --describe

>> Topic: test     TopicId: veccY9n5R3KTdXRUfKfVFg PartitionCount: 6       ReplicationFactor: 3    Configs: min.insync.replicas=2,confluent.placement.constraints={"version":2,"replicas":[{"count":2,"constraints":{"rack":"mel"}},{"count":1,"constraints":{"rack":"euhq"}}],"observers":[{"count":1,"constraints":{"rack":"mel-aop"}},{"count":1,"constraints":{"rack":"euhq-aop"}}]}
        Topic: test     Partition: 0    Leader: 1       Replicas: 1,2,3 Isr: 1,2,3      Offline: 
        Topic: test     Partition: 1    Leader: 2       Replicas: 2,1,3 Isr: 2,1,3      Offline: 
        Topic: test     Partition: 2    Leader: 3       Replicas: 3,1,2 Isr: 3,1,2      Offline: 
```

Update the new replica placement for the `test` topic

``` bash
$ kafka-configs --bootstrap-server broker-avivm281:19091 --topic test --replica-placement /etc/kafka/demo/replica-placement.json --alter

>> Completed updating config for topic test.

$ kafka-topics --bootstrap-server broker-avivm281:19091 --topic test --describe

>> Topic: test     TopicId: veccY9n5R3KTdXRUfKfVFg PartitionCount: 6       ReplicationFactor: 3    Configs: min.insync.replicas=2,confluent.placement.constraints={"version":2,"replicas":[{"count":2,"constraints":{"rack":"mel"}},{"count":1,"constraints":{"rack":"euhq"}}],"observers":[{"count":1,"constraints":{"rack":"mel-aop"}},{"count":1,"constraints":{"rack":"euhq-aop"}}]}
        Topic: test     Partition: 0    Leader: 1       Replicas: 1,2,3 Isr: 1,2,3      Offline: 
        Topic: test     Partition: 1    Leader: 2       Replicas: 2,1,3 Isr: 2,1,3      Offline: 
        Topic: test     Partition: 2    Leader: 3       Replicas: 3,1,2 Isr: 3,1,2      Offline: 
```

**No works, new constraint placement is not applied**

By the way, if you alter the number of partition of the topic `test`. New partitions added use the new placement constraint. But not on the old partitions.

``` bash
$ kafka-topics --bootstrap-server broker-avivm281:19091 --topic test --alter --partitions 6
$ kafka-topics --bootstrap-server broker-avivm281:19091 --topic test --describe

>> Topic: test     TopicId: veccY9n5R3KTdXRUfKfVFg PartitionCount: 6       ReplicationFactor: 3    Configs: min.insync.replicas=2,confluent.placement.constraints={"version":2,"replicas":[{"count":2,"constraints":{"rack":"mel"}},{"count":1,"constraints":{"rack":"euhq"}}],"observers":[{"count":1,"constraints":{"rack":"mel-aop"}},{"count":1,"constraints":{"rack":"euhq-aop"}}]}
        Topic: test     Partition: 0    Leader: 1       Replicas: 1,2,3 Isr: 1,2,3      Offline: 
        Topic: test     Partition: 1    Leader: 2       Replicas: 2,1,3 Isr: 2,1,3      Offline: 
        Topic: test     Partition: 2    Leader: 3       Replicas: 3,1,2 Isr: 3,1,2      Offline: 
        Topic: test     Partition: 3    Leader: 2       Replicas: 2,1,4,5,6     Isr: 2,1,4,5,6  Offline:        Observers: 5,6
        Topic: test     Partition: 4    Leader: 1       Replicas: 1,2,3,5,6     Isr: 1,2,3,5,6  Offline:        Observers: 5,6
        Topic: test     Partition: 5    Leader: 2       Replicas: 2,1,4,5,6     Isr: 2,1,4,5,6  Offline:        Observers: 5,6
```

Stop your cluster

``` bash
docker-compose -f docker-compose-manually-placement.yml down -v
rm -rf kafka_data/
```

## Manually reassign partitions (works)

Start the kafka stack

``` bash
docker-compose -f docker-compose-manually-placement.yml up -d
```

Connect to broker-avivm281 

``` bash
docker exec -it broker-avivm281 bash
```

Create topic with a specified replica assigment list

``` bash
$ kafka-topics --bootstrap-server broker-avivm281:19091 --create --topic test --replica-assignment 1:2:3,2:1:3,3:1:2

>> Created topic test.
```

Use `reassign-partitions` to reassign partitions (replicas & observers) explicitly and it works
``` bash
$ kafka-reassign-partitions --bootstrap-server broker-avivm281:19091 --execute --reassignment-json-file /etc/kafka/demo/reassign-partition.json

>> Current partition replica assignment

{"version":1,"partitions":[{"topic":"test","partition":0,"replicas":[3,2,1],"log_dirs":["any","any","any"]},{"topic":"test","partition":1,"replicas":[1,3,4],"log_dirs":["any","any","any"]},{"topic":"test","partition":2,"replicas":[4,1,2],"log_dirs":["any","any","any"]}]}

Save this to use as the --reassignment-json-file option during rollback
Successfully started partition reassignments for test-0,test-1,test-2

$ kafka-topics --bootstrap-server broker-avivm281:19091 --topic test --describe

>> Topic: test     TopicId: EfxnG4EGSZyPi2t7iIBa_w PartitionCount: 3       ReplicationFactor: 5    Configs: min.insync.replicas=2
        Topic: test     Partition: 0    Leader: 3       Replicas: 1,2,3,5,6     Isr: 3,2,1      Offline:        Observers: 5,6
        Topic: test     Partition: 1    Leader: 1       Replicas: 2,1,3,5,6     Isr: 1,3,2      Offline:        Observers: 5,6
        Topic: test     Partition: 2    Leader: 3       Replicas: 3,2,1,5,6     Isr: 2,1,3      Offline:        Observers: 5,6
```

🚀 It works !!!

Stop your cluster

``` bash
docker-compose -f docker-compose-manually-placement.yml down -v
rm -rf kafka_data/
```