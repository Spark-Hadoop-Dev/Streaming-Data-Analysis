# Streaming-Data-Analysis
Department Wise Traffic Analysis On Streaming Data Coming from Web Servers


========================================================================================================
			 LifeCycle of Streaming Analytics
			  ================================
	1. Get Data from source (using Flume and/or Kafka)
	2. Process Data (using Hadoop and Spark)
	3. Store processed data in target (HDFS and/or RDBMS)
	
Project Requirements: 
==========================

• Input data coming from web server logs in standard text format, need to be Ingested in Hadoop's Distributed file system (HDFS) and analyze which department has highest traffic online simultaneously.

• Need to Develop Apache Flume  use case to Ingest streaming data directly to HDFS and to Kafka to get the desired insights with real-time streaming data processing as below mentioned criteria:
	o	Get streaming data directly to HDFS  using Flume to prevent any loss of data and for future prospects
	o	Write streaming data directly to kafka to enable any number of consumers stream these updates off the tail of the 		   log with millisecond latency.
	o	Use Spark to consume streams of data from kafka and and process it to generate desired results and export that 			final result to hdfs and MySQL database  as well.
	
Getting Started with the Slution:

****************************Flume and Kafka Integration*******************

#a2.kafka.conf : A single-node Flume configuration to read data from #webserver logs and publish to Kafka Topic as well as to HDFS Sink to simulteniously to Prevent any data loss.
=================================
// Flume configuration
=================================
vi FlumeToKafkaSinkAndHdfs.conf
**********************************
#defining source, sink and channel
a2.sources = wl
a2.sinks = kafka, k1 
a2.channels = mem, hd

#Describing the source to read data from.
a2.sources.wl.type = exec
a2.sources.wl.command = tail -F /opt/gen_logs/logs/access.log

a2.sinks.k1.type = hdfs
a2.sinks.k1.hdfs.path = /user/cloudera/problem7/step2/
a2.sinks.k1.hdfs.fileSuffix = .txt
a2.sinks.k1.hdfs.fileType = DataStream
a2.sinks.k1.hdfs.writeFormat = Text

#Describing Kafka sink 
a2.sinks.kafka.type = org.apache.flume.sink.kafka.KafkaSink
a2.sinks.kafka.brokerList = localhost:9999
a2.sinks.kafka.topic = testTopic

#Defining a channel which buffers events in memory for HDFS
a2.channels.hd.type = memory
a2.channels.hd.capacity = 500
a2.channels.hd.transactionCapacity = 100

#Defining a channel which buffers events in memory for Kafka
a2.channels.mem.type = memory
a2.channels.mem.capacity = 500
a2.channels.mem.transactionCapacity = 100

#Binding the source and the sinks to the channel
a2.sources.wl.channels = mem,hd
a2.sinks.kafka.channel = mem
a2.sinks.k1.channel = hd

===============================================================
//Developing Spark Streaming with Kafka Integration
================================================================

import org.apache.spark.SparkConf
import org.apache.spark.streaming.{StreamingContext, Seconds}
import org.apache.spark.streaming.kafka._
import kafka.serializer.StringDecoder

object kafkaStreamingDeptCount
{
        def main(args: Array[String]) {
val conf = new SparkConf().setAppName("Flume+Kafka+Spark Integrated Streaming").setMaster(args(0))
val ssc = new StreamingContext(conf, Seconds(30))

val kafkaParams = Map[String, String]("metadata.broker.list" -> "quickstart.cloudera:9092")
val topicSet = Set("flume2kafka")

val directKafkaStream = KafkaUtils.createDirectStream[String, String, StringDecoder, StringDecoder](ssc, kafkaParams, topicSet)
val messages = directKafkaStream.map(s => s._2)
val filterMessage = messages.filter(rec => {
val endpoint = rec.split(" ")(6)
endpoint.split("/")(2) == "department"
})

val mapMessages = filterMessage.map(msg => {
val msgEndPoint = msg.split(" ")(6)
(msgEndPoint.split("/")(2), 1)
})

val results = mapMessages.reduceByKey(_+_)

results.saveAsTextFiles("/user/hdfs/StreamApp/kfk")

ssc.start()
ssc.awaitTermination()
}
}


//Deploying application on cluster

spark-submit \
--class kafkaStreamingDeptCount \
--master local \
--conf spark.ui.port=12456 \
--jars "/usr/lib/kafka/libs/kafka_2.11-0.10.2-kafka-2.2.0.jar,/usr/lib/kafka/libs/kafka-streams-0.10.2-kafka-2.2.0.jar,/usr/lib/kafka/libs/metrics-core-2.2.0.jar" \
retail_2.10-1.0.jar local localhost 9092
=================================================Ends Here============================

Remarks This Program run and tested on Cloudera QuickStart VM 5.12.0

