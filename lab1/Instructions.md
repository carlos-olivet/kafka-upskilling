Lab 1: Making Sense of the Basics


Prerequisites

Download Kafka: Make sure you have a Kafka binary distribution downloaded and unzipped.

# Replace this path with your own Kafka directory

I would suggest adding this to `.zshrc` or similar
```shell
export KAFKA_HOME=~/kafka_2.13-4.1.0
```

# Step 1: Prepare your files and format your storage

## Step 1.1: Prepare your files
Update your properties files to create 1 controller and one broker.

Config Files: You should have `controller1.properties` and `broker1.properties` in your current directory.


## Step 1.2: Generate a unique ID for our cluster
Generate a cluster ID with the following command. Note down the result as well as it can be useful in future, it can also be found inside logs folder in the `meta.properties` filed.

```shell
export CLUSTER_ID=$($KAFKA_HOME/bin/kafka-storage.sh random-uuid)
```

## Step 1.3: Format Storage (One-time setup)

Before starting, Kafka's new KRaft mode needs to format its storage directory. You only do this once using the controller config

```shell
$KAFKA_HOME/bin/kafka-storage.sh format \
  --cluster-id $CLUSTER_ID \
  --config controller1.properties
```

```shell
$KAFKA_HOME/bin/kafka-storage.sh format \
  --cluster-id $CLUSTER_ID \
  --config broker1.properties
```

Why? This initializes the log.dirs you specified in controller.properties and sets up the cluster with its new unique ID.

# Step 2: Start the Cluster

You will need 4 separate terminals for this.

## Terminal 1: Start Controller

This is the "manager" node. It tracks which brokers are alive and manages the cluster.

```shell
$KAFKA_HOME/bin/kafka-server-start.sh controller1.properties
```

What's happening? This process is now the brain of your cluster, listening on port 9093.

## Terminal 2: Start Broker

This is the "worker" node. It stores the data and serves requests.

```shell
$KAFKA_HOME/bin/kafka-server-start.sh broker1.properties
```

What's happening? This broker starts up, connects to the controller (at localhost:9093) to "register" itself, and starts listening on port 9094 for producers and consumers.

# Step 3: Create a Topic (with Partitions!)

Let's create the topic we'll use for our data.

## Terminal 3: Admin Terminal

Open a new terminal.

```shell
$KAFKA_HOME/bin/kafka-topics.sh --create \
  --topic MoviesWatched \
  --bootstrap-server localhost:9094 \
  --partitions 3 \
  --replication-factor 1
```

Let's break down the flags:

`--create --topic MoviesWatched`: The action and the name of our topic.

`--bootstrap-server localhost:9094`: We point it at our broker's public-facing port.

`--partitions 3`: This is key. We're not creating one log; we're creating three parallel logs. This is for scalability.

`--replication-factor 1`: This indicates how many copies of the data we will have. We're using 1 because we only have one broker. In production, this must be 3 or more for fault tolerance.

# Step 4: Produce Messages (with Keys)

Keep Terminal 3 open. This will be our producer.

## Terminal 3
Let's create a producer and send some data

```shell
$KAFKA_HOME/bin/kafka-console-producer.sh \
  --topic MoviesWatched \
  --bootstrap-server localhost:9094
```

Now, type the following messages one by one and press Enter.

```json
{"MovieName": "Halloween", "Genre": "Horror", "UserId":123435}
{"MovieName": "Halloween", "Genre": "Horror", "UserId":567345}
{"MovieName": "Nightmare before Christmas", "Genre": "Horror", "UserId":123435}
{"MovieName": "Muppet's Christmas Carol", "Genre": "Christmas", "UserId":123435}
```

# Step 5: Consume Messages (with a Group)

This is the final piece. Let's read the data.

Terminal 4: Consumer Terminal

Open a new terminal.

## Terminal 4
```shell
$KAFKA_HOME/bin/kafka-console-consumer.sh \
  --topic MoviesWatched \
  --bootstrap-server localhost:9094 \
  --from-beginning \
  --group MyFirstGroup
```

You should see all three messages you sent appear immediately.

Why these flags?

--from-beginning: Start at offset 0 (the start of the log). Without this, it only shows new messages sent after it starts.

--group MyFirstGroup: This is the most important part. We've given our consumer a name. Kafka is now tracking the offset for MyFirstGroup.

# Step 6: Test Offset Tracking

In Terminal 4 (the consumer), press Ctrl+C to stop it.

Now, re-run the exact same command:

## Terminal 4 (run this again)
```shell
$KAFKA_HOME/bin/kafka-console-consumer.sh \
  --topic MoviesWatched \
  --bootstrap-server localhost:9094 \
  --from-beginning \
  --group MyFirstGroup
```

kafka now knows the group already consumed some messages and ignores the --from-beginning.

Go back to Terminal 3 (the producer) and send a new message:

```json
{"MovieName": "Muppet's Christmas Carol", "Genre": "Christmas", "UserId":123435}
```

Look at Terminal 4. The new message appears instantly! Your consumer is just waiting at the end of the log for new data.