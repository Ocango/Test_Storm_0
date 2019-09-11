# 开发环境准备
1.  Java 1.8.0
2.  Maven 3.6.0
3.  VS code 1.33.1
开发IDE其实不限，鱼只是习惯性用VS code
Maven的基础见识详见[看懂MAVEN的结构目录](https://www.jianshu.com/p/967ac0eb8783)

# 第一件事，开发依赖包变更
官网描述如下：
> In the latest version, the class packages have been changed from "backtype.storm" to "org.apache.storm" so the topology code compiled with older version won't run on the Storm 1.0.0 just like that. Backward compatibility is available through following configuration
> 
> client.jartransformer.class: "org.apache.storm.hack.StormShadeTransformer"

所以一定要注意在网上找资料和学习的时候，区分不同版本。鱼因为这个把server端和local端JDK全重装了，最后才发现问题。
# Storm开发本地调试
作为初学者，首先要学会用，所以简单直接一点，我们去maven库里面看看有没有可以学习的内容。
![搜索MAVEN库](https://upload-images.jianshu.io/upload_images/15615374-0121d5998eb74b5a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
果然，看介绍是一个git上的单词计数器，好的来试试效果。

![Word Count program目录架构](https://upload-images.jianshu.io/upload_images/15615374-5b462153fdf3d608.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
如果找不到此storm示例项目，可以直接按以下的原文件进行配置。
# POM.XML配置
```
<?xml version="1.0" encoding="UTF-8"?>
<!-- xml声明，包括版本和编码方式 -->
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
<!-- xmlns，xml的默认命名空间；xmlns:xsi，自定义前缀的命名空间；xsi:schemaLocation，由一个URI引用对组成，两个URI之间以空白符分隔。第一个URI是名称空间的名字，第二个URI给出模式文档的位置，模式处理器将从这个位置读取模式文档，该模式文档的目标名称空间必须与第一个URI相匹配。 -->
  <modelVersion>4.0.0</modelVersion>
  <!-- 指定了当前pom.xml版本 -->
  <groupId>Ocango</groupId>
  <!-- <groupId>反写公司网址+项目名</groupId>：主项目标识，用于显示此项目属于哪个主项目下。 -->
  <artifactId>storm_test</artifactId>
  <!-- <artifactId>项目名+模块名</artifactId>：此项目属于主项目中的某个模块 -->
  <version>1.0-SNAPSHOT</version>
  <!-- 例如： 
      当前项目的版本号0.0.1： 。
      第一个0标识大版本号；  
      第二个0表示分支版本号；  
      第三个0表示小版本号 。
  版本类型：  
      snapshot：快照    
      alpha：内部测试        
      beta：公测         
      release：稳定          
      GA：正式发布 -->
  <!-- <packaging></packaging>：Maven项目打包的方式。默认是jar，还可以打包为war、pom、zip。-->
  <!-- 依赖列表： 参考 https://blog.csdn.net/codejas/article/details/79490030 -->
  <dependencies>
   	<dependency>
  		<groupId>org.apache.storm</groupId>
  		<artifactId>storm-core</artifactId>
  		<version>1.2.2</version>
  		<scope>provided</scope><!-- ：依赖的范围，provided不参与打包 -->
      <!-- <optional></optional>：有true、false两个值，默认是false；意思为：设置依赖是否可选。如果为true，则子项目必须显式引入此依赖。
      <exclusions>：排除传递依赖的列表
        <exclusion></exclusion>：
      </exclusions> -->
  	</dependency>
  </dependencies>
  <!-- 为构建行为提供支持： -->
  <properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
  </properties>
  <build>
    <plugins> <!-- ：插件列表 https://www.cnblogs.com/zhangxh20/p/6298062.html -->
      <plugin>
      <!-- 使用编译器的定义，比如jdk版本等 -->
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-compiler-plugin</artifactId>
        <version>3.6.0</version>
        <configuration>
          <source>1.8</source>
          <target>1.8</target>
        </configuration>
      </plugin>
      <plugin>
      <!-- 用于把多个jar包，打成1个jar包 -->
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-shade-plugin</artifactId>
        <version>3.2.1</version>
        <executions>
          <execution>
            <phase>package</phase>
            <goals>
              <goal>shade</goal>
            </goals>
          </execution>
        </executions>
        <configuration>
          <finalName>uber-storm_test-1.0</finalName>
        </configuration>
      </plugin> 
      
    </plugins>
  </build>
  <!-- 聚合运行多个的maven项目: -->
  <!-- <modules>
    <model></model>
  </modules> -->
</project>
```
*备注说明下关于原文件的修改：*
1.storm-core原先是0.10.0，但是server个人配置的最新的1.2.2，同时这个问题也涉及到对包名的修改。此处见开头。
2.指定编译使用UTF-8。
3.原先maven-shade-plugin未指定版本，这里增加了指定，不然会警报。
#配置SPOUT
```
package count_word;
import org.apache.storm.spout.SpoutOutputCollector;
import org.apache.storm.task.TopologyContext;
import org.apache.storm.topology.OutputFieldsDeclarer;
import org.apache.storm.topology.base.BaseRichSpout;
import org.apache.storm.tuple.Fields;
import org.apache.storm.tuple.Values;
import org.apache.storm.utils.Utils;

import java.util.Map;
import java.util.Random;
//定义数据源
public class RandomSentenceSpout extends BaseRichSpout {
  SpoutOutputCollector _collector;
  //定义一个用来管控tuples的output collector，其能确保每个tuple至少被处理一次
  Random _rand;


  @Override
  //初始化spout
  public void open(Map conf, TopologyContext context, SpoutOutputCollector collector) {
    _collector = collector;
    _rand = new Random();
  }
/**
 * Storm会在同一个线程中执行所有的ack、fail、nextTuple方法，因此实现者无需担心Spout
 * 的这些方法间的并发问题，因此实现者应该保证nextTuple方法是非阻塞的，否则会阻塞
 * Storm处理ack和fail方法。
 */
/**
 * 该方法会不断的调用，从而不断的向外发射Tuple.
 * 该方法应该是非阻塞的，如果这个Spout没有Tuple要发射，那么这个方法应该立即返回
 * 同时如果没有Tuple要发射，将会Sleep一段时间，防止浪费过多CPU资源
 */
  @Override
  public void nextTuple() {
    Utils.sleep(100);
    String[] sentences = new String[]{ "the cow jumped over the moon", "an apple a day keeps the doctor away",
        "four score and seven years ago", "snow white and the seven dwarfs", "i am at two with nature" };
    String sentence = sentences[_rand.nextInt(sentences.length)];
    _collector.emit(new Values(sentence));
  }
/**
 * Storm已经确定由该Spout发出的具有msgId标识符的Tuple已经被完全处理，该
 * 方法会被调用。。
 * 通常情况下，这种方法的实现会将该消息从队列中取出并阻止其重播。
 */
  @Override
  public void ack(Object id) {
  }
/**
 * 该Spout发出的带有msgId标识符的Tuple未能完全处理，此方法将会被调用。。
 * 通常，此方法的实现将把该消息放回到队列中，以便稍后重播。
 */
  @Override
  public void fail(Object id) {
  }

  @Override
  //声明此topology的所有spout的输出模式
  public void declareOutputFields(OutputFieldsDeclarer declarer) {
    declarer.declare(new Fields("word"));
  }

}
```
大多数函数意义在备注中已写全
#配置BOLT和TOPOLOGY
```
package count_word;

import org.apache.storm.Config;
import org.apache.storm.LocalCluster;
import org.apache.storm.StormSubmitter;
import org.apache.storm.task.ShellBolt;
import org.apache.storm.topology.BasicOutputCollector;
import org.apache.storm.topology.IRichBolt;
import org.apache.storm.topology.OutputFieldsDeclarer;
import org.apache.storm.topology.TopologyBuilder;
import org.apache.storm.topology.base.BaseBasicBolt;
import org.apache.storm.tuple.Fields;
import org.apache.storm.tuple.Tuple;
import org.apache.storm.tuple.Values;

import java.util.HashMap;
import java.util.Map;
import java.util.Set;

/**
 * This topology demonstrates Storm's stream groupings
 * Adapted from https://github.com/nathanmarz/storm-starter under the Apache license
 */
public class WordCountTopology {
  //定义一个切分单词用的Bolt
  public static class SplitSentence extends BaseBasicBolt {

    //声明此Bolt的所有spout的输出模式
    @Override
    public void declareOutputFields(OutputFieldsDeclarer declarer) {
      declarer.declare(new Fields("word"));
      
    }
  /**
   * 处理 tuple并输出新的tuple，处理失败可以抛出FailedException异常
   */
    @Override
    public void execute(Tuple input, BasicOutputCollector collector) {
      String[] words = input.getString(0).split(" ");
      for(String word: words)
        collector.emit(new Values(word));
	}

  }
  //定义一个统计用的Bolt
  public static class WordCount extends BaseBasicBolt {
    Map<String, Integer> counts = new HashMap<String, Integer>();

    @Override
    public void execute(Tuple tuple, BasicOutputCollector collector) {
      String word = tuple.getString(0);
      Integer count = counts.get(word);
      if (count == null)
        count = 0;
      count++;
      counts.put(word, count);
      collector.emit(new Values(word, count));
      printReport();
    }
    //声明此Bolt的所有spout的输出模式
    @Override
    public void declareOutputFields(OutputFieldsDeclarer declarer) {
      declarer.declare(new Fields("word", "count"));
    }
    //输出LOG
    private void printReport(){
      System.out.println("------begin------");
      Set<String> words = counts.keySet();
      for(String word : words){
        System.out.println(word + "\t-->\t" + counts.get(word));
      }
      System.out.println("-------end-------");
    }

  }
//main
  public static void main(String[] args) throws Exception {

    TopologyBuilder builder = new TopologyBuilder();

    builder.setSpout("spout", new RandomSentenceSpout(), 5);

    builder.setBolt("split", new SplitSentence(), 8).shuffleGrouping("spout");
    builder.setBolt("count", new WordCount(), 12).fieldsGrouping("split", new Fields("word"));

    Config conf = new Config();
    conf.setDebug(true);


    if (args != null && args.length > 0) {
      conf.setNumWorkers(3);
      //提交一个storm，以便在集群上运行。storm将永远运行，或者直到显式地终止。
      //拓扑名
      //设定
      //指定执行的Topology
      StormSubmitter.submitTopologyWithProgressBar(args[0], conf, builder.createTopology());
    }
    else {
      // 本地测试
      // 设定线程数上线
      conf.setMaxTaskParallelism(3);

      LocalCluster cluster = new LocalCluster();
      cluster.submitTopology("word-count", conf, builder.createTopology());

      Thread.sleep(20000);

      cluster.shutdown();
    }
  }
}
```
注意，此TOPOLOGY当无参数时为LOCAL测试，此处设置20S后自动KILL。当有参数时，参数为TOPOLOGY的ID，即任务名，即为正式提交TOPOLOGY。

# 点运行！本地调试
![满心期待！](https://upload-images.jianshu.io/upload_images/15615374-d2b6b9525cb0fda4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![部分输出结果](https://upload-images.jianshu.io/upload_images/15615374-a7e6bde3cb0c08ad.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
可以看到本地测试非常成功，但是，本地测试有个鬼用，我们上server！
server上本地测试：
![server本地测试](https://upload-images.jianshu.io/upload_images/15615374-847888d211433650.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

同样没有问题，但是请注意，不要进行集群测试，除非你的storm已经配置完整，不然会难受的~··~
# 集群配置
storm的运行基础zookeeper虚拟环境配置详见[Storm开发——Zookeeper集群设置(集群)](https://www.jianshu.com/p/194cd9c7926f)
以上文为基础
那么，基于zookeeper，storm如何向集群提交TOPOLOGY呢？？？
修改storm配置档（还是那句老话，空格不是装饰品，请注意）
```
#按照目前zookeeper的设定即可
 storm.zookeeper.servers:
  - "xxxx"
  - "yyyy"
  - "zzzz"
#zookeeper的端口
 storm.zookeeper.port: 2181
#设定为集群模式
 storm.cluster.mode: distributed
#MASTER，即nimbus
 nimbus.seeds: ["xxxx"]
 storm.zookeeper.root: /storm
#log和data目录，有时无法执行尝试清空重建
 storm.log.dir: "/usr/local/storm/logs"
 storm.local.dir: "/usr/local/storm/data"
 #nimbus.host: "localhost"
#UI的端口
 ui.port: 9090
#slot端口，请依照机能设置，我这1G渣渣就放四个了
 supervisor.slots.ports:
  - 6700
  - 6701
  - 6702
  - 6703 
```
注意点：
1.此处server都是用的主机名，需要在/etc/hosts维护
2.UI的端口最好要额外设置，因为正常情况下默认端口8080会被其他服务占用（netstat -ntulp）
3.nimbus的地址现在用nimbus.seeds，但是其实也还可以用nimbus.host
4.storm.local.dir，storm.log.dir不存在会自建，但是还是要注意第一执行权限，第二发现无解的错误需要重建下此目录
5.空格，空格。空格，空格
6.运行状态下nimbus主机上再挂一个supervisor，会导致nimbus异常退出，原因暂时未知，可能和配置文件有关，或者就是这么设置的

复制此文件至集群其他server上

#向集群提交TOPOLOGY
打包
```
> mvn install
```
上传

提交
```
./bin/storm jar ~/storm_jar/uber-storm_test-1.0.jar count_word.WordCountTopology count_world
#./bin/storm jar   封装包   main类，  入口   TOPOLOGY名 
```
到UI查看TOPOLOGY状态

当然，作为演示这样是可以的，但是啊，还有三个问题：
1.spout的来源
2.TOPOLOGY的提交方式还停留在0.10.0版本
3.bolt与处理完数据的流出

好的我们慢慢来！

本文文集链接[Storm开发历程](https://www.jianshu.com/nb/32785021)