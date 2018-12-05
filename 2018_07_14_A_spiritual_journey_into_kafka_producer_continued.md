# Digging further into KafkaProducer - Tuning batch.size and linger.ms
Hai. Several months ago, I posted a deep dive into the [architecture of Kafka Producer](http://blog.vigneshwaran.in/post/174149490531/a-spiritual-journey-of-digging-into-kafka-producer). 

This is a follow-up to that focusing only on how the Producer accumulates data in memory and send it to the network. I hope this knowledge can give clarity in tuning producer configurations for better scaling.

---

### Kafka is not scaling for us! How can we squeeze more out of it?

![image](https://raw.githubusercontent.com/vigneshwaranr/blog_posts/master/screenshots/A_spiritual_journey_into_kafka_producer/Level3.png)

For any distributed system, it helps to do the following:

* Understand the overview of its system architecture
* Have a look at [every single configuration](https://kafka.apache.org/documentation/#producerconfigs) it has and spot what you can tune with safe assumptions at first and then keep tweating by experimenting with different values.
* If still not helping, then it's time to do such deep diving to find where is the bottleneck.

For Kafka, I found that these three are the main configurations we can tweak:

* `batch.size`
  * Kafka can accumulate multiple records *of a particular topic-partition* in batches and send multiple batches together instead of sending individual records.
  * `batch.size` sets the maximum size for a single batch.
  * Once this size is reached, the batch will be sent over the network if the [Sender thread](https://github.com/apache/kafka/blob/1.1/clients/src/main/java/org/apache/kafka/clients/producer/internals/Sender.java) is free.
* `linger.ms`
  * `linger.ms` determines how long [since a batch was created](https://github.com/apache/kafka/blob/1.1/clients/src/main/java/org/apache/kafka/clients/producer/internals/ProducerBatch.java#L61) must it linger in memory before being sent over the network **IFF** the batch hasn't reached `batch.size` yet.
  * This means we introduce a latency of at least `linger.ms` if the rate of producing is too slow for batches to reach `batch.size` within this time.
  * IF you have a high latency between your producer and broker nodes, you can increase `linger.ms` and `batch.size` to pile up and send bigger batches for efficient throughput.
* `max.request.size`
  * This specifies the maximum size of a single request to a broker node.
  * Because a broker node can be leader for multiple topic-partitions, the producer will group the batches for those partitions together and send them in a single request. 
  * This setting acts as an upper bound for a request containing multiple batches for a broker node.


### That's informative.. But difficult to visualize though..

![image](https://raw.githubusercontent.com/vigneshwaranr/blog_posts/master/screenshots/A_spiritual_journey_into_kafka_producer/Level4.png)

Ok. I tried my best to lay it out in a digram. This is essentially the 

![image](https://raw.githubusercontent.com/vigneshwaranr/blog_posts/master/screenshots/A_spiritual_journey_into_kafka_producer/RecordAccumulator.png)