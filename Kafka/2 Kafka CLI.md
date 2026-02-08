# Kafka CLI

## Kafka Topic Management

- Base command - 
```
kafka-topics.sh --bootstrap-server <hostname:port>
```

- Optionally, use `--command-config <config_file>` to include configs such as credentials.

- __Create Topics__ - `--create --topic <topic_name>`
  - Number of partitions - `--partitions 5`
  - Replication factor - `--replication-factor 3`
  - Set default number of partitions - `num.partitions` in `config/server.properties`.

- __List Topics__ - `--list`

- __Describe Topics__ - `--topic <topic_name> --describe`
  - Lists partitions, their leaders, replicas and ISRs.

- __Delete a Topic__ - `--topic <topic_name> --delete`

## Kafka Console Producer

- Starting a producer - 
```
kafka-console-producer.sh --bootstrap-server <hostname:port> --topic <topic_name>
```

- This will open a prompt where you can send messages _without a key_ i.e. the key will be `null`.
- Optionally, use `--producer.config <config_file>` to include configs such as credentials.

- Adding Producer Properties - 
  - `--producer-property acks=all`
  - `--producer-property partitioner.class=org.apache.kafka.clients.producer.RoundRobinPartitioner`
    - This property will send messages to different partition everytime in round-robin fashion.
    - However, by default, Kafka keeps producing to the same partition until we send at least 16 KBs of data to it and then it will switch partition - better in production.

> [!TIP]
> When producing to a non-existing topic, it is dependent on `auto.create.topics.enable` config in `config/server.properties` file.
>
> If `true` - Gives warnings `LEADER_NOT_AVAILABLE` and eventually auto-creates a new topic and finally succeeds.
>
> If `false` - throws an exception - `Topic not present in metadata`.

- Producing with keys -
```
--property parse.key=true --property key.separator=:
```

> [!WARNING]
> When a producer is started with a key, but the seperator is included in the message then it will throw an exception - `No key separator found`.

## Kafka Console Consumer

- Starting a consumer -
```
kafka-console-consumer.sh --bootstrap-server <hostname:port> --topic <topic_name>
```

- Optionally, use `--consumer.config <config_file>` to include configs such as credentials.
- This will start consuming messages _from this instant_, not the existing messages.

> [!NOTE]
> The ordering of consumed messages may be out-of-order because the ordering is only maintained within the partitions, not within the topic.

- Consuming messages from beginning - `--from-beginning`
- Formatting the output - 
```
--formatter kafka.tools.DefaultMessageFormatter
--property print.timestamp=true
--property print.key=true
--property print.value=true
--property print.partition = true
```

- __Specifying consumer group in Kafka consumer__ - `--group <consumer_group_name>`

> [!TIP]
> `--from-beginning` has no effect when consumers are part of a consumer group. They will start reading messages from last committed offset.

## Kafka Consumer Groups CLI

- Base command - 
```
kafka-consumer-groups.sh --bootstrap-server <hostname:port>
```

- Optionally, use `--command-config <config_file>` to include configs such as credentials.
- List consumer groups - `--list`
  - This will also list consumers which are not assigned a consumer group.
  - Kafka internally assigns a temporary consumer group for them.
- Describe consumer groups - `--describe --group <consumer_group_name>`
  - Shows topic name, partitions, current offset, end offset, LAG, consumer ids, host, client id.
- Delete consumer groups - `--delete --group <consumer_group_name>`

- __Reset Offsets__ -
  - Dry run - `--reset-offsets --to-earliest --topic <topic_name> --dry-run`
  - Execute - `--reset-offsets --to-earliest --topic <topic_name> --execute`

> [!NOTE]
> There must be no consumers running when we reset the offsets.

