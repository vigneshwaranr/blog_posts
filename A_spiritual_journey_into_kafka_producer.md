# A spiritual journey of digging into KafkaProducer

Hi. In this post, I am going to walk you through the Kafka Producer source code and share my knowledge on the Producer Architecture.

I couldn’t find any such article about Kafka Producer Architecture till date and wanted to post one myself for a long time but 
a recent tech discussion about Kafka at my workplace triggered the motivation to write this.

**Note**: I'm going to focus only on internal implementation of Kafka Producer so if you have no idea about what is Kafka at all, 
this article is **not for you**! You might want to start here -> https://kafka.apache.org/intro

This article is for those who are already familiar with Kafka and doesn't mind digging into the details. 

I’ll be referring Kafka Producer v1.1 (only the idempotent producer mode) for this article.


### Alright, lets start our journey on Kafka Producer!

![image](https://raw.githubusercontent.com/vigneshwaranr/blog_posts/master/screenshots/A_spiritual_journey_into_kafka_producer/Level1.png)

It’d help if you could check out&nbsp;[https://github.com/apache/kafka/tree/1.1](https://github.com/apache/kafka/tree/1.1)&nbsp;and use your IDE’s _⌘+Click_ or _Ctrl+Click_ to follow me along. Or use the links to see the code directly in github.

A brief overview about the essential components:

**Section A**

* *[KafkaProducer](https://github.com/apache/kafka/blob/1.1/clients/src/main/java/org/apache/kafka/clients/producer/KafkaProducer.java)* - The thread-safe boss class that loads up your configs and instantiates all other components. You will call its *send* method to send the *ProducerRecord* over to the Kafka servers.

* *[ProducerRecord](https://github.com/apache/kafka/blob/1.1/clients/src/main/java/org/apache/kafka/clients/producer/ProducerRecord.java)* - A POJO that specifies the data to be sent (key, value, timestamp and headers), name of the topic to be sent to and optionally the partition id (if you need to specify which partition it should go to)

* *[Partitioner](https://github.com/apache/kafka/blob/1.1/clients/src/main/java/org/apache/kafka/clients/producer/Partitioner.java)* - If you don't explicitly specify the partition id in the *ProducerRecord*, then this class determines which record can go to which partition. If you haven't configured your own *Partitioner* using *partitioner.class*, then the [default implementation](https://github.com/apache/kafka/blob/1.1/clients/src/main/java/org/apache/kafka/clients/producer/internals/DefaultPartitioner.java) will be used. More on this later.

* *[Metadata](https://github.com/apache/kafka/blob/1.1/clients/src/main/java/org/apache/kafka/clients/Metadata.java)* / *[Cluster](https://github.com/apache/kafka/blob/1.1/clients/src/main/java/org/apache/kafka/common/Cluster.java)* - These guys store the metadata info about the available topics and their partition counts, number of Kafka Broker nodes and which node is leader for which *[TopicPartition](https://github.com/apache/kafka/blob/1.1/clients/src/main/java/org/apache/kafka/common/TopicPartition.java)* (a POJO that uniquely identifies a partition of a topic). They have the methods to fetch these info from the brokers as well.

**Section B**

* *[RecordAccumulator](https://github.com/apache/kafka/blob/1.1/clients/src/main/java/org/apache/kafka/clients/producer/internals/RecordAccumulator.java#L81)* - This guy holds a *Queue* of *ProducerBatch*-es - one for each *TopicPartition* and also a *ByteBuffer* Pool. Whenever you call *KafkaProducer#send(ProducerRecord)*, they get accumulated here until they are sent over to the brokers when either
  * batching time specified by *linger.ms* is elapsed, or
  * configured *batch.size* is reached even before *linger.ms* is elapsed.

  
* *[ProducerBatch](https://github.com/apache/kafka/blob/1.1/clients/src/main/java/org/apache/kafka/clients/producer/internals/ProducerBatch.java)* - Uses a *[MemoryRecordsBuilder](https://github.com/apache/kafka/blob/1.1/clients/src/main/java/org/apache/kafka/common/record/MemoryRecordsBuilder.java)* to handle compression and write the records into a *ByteBuffer* borrowed from the *BufferPool* in *RecordAccumulator*

**Section C**


* *[NetworkClient](https://github.com/apache/kafka/blob/1.1/clients/src/main/java/org/apache/kafka/clients/NetworkClient.java)* - Uses Java NIO to send *ByteBuffer*-s of a particular *TopicPartition* to the right broker which is the leader of this *TopicPartition*

* *[Sender](https://github.com/apache/kafka/blob/1.1/clients/src/main/java/org/apache/kafka/clients/producer/internals/Sender.java)* / *[KafkaThread](https://github.com/apache/kafka/blob/1.1/clients/src/main/java/org/apache/kafka/clients/producer/KafkaProducer.java#L446)* - Runnable thread that runs forever and polls the data buffers from *RecordAccumulator* and uses *NetworkClient* to send it over to the brokers


### Ermm.. .. Okay..
![image](https://raw.githubusercontent.com/vigneshwaranr/blog_posts/master/screenshots/A_spiritual_journey_into_kafka_producer/Level2.png)

What I listed above covers almost everything. But if you like more details, follow me while I walk you through the code starting from _[KafkaProducer#send()](https://github.com/apache/kafka/blob/1.1/clients/src/main/java/org/apache/kafka/clients/producer/KafkaProducer.java#L790)_ method which is like the main() method of Kafka Producer. Let me screenshot some code so you don't have to jump between tabs.

![image](https://raw.githubusercontent.com/vigneshwaranr/blog_posts/master/screenshots/A_spiritual_journey_into_kafka_producer/doSend1.png)

* Whenever you call send, the first thing it does is making sure the metadata for that topic is available.


* If the metadata for the topic already exists, the&nbsp;_[waitOnMetadata](https://github.com/apache/kafka/blob/1.1/clients/src/main/java/org/apache/kafka/clients/producer/KafkaProducer.java#L888)_ returns immediately

* Or if it is the very first send and it doesn't have metadata info yet, then it issues a blocking request to the Kafka Brokers to fetch and update the *Metadata* and *Cluster* instances. The timeout for this blocking call is specified by&nbsp;_max.block.ms_

* Then it serializes the key and value using the provided serializers or the default ones.


![image](https://raw.githubusercontent.com/vigneshwaranr/blog_posts/master/screenshots/A_spiritual_journey_into_kafka_producer/doSend2.png)

* Then it finds out which partition of the topic this data should go to.
  * If the partition id is explicitly specified, it is used.
  * Otherwise if a custom partitioner is used, that is used to find the partition id.
  * If one is not configured, the *[default partitioner](https://github.com/apache/kafka/blob/1.1/clients/src/main/java/org/apache/kafka/clients/producer/internals/DefaultPartitioner.java#L54)* is used. If a key is provided, the partition will be chosen based on hashing of the key. (Skewing can happen if the keys are not evenly distributed). Otherwise it does round robin.

* Now that it knows which partition to send the data to, it creates the *TopicPartition* POJO.


![image](https://raw.githubusercontent.com/vigneshwaranr/blog_posts/master/screenshots/A_spiritual_journey_into_kafka_producer/doSend3.png)

* Passes that and the serialized data to the *RecordAccumulator* and returns a *Future* that will by completed by the *Sender* when it sends over the *ProducerBatch* containing this *ProducerRecord* (and many other records) to the brokers and receives back the response.


### That's Awesome! Can you explain more on how the accumulator thing works?

![image](https://raw.githubusercontent.com/vigneshwaranr/blog_posts/master/screenshots/A_spiritual_journey_into_kafka_producer/Level3.png)

We shall discuss about that and how to configure *batch.size* and *linger.ms* on high scale or high latency. Stay tuned!
