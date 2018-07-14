# Digging further into KafkaProducer - Tuning batch.size and linger.ms
Hai. About a month ago, I posted a deep dive into the [architecture of Kafka Producer](http://blog.vigneshwaran.in/post/174149490531/a-spiritual-journey-of-digging-into-kafka-producer). 

This is a follow-up to that focusing on how the Producer accumulates data in memory and send it to the network. I hope this knowledge can give clarity in tuning producer configurations for better scale.

---

### Kafka is not scaling for us! How can we squeeze more out of it?

![image](https://raw.githubusercontent.com/vigneshwaranr/blog_posts/master/screenshots/A_spiritual_journey_into_kafka_producer/Level3.png)

For any distributed system, it helps to do the following:

* Understand the overview of the system architecture
* Have a look at [every single configuration](https://kafka.apache.org/documentation/#producerconfigs) and spot what you can tune with safe assumptions at first and then running experiments with different values.
* If still not helping, then it's time to do such deep diving to find where is the bottleneck.

For Kafka, at a minimum it helps to tweak the following configurations:

* batch.size
  * Kafka can batch multiple records *for a topic partition* in batches and send batches together instead of sending individual records.
* linger.ms
  * This setting will keep the batch around for at least *linger.ms* time since the batch was created.
  * This means we introduce a latency of that much time at a minimum.
  * This setting can help if you want to pile up and send bigger batches on a high latency network.
* max.request.size
  * This specifies the maximum size of a single request to a broker node.
  * Because a broker node can be leader for multiple topic-partitions, the producer will group the batches for those partitions together and send them in a single request. 
  * This setting acts as an upper bound for multiple *batch.size*-s for a broker node


###Quite terse.. Not helping! >,<
![image](https://raw.githubusercontent.com/vigneshwaranr/blog_posts/master/screenshots/A_spiritual_journey_into_kafka_producer/Level4.png)


