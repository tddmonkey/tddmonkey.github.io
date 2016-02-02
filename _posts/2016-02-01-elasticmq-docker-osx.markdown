---
layout: post
title:  "Test SQS with ElasticMq and Docker on OSX"
date:   2016-02-01 19:17:44
categories: testing sqs docker elasticmq
---
First off I'd like to say that both ElasticMq and Docker are both awesome products and a Godsend when it comes to testing code against SQS.  Unfortunately getting them playing nice can be a bit of a pain, especially if you're using a non Linux host (like I am) and don't quite understand the relationship between Docker networking and how ElasticMq serves queue urls.

I'll assume you're familiar with ElasticMq and Docker so we can get straight to the good stuff.

Running ElasticMq standalone is as easy as just running the jar file...
{% highlight bash %}
$ java -jar elasticmq-0.0.8.jar
{% endhighlight %}
Now you have a local instance of ElasticMq running which you can interact with just like you would SQS.
{% highlight java %}
AmazonSQSClient client = new AmazonSQSClient(credentials);
client.setEndpoint("http://localhost:9324");
String queueUrl = client.createQueue("test-queue").getQueueUrl();
System.out.println("Queue url is " + queueUrl);
client.deleteQueue(queueUrl);
{% endhighlight %}
This will create a queue, print queue url and then delete the queue again, all successfully and finishing with...
{% highlight bash %}
Queue url is http://localhost:9324/queue/test-queue
{% endhighlight %}

All good so far.  As part of my automated builds I don't want to have to start ElasticMq by hand, so I'll include it as part of my Docker setup.  Again this is easy, so we fire up the container
{% highlight bash %}
$ docker run -p 9324:9324 tddmonkey/elasticmq
{% endhighlight %}
And re-run the code with AWS endpoint now using the boot2docker or docker-machine IP.
{% highlight java %}
client.setEndpoint("http://192.168.99.100:9324");
{% endhighlight %}
Everything should work fine yes? 
{% highlight bash %}
Queue url is http://172.17.0.27:9324/queue/test-queue
Feb 02, 2016 7:44:54 PM com.amazonaws.http.AmazonHttpClient executeHelper
INFO: Unable to execute HTTP request: Connect to 172.17.0.27:9324 timed out
org.apache.http.conn.ConnectTimeoutException: Connect to 172.17.0.27:9324 timed out
{% endhighlight %}
D'oh!

The problem here is that the semantics of SQS are that you

1. Set the region endpoint you wish to use
2. Retrieve a queue url (either by creating for one or asking for it by name)
3. Perform all operations against that queue url

ElasticMq works by using what is configured in the `node-address` configuration block to generate queue urls, for example...
{% highlight scala %}
node-address {
    protocol = http
    host = localhost
    port = 9324
    context-path = ""
}
{% endhighlight %}

When running inside Docker of course the IP address of the localhost is NOT something you can reach as it's the IP address of the container, whereas you need the IP address of your boot2docker or docker-machine VM which is determined by your system and is different for every member of your team.

Fixing the Queue Url
--------------------
The `tddmonkey/elasticmq` Docker image can be started with an environment variable to dynamically set the node host.  This is done at boot time and can be done like this:
{% highlight bash %}
$ docker run -e NODE_HOST=`docker-machine ip default` -p 9324:9324 tddmonkey/elasticmq
{% endhighlight %}
(replace with boot2docker if you're going old school)

The queue urls returned will now have the correct IP for addressing queues, and the code presented earlier will start working again.  

As part of my automated test suites I can bring up several instances of ElasticMq and generally use different ports for them.  Mapping the port to something other than the default leads us back to the same problem as we had with the host - an in valid queue url.  As with the host, the port can also be provided as the `NODE_PORT` environment variable, like so:
{% highlight bash %}
$ docker run -e NODE_HOST=`docker-machine ip default` -e NODE_PORT=8000 -p 8000:9324 tddmonkey/elasticmq
{% endhighlight %}
The important thing to note here is that the `NODE_PORT` being used is the same as the **host** port being mapped to.  The container port is always 9324. Both the `NODE_HOST` and the `NODE_PORT` are what ElasticMq will return for queue urls.

Thanks to [chr0n1x] for the original implementation of the [Docker container][behance-elasticmq-docker] which gave me the inspiration to do this version.  If you want to see the Docker configuration it's available [here][tddmonkey-elasticmq-docker].

[chr0n1x]: https://github.com/chr0n1x
[behance-elasticmq-docker]: https://github.com/behance/elasticmq-docker
[tddmonkey-elasticmq-docker]:http://github.com/tddmonkey/elasticmq-docker 
