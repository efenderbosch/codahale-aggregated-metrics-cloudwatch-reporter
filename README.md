# CodaHale Aggregated Metrics CloudWatch Reporter

This is a CloudWatch Reporter for the stable version of Dropwizard Metrics (formerly CodaHale & Yammer Metrics). The reporter is an implementation of [ScheduledReporter](http://metrics.dropwizard.io/3.1.0/apidocs/com/codahale/metrics/ScheduledReporter.html) from Dropwizard Metrics v3.1. 

## Table of Contents

  - [Prerequisites](#prerequisites)
  - [Summary](#summary)
  - [Reportables](#reportables)
  - [Defaults](#defaults)
  - [Dependencies](#dependencies)
  - [Usage](#usage)
    - [Dry run](#dry-run)
  - [Credits](#credits)
  - [License](#license)

### Prerequisites

- Java 1.8

### Summary

- This CloudWatchReporter reports the metric data to CloudWatch asynchronously using the [AmazonCloudWatchAsyncClient (AWS)](http://docs.aws.amazon.com/AWSJavaSDK/latest/javadoc/com/amazonaws/services/cloudwatch/AmazonCloudWatchAsyncClient.html) interface 
- Each reportable value in CodeHale [Metric](http://metrics.dropwizard.io/3.1.0/apidocs/com/codahale/metrics/Metric.html) is reported as a separate [MetricDatum (AWS)](http://docs.aws.amazon.com/AWSJavaSDK/latest/javadoc/com/amazonaws/services/cloudwatch/model/MetricDatum.html) 
- When reporting [Meter](http://metrics.dropwizard.io/3.1.0/apidocs/com/codahale/metrics/Meter.html), [Counter](http://metrics.dropwizard.io/3.1.0/apidocs/com/codahale/metrics/Counter.html), [Histogram](http://metrics.dropwizard.io/3.1.0/apidocs/com/codahale/metrics/Histogram.html) and [Timer](http://metrics.dropwizard.io/3.1.0/apidocs/com/codahale/metrics/Timer.html) count metrics (`getCount()`) as [MetricDatum (AWS)](http://docs.aws.amazon.com/AWSJavaSDK/latest/javadoc/com/amazonaws/services/cloudwatch/model/MetricDatum.html), only the count difference since the last report is reported. This way the counters do not require a reset within the application using this reporter.
- If configured, each [Snapshot](http://metrics.dropwizard.io/3.1.0/apidocs/com/codahale/metrics/Snapshot.html) translated into [StatisticSet (AWS)](http://docs.aws.amazon.com/AWSJavaSDK/latest/javadoc/com/amazonaws/services/cloudwatch/model/StatisticSet.html) in the most direct way possible.
- If configured, JVM statistic is reported

### Reportables

Currently the only metric values that are reportable through configuration are:

- Values of type `Number` from [Gauge](http://metrics.dropwizard.io/3.1.0/apidocs/com/codahale/metrics/Gauge.html)
- Counts from [Counter](http://metrics.dropwizard.io/3.1.0/apidocs/com/codahale/metrics/Counter.html), [Histogram](http://metrics.dropwizard.io/3.1.0/apidocs/com/codahale/metrics/Histogram.html), [Meter](http://metrics.dropwizard.io/3.1.0/apidocs/com/codahale/metrics/Meter.html) and [Timer](http://metrics.dropwizard.io/3.1.0/apidocs/com/codahale/metrics/Timer.html)
- Percentiles from [Snapshot](http://metrics.dropwizard.io/3.1.0/apidocs/com/codahale/metrics/Snapshot.html) in [Histogram](http://metrics.dropwizard.io/3.1.0/apidocs/com/codahale/metrics/Histogram.html) and [Timer](http://metrics.dropwizard.io/3.1.0/apidocs/com/codahale/metrics/Timer.html)
- Arithmetic mean & standard deviation of [Snapshot](http://metrics.dropwizard.io/3.1.0/apidocs/com/codahale/metrics/Snapshot.html) values in [Histogram](http://metrics.dropwizard.io/3.1.0/apidocs/com/codahale/metrics/Histogram.html) and [Timer](http://metrics.dropwizard.io/3.1.0/apidocs/com/codahale/metrics/Timer.html)
- Mean rates from [Meter](http://metrics.dropwizard.io/3.1.0/apidocs/com/codahale/metrics/Meter.html) and [Timer](http://metrics.dropwizard.io/3.1.0/apidocs/com/codahale/metrics/Timer.html)
- Summaries of [Snapshot](http://metrics.dropwizard.io/3.1.0/apidocs/com/codahale/metrics/Snapshot.html) values in [Histogram](http://metrics.dropwizard.io/3.1.0/apidocs/com/codahale/metrics/Histogram.html) and [Timer](http://metrics.dropwizard.io/3.1.0/apidocs/com/codahale/metrics/Timer.html) as [StatisticSet (AWS)](http://docs.aws.amazon.com/AWSJavaSDK/latest/javadoc/com/amazonaws/services/cloudwatch/model/StatisticSet.html)

### Defaults

The Reporter uses the following defaults which can be configured:

- Rate metrics are in `TimeUnit.Seconds`
- Duration metrics are in `TimeUnit.Milliseconds`
- `MetricFilter.ALL` will be used for the Filter
- `Clock.defaultClock()` will be used for the Clock (Unconfigurable)
- Empty global [Dimension (AWS)](http://docs.aws.amazon.com/AWSJavaSDK/latest/javadoc/com/amazonaws/services/cloudwatch/model/Dimension.html) list
- The reporter adds a `Type` [Dimension (AWS)](http://docs.aws.amazon.com/AWSJavaSDK/latest/javadoc/com/amazonaws/services/cloudwatch/model/Dimension.html) to each reported metric, e.g:

| Type                                      | Metric Name                                                     |
| ----------------------------------------- | --------------------------------------------------------------- |
| 1-min-mean-rate   [per-second]            | com.example.component.SomeComponent.timer                       |
| snapshot-mean     [in-milliseconds]       | com.example.component.SomeComponent.timer                       |
| snapshot-mean                             | com.example.component.SomeComponent.histogram                   |
| 95%                                       | com.example.component.SomeComponent.timer                       |
| 99.5%                                     | com.example.component.SomeComponent.timer                       |
| 99.5%                                     | com.example.component.SomeComponent.histogram                   |
| count                                     | com.example.component.SomeComponent.counter                     |

The __only__ metrics that are reportable __by default__  are:

- Count values (`getCount()`) from [Meter](http://metrics.dropwizard.io/3.1.0/apidocs/com/codahale/metrics/Meter.html), [Counter](http://metrics.dropwizard.io/3.1.0/apidocs/com/codahale/metrics/Counter.html), [Histogram](http://metrics.dropwizard.io/3.1.0/apidocs/com/codahale/metrics/Histogram.html) and [Timer](http://metrics.dropwizard.io/3.1.0/apidocs/com/codahale/metrics/Timer.html) 
- Percentile values (`75%`, `95%`, `99.9%`) from [Histogram](http://metrics.dropwizard.io/3.1.0/apidocs/com/codahale/metrics/Histogram.html) and [Timer](http://metrics.dropwizard.io/3.1.0/apidocs/com/codahale/metrics/Timer.html)

All other metrics have to be confugured for reporting by invoking their respective `withXXXX()` methods on the `CloudWatchReporter.Builder` instance


### Dependencies

Gradle

```
repositories {
    mavenCentral()
}

dependencies { 
    compile("io.github.azagniotov:dropwizard-metrics-cloudwatch:1.0.2")
}
```

The library fetches the following transitive dependencies:

```
    io.dropwizard.metrics:metrics-core:3.1.0
    io.dropwizard.metrics:metrics-jvm:3.1.0
    com.amazonaws:aws-java-sdk-cloudwatch:1.11.86
    com.google.guava:guava:19.0
```



### Usage

The reporter provides a fine-grained configuration options through its builder to configure what metrics should be reported to CloudWatch. Since AWS costs money, you probably do not want to report `all` the values from [Metric](http://metrics.dropwizard.io/3.1.0/apidocs/com/codahale/metrics/Metric.html) classes or [Snapshot](http://metrics.dropwizard.io/3.1.0/apidocs/com/codahale/metrics/Snapshot.html), but only what's really useful to you.


```
    final AmazonCloudWatchAsync amazonCloudWatchAsync = new AmazonCloudWatchAsyncClient();
    amazonCloudWatchAsync.setRegion(Region.getRegion(Regions.US_WEST_2));

    final CloudWatchReporter cloudWatchReporter =
        CloudWatchReporter.forRegistry(metricRegistry, amazonCloudWatchAsync, Main.class.getName())
                .convertRatesTo(TimeUnit.SECONDS)
                .convertDurationsTo(TimeUnit.MILLISECONDS)
                .filter(MetricFilter.ALL)
                .withPercentiles(Percentile.P75, Percentile.P99)
                .withOneMinuteMeanRate()
                .withFiveMinuteMeanRate()
                .withFifteenMinuteMeanRate()
                .withMeanRate()
                .withArithmeticMean()
                .withStdDev()
                .withStatisticSet()
                .withJvmMetrics()
                .withGlobalDimensions("Region=us-west-2", "Instance=stage")
                .withDryRun()
                .build();

    cloudWatchReporter.start(10, TimeUnit.SECONDS);
```


#### Dry run
The reporter can be configured to run in `DRY RUN` mode by invoking `.withDryRun()` on the `Builder`. In that case, the reporter will `log.DEBUG` the created instance of [PutMetricDataRequest (AWS)](http://docs.aws.amazon.com/AWSJavaSDK/latest/javadoc/com/amazonaws/services/cloudwatch/model/PutMetricDataRequest.html), instead of doing a real `POST` to CloudWatch. 



### Credits
* https://github.com/blacklocus/metrics-cloudwatch
* https://github.com/tptodorov/metrics-cloudwatch
* https://github.com/basis-technology-corp/metrics-cloudwatch-reporter
* https://github.com/wavefrontHQ/java/tree/master/dropwizard-metrics/3.1


### License
MIT
