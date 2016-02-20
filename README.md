# Twitter Stream Sentiment Analysis and Query on a Fast Data Stack
A Reactive Application that ingests Twitter streams, performs Sentiment Analysis, provides for queries and visualizes the results

* Streams tweets using the Twitter stream API to [Akka](http://http://akka.io/)
* Captures the tweets and places them in [Kafka](http://kafka.apache.org)
* Buffers them until they are picked up by [Spark](http://spark.apache.org)
* Performs Sentiment Analysis and stores tweets and opinions in [Cassandra](http://cassandra.apache.org)
* Makes the data queryable via SQL using [Ignite](https://ignite.apache.org)
* Visualizes the query results using [Zeppelin](https://zeppelin.incubator.apache.org)


ScaledAction pipeline
![ScaledAction pipeline](https://github.com/scaledaction/sentiment-analysis/blob/images/images/Pipeline1b.png)


# Deployment via DTK
Instructions for deployment . . .

# Execute SQL queries with Ignite and Zeppelin Notebook
![Zeppelin example](https://raw.githubusercontent.com/abajwa-hw/zeppelin-stack/master/screenshots/4.png)

# Local execution for development

#### Compile application into assembly jars and Docker images
Execute the following in the project base directory:
```
./buildProject.sh 
```

### Start Kafka in a Docker container

Linux:
```
docker run --detach --name kafka1 -p 2181:2181 -p 9092:9092 --env ADVERTISED_HOST=kafka1 --env ADVERTISED_PORT=9092 spotify/kafka
```

Mac:
```
docker run --detach --name kafka1 -p 2181:2181 -p 9092:9092 --env ADVERTISED_HOST=`docker-machine ip \`docker-machine active\`` --env ADVERTISED_PORT=9092 spotify/kafka
```

### Configure Kafka
```
# Connect to the container
docker exec -i -t kafka1 bash

# cd to Kafka installation directory
cd /opt/kafka_2.11-0.8.2.1

# Create a topic
bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic tweets

# Exit container
exit
```

### Start Cassandra in a Docker container

```
docker run --detach --name cassandra1 -p 9042:9042 poklet/cassandra
```

### Configure Cassandra
```
# Connect to Cassandra using cqlsh
docker run -it --rm --net container:cassandra1 poklet/cassandra cqlsh

# Paste the following into your cqlsh prompt to create a twitter keyspace, and a tweets table:

CREATE KEYSPACE IF NOT EXISTS twitter WITH REPLICATION = 
{'class': 'SimpleStrategy', 'replication_factor': 1};

USE twitter;

CREATE TABLE IF NOT EXISTS tweets (
  tweet text, 
  score double, 
  batchTime bigint, 
  tweet_text text, 
  query text, 
  PRIMARY KEY(tweet)
);

# Exit cqlsh and container
exit
```

#### Start the Akka application (frontend) in a Docker container
Note: "TweetSubject" is the subject for selecting tweets from the tweet stream.
```
docker run --detach --name frontend -e TWITTER_CONSUMER_KEY="" -e TWITTER_CONSUMER_SECRET="" -e TWITTER_TOKEN_KEY="" -e TWITTER_TOKEN_SECRET="" -e KAFKA_BROKERS="kafka1:9092" scaledaction/sentiment-analysis-ingest-frontend TweetSubject
```

#### Start the Spark application (backend) in a Docker container
```
docker run --detach --name backend --link kafka1:kafka --link cassandra1:cassandra -e KAFKA_BROKERS="kafka:9092" -e CASSANDRA_SEEDNODES="cassandra" scaledaction/sentiment-analysis-ingest-backend
```

#### To follow the logs of an application container
```
docker logs --follow CONTAINER
```

#### To stop and remove the application containers
```
docker stop frontend
docker stop backend
docker stop kafka1
docker stop cassandra1

docker rm -v frontend
docker rm -v backend
docker rm -v kafka1
docker rm -v cassandra1
```
