一、Flink初体验

1、quickstart
https://ci.apache.org/projects/flink/flink-docs-master/quickstart/setup_quickstart.html


2、下载
https://ci.apache.org/projects/flink/flink-docs-master/quickstart/setup_quickstart.html#download

下载2.7Hadoop以及scala 2.11版本的flink，大小84MB


3、解压
tar zxvf flink-1.0.0-bin-hadoop27-scala_2.11.tgz

4、运行本地版本
lemonhall@HalldeMacBook-Pro:~/Downloads/flink-1.0.0$ bin/start-local.sh
Starting jobmanager daemon on host HalldeMacBook-Pro.local.
lemonhall@HalldeMacBook-Pro:~/Downloads/flink-1.0.0$

5、访问console控制台
 http://localhost:8081 


6、nc打开一个输入
First of all, we use netcat to start local server via

$ nc -l 9000

7、启动一个job
bin/flink run examples/streaming/SocketTextStreamWordCount.jar \
  --hostname localhost \
  --port 9000

8、在nc界面输入流


9、看日志输出
tail -f log/flink-*-jobmanager-*.out

=====================================================================================
=====================================================================================
=====================================================================================
=====================================================================================
=====================================================================================
=====================================================================================

二、Flink 从头开始

1、生成一个空工程:
注意，官方教程上写得archetypeVersion=1.0.0，有问题：
去官方的：http://repo1.maven.org/maven2/archetype-catalog.xml
一看，你会发现，只有1.0.0的，没有所谓的-DarchetypeVersion=1.1-SNAPSHOT
估计还得改mvn的setting才能让那个架构的通过，所以暂时还是用稳定版的1.0.0的吧；

mvn archetype:generate                               \
      -DarchetypeGroupId=org.apache.flink              \
      -DarchetypeArtifactId=flink-quickstart-java      \
      -DarchetypeVersion=1.0.0


Define value for property 'groupId': : com.lsl.lemonhall
Define value for property 'artifactId': : flink.demo
Define value for property 'version':  1.0-SNAPSHOT: :
Define value for property 'package':  com.lsl.lemonhall: :
Confirm properties configuration:
groupId: com.lsl.lemonhall
artifactId: flink.demo
version: 1.0-SNAPSHOT
package: com.lsl.lemonhall
 Y: :
[INFO] ----------------------------------------------------------------------------
[INFO] Using following parameters for creating project from Archetype: flink-quickstart-java:1.0.0
[INFO] ----------------------------------------------------------------------------
[INFO] Parameter: groupId, Value: com.lsl.lemonhall
[INFO] Parameter: artifactId, Value: flink.demo
[INFO] Parameter: version, Value: 1.0-SNAPSHOT
[INFO] Parameter: package, Value: com.lsl.lemonhall
[INFO] Parameter: packageInPathFormat, Value: com/lsl/lemonhall
[INFO] Parameter: package, Value: com.lsl.lemonhall
[INFO] Parameter: version, Value: 1.0-SNAPSHOT
[INFO] Parameter: groupId, Value: com.lsl.lemonhall
[INFO] Parameter: artifactId, Value: flink.demo
[WARNING] CP Don't override file /Users/lemonhall/Downloads/flink.demo/src/main/resources
[INFO] project created from Archetype in dir: /Users/lemonhall/Downloads/flink.demo
[INFO] ------------------------------------------------------------------------






2、导入mvn工程到eclipse



3、导入kafka的支持
官方文档：
https://ci.apache.org/projects/flink/flink-docs-master/apis/streaming/connectors/kafka.html
仓库地址：
http://mvnrepository.com/artifact/org.apache.flink/flink-connector-kafka-0.8_2.10
实际地址：
http://mvnrepository.com/artifact/org.apache.flink/flink-connector-kafka-0.8_2.10/1.0.0


加入到pom
<dependency>
	<groupId>org.apache.flink</groupId>
	<artifactId>flink-connector-kafka-0.8_2.10</artifactId>
	<version>1.0.0</version>
</dependency>



4、安装kafak
下载kfaka
https://kafka.apache.org/downloads.html

5、解压kafka
tar zxvf kafka_2.10-0.8.2.1.tgz

6、启动一个本地kafka环境
https://kafka.apache.org/documentation.html#quickstart


6.1 启动一个zk
bin/zookeeper-server-start.sh config/zookeeper.properties

6.2 启动server
bin/kafka-server-start.sh config/server.properties

6.3 创建一个topic
bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic test

6.4 验证这个topic
bin/kafka-topics.sh --list --zookeeper localhost:2181



7、写程序
package com.lsl.lemonhall;

/**
 * Licensed to the Apache Software Foundation (ASF) under one
 * or more contributor license agreements.  See the NOTICE file
 * distributed with this work for additional information
 * regarding copyright ownership.  The ASF licenses this file
 * to you under the Apache License, Version 2.0 (the
 * "License"); you may not use this file except in compliance
 * with the License.  You may obtain a copy of the License at
 *
 *     http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

import org.apache.flink.api.java.DataSet;
import org.apache.flink.api.java.ExecutionEnvironment;

import java.util.Properties;

import org.apache.flink.api.common.functions.FlatMapFunction;
import org.apache.flink.api.java.tuple.Tuple2;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.datastream.DataStreamSink;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.connectors.kafka.FlinkKafkaConsumer08;
import org.apache.flink.streaming.util.serialization.SimpleStringSchema;
import org.apache.flink.util.Collector;

/**
 * Implements the "WordCount" program that computes a simple word occurrence histogram
 * over some sample data
 *
 * <p>
 * This example shows how to:
 * <ul>
 * <li>write a simple Flink program.
 * <li>use Tuple data types.
 * <li>write and use user-defined functions.
 * </ul>
 *
 */
public class KafkaWordCount {

	//
	//	Program
	//

	public static void main(String[] args) throws Exception {

		// set up the execution environment
		final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
		env.enableCheckpointing(5000); // checkpoint every 5000 msecs
		
		Properties properties = new Properties();
		properties.setProperty("bootstrap.servers", "localhost:9092");
		// only required for Kafka 0.8
		properties.setProperty("zookeeper.connect", "localhost:2181");
		properties.setProperty("group.id", "test");
		DataStreamSink<String> dataStream =env.
				addSource(new FlinkKafkaConsumer08<>("test", new SimpleStringSchema(), properties))
				.print();
		
		env.execute("KafkaWordCount");
		
	}

	//
	// 	User Functions
	//

	/**
	 * Implements the string tokenizer that splits sentences into words as a user-defined
	 * FlatMapFunction. The function takes a line (String) and splits it into
	 * multiple pairs in the form of "(word,1)" (Tuple2<String, Integer>).
	 */
	public static final class LineSplitter implements FlatMapFunction<String, Tuple2<String, Integer>> {

		@Override
		public void flatMap(String value, Collector<Tuple2<String, Integer>> out) {
			// normalize and split the line
			String[] tokens = value.toLowerCase().split("\\W+");

			// emit the pairs
			for (String token : tokens) {
				if (token.length() > 0) {
					out.collect(new Tuple2<String, Integer>(token, 1));
				}
			}
		}
	}
}



8、启动程序


9、写数据到topic
bin/kafka-console-producer.sh --broker-list localhost:9092 --topic test

CTRL+C结束


10、Flink的connectors，对应kafka部分的文档细节
https://ci.apache.org/projects/flink/flink-docs-master/apis/streaming/connectors/kafka.html

=====================================================================================
=====================================================================================
=====================================================================================
=====================================================================================
=====================================================================================
=====================================================================================

三、





