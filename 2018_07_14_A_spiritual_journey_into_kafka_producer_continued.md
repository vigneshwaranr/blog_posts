# Digging further into KafkaProducer - Tuning batch.size and linger.ms
Hi. Several months ago, I posted a deep dive into the [anatomy of Kafka Producer](http://blog.vigneshwaran.in/post/174149490531/a-spiritual-journey-of-digging-into-kafka-producer). (Much like what [Christopher-Stoll had done to Pokemons](https://www.deviantart.com/christopher-stoll/art/Charmander-Anatomy-Pokedex-Entry-626850199).)

This is a follow-up to that post but focusing only on how the Producer accumulates data in memory and send it to the network. This is so that it can give you better clarity in tuning producer configurations for better scaling.

Let's get back to our curious monkey and elephant on their journey.


---


### Master! We need more throughput from Kafka. How can we squeeze more out of it?

![image](https://raw.githubusercontent.com/vigneshwaranr/blog_posts/master/screenshots/A_spiritual_journey_into_kafka_producer/Level3.png)

Hmm.. For any distributed system, it always helps to do the following:

* Understand the overview of its system architecture
* Have a look at [every single configuration](https://kafka.apache.org/documentation/#producerconfigs) it has and spot what you can tune with safe assumptions at first and then keep tweaking by experimenting with different values.
* If still not helping, then it's time to do such deep diving to find where is the bottleneck.

For Kafka, I found it helpful to tweak these three configurations:

* `batch.size`
  * Kafka can accumulate multiple records *of a particular topic-partition* in batches and send multiple batches of records together instead of sending individual records.
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


### That's was very informative.. .. but quite terse like all your previous explanations 😒..

![image](https://raw.githubusercontent.com/vigneshwaranr/blog_posts/master/screenshots/A_spiritual_journey_into_kafka_producer/Level4.png)

Ok. I hear this complaint a lot. This time I will explain with diagrams. I tried my best to lay out the RecordAccumulator architecture.

#### A quick overview first

![image](https://raw.githubusercontent.com/vigneshwaranr/blog_posts/master/screenshots/A_spiritual_journey_into_kafka_producer/ProducerOverview.png)

1. When you send a message using [KafkaProducer](https://github.com/apache/kafka/blob/1.1/clients/src/main/java/org/apache/kafka/clients/producer/KafkaProducer.java), it stores them in RecordAccumulator in batches.

2. Whenever a batch reaches its `batch.size`, the RecordAccumulator will notify the Sender thread.

3. If the Sender thread is not busy sending data, it will look for batches that are ready to send (full size batches or smaller batches that have lingered longer than `linger.ms`)

4. The Sender thread will send the batch to appropriate broker node which is leader for that batch.

#### Configurations At Play..


![image](https://raw.githubusercontent.com/vigneshwaranr/blog_posts/master/screenshots/A_spiritual_journey_into_kafka_producer/RecordAccumulator.png)

In this diagram, I demonstrate a fictional situation over 60ms period in KafkaProducer for explaining purpose.

**Kafka setup:**

* 2 Topics
* Topic1 has 2 partitions
* Topic2 has 1 partition
* 2 Brokers
  * Broker1 is leader for Topic1-Partition1
  * Broker2 is leader for Topic1-Partition2 and Topic2-Partition1

**Configurations:**

* batch.size = 15kB
* linger.ms = 20ms

The RecordAccumulator holds a [concurrent map](https://github.com/apache/kafka/blob/1.1/clients/src/main/java/org/apache/kafka/clients/producer/internals/RecordAccumulator.java#L81) where keys are TopicPartitions and values are Queues of associated batches.

**Topic1-Partition1 between 0 and 30ms timelines:**

* *Topic1-Partition1* continuously receives lot of messages (about 24kB) in this period.

* Because *batch.size* is 15kB, it is accumulated as two batches in the queue for *Topic1-Partition1*

* As soon as *Batch1* is filled, RecordAccumulator notifies Sender thread just in case it is idle.

* At 20ms timeline, the Sender is indeed idle, so it will start sending this batch over the network to its leader node *Broker1*.

**Topic1-Partition2 between 0 and 30ms timelines:**

* *Topic1-Partition2* receives a full batch of messages between 10 and 30ms timeline and RecordAccumulator notifies Sender that batch is full.

* If the Sender had already finished sending *Batch1*, it'll pick up and send *Batch4* to its leader *Broker2*

* *Topic2-Partition1* received only 9kB of messages so the *Batch5* just lingers in memory similar to *Batch2*

**Between 30 and 60ms timelines:**

* In this example, the Sender got free of things to do at 40ms timeline. So it looks around for any batches to send.

* It notices that the *Batch2* and *Batch5* were lingering in memory longer than the configured limit of 20ms or more so it picks them up and sends them to their appropriate leaders.

* I leave the possibilities of *Batch3*'s fate to your imagination.

### Wow! Now we can tweak these configurations with this understanding! Arigatou Sensei!

![image](https://raw.githubusercontent.com/vigneshwaranr/blog_posts/master/screenshots/A_spiritual_journey_into_kafka_producer/Level5.png)

Great! It would also help to note the following learnings.

* Measure how long it takes for your messages to be delivered between producer and consumer. If latency is negligible, you don't need to worry much.

* I faced where there was high latency between Producer and Broker nodes because they are in separate faraway physical locations. The Producer frequently sent data to Broker and it hurt throughput.

* To solve that, I set a large enough value for `batch.size`, `linger.ms` and `max.request.size`. This will make Producer wait, accumulate and send bigger bulk of data instead of being chatty. But not too large enough that it slows down everything.

* Make sure the bandwith of your network is big enough to accommodate such large payload. And check your broker configurations too whether they can receive your payload and tweak as per your needs.

---

P.S. Influences for the extravagant style of this article come from the following :)

* [Head First Design Patterns](https://amzn.to/2EXaJ8N)
* [Why's (poignant) guide to ruby](https://poignant.guide/)
* [Learn You a Haskell for Great Good!](http://learnyouahaskell.com/)
* [Ruby Koans](http://rubykoans.com/)
