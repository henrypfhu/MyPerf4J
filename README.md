# MyPerf4J
A Extremely Fast Performance Monitoring and Statistics for Java Code. Inspired by [perf4j](https://github.com/perf4j/perf4j) and [TProfiler](https://github.com/alibaba/TProfiler).
Committed to becoming a performance monitoring and statistics tool that can be used for a long time in a production environment!


## Multilingual document
* English 
* 中文 [README.CN.md](https://github.com/ThinkpadNC5/MyPerf4J/blob/develop/README.CN.md)

## Background
* I need a program that can measure the response time of method.
* The existing statistics of [perf4j](https://github.com/perf4j/perf4j) cannot meet my needs.

## Requirements
* Statistics on the performance indicators of the method such as RPS, Avg, Min, Max, StdDev, TP50, TP90, TP95, TP99, TP999 and TP9999, etc.
* It can be configured by properties.
* Does not take up too much memory, does not affect the normal response of the program.
* The processing of performance indicators can be customized, for example: log collection, reporting to log collection services, etc.

## Design
* By recording all response times, all possible analyses can be performed, including RPS, TP99, etc. How can we store these data efficiently?
    - In the form of K-V, K is the response time, and V is the corresponding number of times, so the memory occupancy is only related to the number of different response times, regardless of the total number of requests.
    - If we simply use Map to store data, it will take up a lot of unnecessary memory.
    - According to Pareto principle, most of the of interface response time is in a very small range. This small range is particularly suitable for storage using arrays. The array subscript is the response time, and the corresponding element is the number corresponding to the response time. A small number of interface response time distribution will be relatively large, suitable for storage with Map;
    - In summary, the core data structure is: array & Map, the response time less than a certain threshold is recorded in the array, and the response time greater than or equal to the threshold is recorded in the Map.
* Using AOP for response time acquisition, using [ASM](http://asm.ow2.io/) to implement AOP and improve performance
* Configuration through properties and tuning of memory usage of core data structures through parameter configuration.
* It can collect response time by synchronous way, avoid creating too many Runnable objects, and affect GC of program.
* Acquire the collection result asynchronously to avoid affecting the response time of the interface.
* Expose the interface for processing the collected results to facilitate customized processing.

## Memory
* Prerequisites
    - There are 1024 interfaces on the service that need to be monitored.
    - Most of the response time of each interface is within 300ms, and there are 100 different response times larger than 300ms.
    - Non-core data structures occupy 2MB.
* Rough Mode
    - Only record requests with response time less than 1000ms.
    - 2 * 1024 * (1000 * 4B) + 2MB ≈ 10MB
* Accurate Mode
    - Record all response times.
    - 2 * 1024 * (300 * 4B + 100 * 90B) + 2MB ≈ 22MB 

## Benchmark
* Test Platform
    - OS: macOS High Sierra 10.13.3
    - JDK: 1.8.0_161
    - JVM options: -server -Xmx4G -Xms4G -Xmn2G
    - CPU: Intel(R) Core(TM) i7-7920HQ CPU@3.10GHz

* Test way
    - Run empty methods.
    - In order to avoid the performance degradation caused by the high competition of multiple threads due to the high execution speed of the empty method, eight empty methods are executed by polling, and then the RPS of the eight methods is added to obtain the result.
    - The time slice is 10s, each press pauses for 20s, and executes `System.gc();`  

* MyPerf4J-ASM

| Threads | Number of loops per thread | RPS |
|-------|-----|------|
|1|1000000000|13815816|
|2|1000000000|16199712|
|4|1000000000|33060632|
|8|1000000000|55981416|

* Summary
    - From the benchmark results
        - MyPerf4J-ASM can support 13.81 million method calls per second in a single thread. The average time per method call is 72.3ns, which can meet the requirements of most people, and does not affect the response time of the program itself.
    - Reason for high performance
        - MyPerf4J-ASM modifies the bytecode of the class through the ASM framework, inserting two lines of methods before and after the method, without generating redundant objects, and not triggering any GC in the whole process of the benchmark (except for the `System.gc();` executed in the code).

## Usage

* Add VM options:  -javaagent:/your/path/to/MyPerf4J-ASM-${MyPerf4J-version}.jar
* Add VM options: -DMyPerf4JPropFile=/your/path/to/myPerf4J.properties, and add properties in `/your/path/to/myPerf4J.properties`

```
#configure PerfStatsProcessor
PerfStatsProcessor=cn.perf4j.test.profiler.MyPerfStatsProcessor

#configure RecordMode，accurate/rough
RecorderMode=accurate

#configure TimeSlice，time unit: ms，min:30s，max:600s
MillTimeSlice=60000

#configure packages，separated with ';'
IncludePackages=cn.perf4j;org.myperf4j

#configure packages，separated with ';'
ExcludePackages=org.spring;

#print debug，true/false
Debug.PrintDebugLog=true

#configure methods，separated with ';'
ExcludeMethods=equals;hash

#true/false
ExcludePrivateMethod=true

#separated with ';'
ExcludeClassLoaders=

#The configuration file performs memory usage tuning by specifying certain method execution time thresholds
ProfilingParamsFile=/your/path/to/myPerf4J.profilingParams

#General method execution time threshold in ms
ProfilingTimeThreshold=1000

#The number of times the method execution time threshold is exceeded in a time slice, valid only when RecorderMode=accurate
ProfilingOutThresholdCount=10
```

* If you want to optimize the memory usage of MyPerf4J, you can specify ProfilingParamsFile in `/your/path/to/myPerf4J.properties` and specify the execution time threshold of the specific method in `/your/path/to/myPerf4J.profilingParams`:

```
#The format is: fullClassName.methodName=methodExecutionTimeThreshold:numberOfTimesTheMethodExecutionTimeThresholdIsExceeded
cn.perf4j.demo.UserServiceImpl.getId1=10:10
cn.perf4j.demo.UserServiceImpl.getId2=20:20
```

* If you need to customize performance statistics, you need to add dependencies in the `pom.xml` file and create a new class that implements the PerfStatsProcessor interface.
    - Add maven dependency in `pom.xml`
    
        ```
        <dependencies>
            <dependency>
                <groupId>MyPerf4J</groupId>
                <artifactId>MyPerf4J-Base</artifactId>
                <version>${MyPerf4J-version}</version>
            </dependency>
        </dependencies>
        ```

    - Create a new class that implements the PerfStatsProcessor interface.
    
        ``` 
        package cn.perf4j.demo;
        
        import cn.myperf4j.base.PerfStats;
        import cn.myperf4j.base.PerfStatsFormatter;
        import cn.myperf4j.base.PerfStatsProcessor;
        
        import java.util.List;
        
        /**
         * Created by LinShunkang on 2018/4/9
         */
        public class MyPerfStatsProcessor implements PerfStatsProcessor {
        
            @Override
            public void process(List<PerfStats> perfStatsList, long startMillis, long stopMillis) {
                //You can do anything you want to do :)
                System.out.println(PerfStatsFormatter.getFormatStr(perfStatsList, startMillis, stopMillis));
            }
        
        }
        ```
        

* MyPerf4J will perform performance analysis on all classes of the `IncludePackages` specified package, and exclude `ExcludePackages` from specifying all classes of the package.


```
package cn.perf4j.demo;

import java.util.concurrent.TimeUnit;

/**
 * Created by LinShunkang on 2018/4/7
 */
public class UserServiceImpl implements UserService {

    private long f1;

    @Override
    public long getId1(long id) throws InterruptedException {
        TimeUnit.MILLISECONDS.sleep(4);
        return id + 100;
    }

    private long privateGetId1() {
        return 1L;
    }

    @Override
    public long getId2(long id) {
        return id + 2;
    }

    @Override
    public long getId3(long id) {
        return 0;
    }

    public long getF1() {
        return f1;
    }

    public void setF1(long f1) {
        this.f1 = f1;
    }
}
```

* If you need to customize the performance statistics, create a new class and implements `PerfStatsProcessor`

``` 
package cn.perf4j.demo;


import cn.myperf4j.base.PerfStats;
import cn.myperf4j.base.PerfStatsFormatter;
import cn.myperf4j.base.PerfStatsProcessor;

import java.util.List;

/**
 * Created by LinShunkang on 2018/4/9
 */
public class MyPerfStatsProcessor implements PerfStatsProcessor {

    @Override
    public void process(List<PerfStats> perfStatsList, long startMillis, long stopMillis) {
        //You can do anything you want to do :)
        System.out.println(PerfStatsFormatter.getFormatStr(perfStatsList, startMillis, stopMillis));
    }

}
```

* Execute command `mvn clean package`

* Run your application

* Performance Statistics

```
MyPerf4J Performance Statistics [2018-06-09 16:40:00, 2018-06-09 16:40:30]
Api                         RPS  Avg(ms)  Min(ms)  Max(ms)   StdDev     Count     TP50     TP90     TP95     TP99    TP999   TP9999  TP99999    TP100
UserServiceImpl.getId1       62     4.60        4        5     0.02      1922        5        5        5        5        5        5        5        5
UserServiceImpl.getId2       62     0.00        0        0     0.00      1922        0        0        0        0        0        0        0        0
```

## About Rough Mode and Accurate Mode
* Rough Mode
    - The accuracy is slightly worse, and it will discard the record whose response time exceeds the specified threshold.
    - It saves more memory, and only uses array to record response time.
    - The speed is a little faster.
    - Default mode.

* Accurate Mode
    - High accuracy, records all response times.
    - It consumes relatively memory and uses array & Map to record response time.
    - The speed is slightly slower.
    - Need to add property RecorderMode=accurate in `/your/path/to/myPerf4J.properties`.

* Suggestions
    - For memory-sensitive or precision applications that are not particularly demanding, Rough Mode is recommended.
    - The Accurate Mode is recommended for applications that are insensitive to memory and require high accuracy.

## About MyPerf4J-ASM
* MyPerf4J-ASM
    * Implement aspect weaving using [ASM](http://asm.ow2.io/)
    * From a technical perspective, it is not mature enough,
    * Extremely fast! 
    * Does not produce any unnecessary objects, does not affect the program's own GC.
