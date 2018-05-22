# A spiritual journey of digging down KafkaProducer

Hi. In this post, I am going to walk you through the Kafka Producer source code and share my knowledge on the Producer Architecture.

I couldn’t find any such article about Kafka Producer Architecture till date and wanted to post one myself for a long time but 
a recent tech discussion about Kafka at my workplace triggered the motivation to write this.

**Note**: I'm going to focus only on internal implementation of Kafka Producer so if you have no idea about what is Kafka at all, 
this article is **not for you**! 

This article is for those who are already familiar with Kafka and doesn't mind digging into the details. 

I’ll be referring Kafka Producer v1.1 (only the idempotent producer mode) for this article.

## Alright, lets start our journey on Kafka Producer!
![image](https://raw.githubusercontent.com/vigneshwaranr/blog_posts/master/screenshots/A_spiritual_journey_into_kafka_producer/Level1.png)

It’d help if you check out&nbsp;[https://github.com/apache/kafka/tree/1.1](https://github.com/apache/kafka/tree/1.1)&nbsp;and use your IDE’s _⌘+Click_ or _Ctrl+Click_ to follow along.

A brief overview about the essential components:

* 

*   The _RecordAccumulator_ will notify the _Sender_ to send the data soon (even before _linger.ms_&nbsp;is elapsed)&nbsp;if the configured _batch.size_&nbsp;-&nbsp; the maximum size for a _ByteBuffer_ is reached

## Umm..
![image](https://raw.githubusercontent.com/vigneshwaranr/blog_posts/master/screenshots/A_spiritual_journey_into_kafka_producer/Level2.png)

Let me walk you through the code starting from _[KafkaProducer#send()](https://github.com/apache/kafka/blob/1.1/clients/src/main/java/org/apache/kafka/clients/producer/KafkaProducer.java#L790)_ method which is like the main() method of Kafka Producer.

*   Whenever you call send, the first thing it does is making sure the metadata for that topic is available.> `ClusterAndWaitTime clusterAndWaitTime = waitOnMetadata(record.topic(), record.partition(), maxBlockTimeMs);`

*   If the metadata for the topic already exists, the&nbsp;_[waitOnMetadata](https://github.com/apache/kafka/blob/1.1/clients/src/main/java/org/apache/kafka/clients/producer/KafkaProducer.java#L888)_ returns immediately

*   Otherwise it issues a blocking call to the Kafka Brokers to fetch and update the Metadata and Cluster info. The timeout for this blocking call is specified by&nbsp;_max.block.ms_
