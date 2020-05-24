---
title: "How to build a Python alternative to Logstash"
date: 2020-05-22T10:09:23-05:00
draft: false
toc: false
images:
tags:
- python
- asyncio
- concurrency
- docker
---
Python's asyncio library provides tools for writing high-performance concurrent code using the async/await syntax. In this post we'll explore asynchronous programming by building a lightweight alternative to [Logstash](https://www.elastic.co/logstash), a data processing pipeline that ingests data from different sources, transforms it, and then sends it to a data stash. The pipeline will run in a docker container using the [sidecar pattern](https://docs.microsoft.com/en-us/azure/architecture/patterns/sidecar). It'll fetch data (container logs) from a running container, structure it using regular expressions, and ship it to an Elasticsearch instance. Some parts of the program were omitted for clarity. The full example can be found on [Github](). Let's get started.

We'll start by creating a dockerfile for the log producer and consumer containers.

*Dockerfile*
``` yaml
FROM python:3.8-slim

WORKDIR /code

RUN pip install -U uvloop aiohttp urllib3 docker

COPY . .

```
Next we'll use Docker Compose to set up a multi-node Elasticsearch cluster and also include producer and consumer services to emit and consume logs.

*docker-compose.yml*
``` yaml
version: '3.8'
services:
  producer:
    image: base
    build: .
    container_name: prod1
    volumes:
      - .:/code
    command: python producer.py
    environment:
      - PYTHONUNBUFFERED=1
    networks:
      - elastic
  consumer:
    image: base
    volumes:
      - .:/code
      - /var/run/docker.sock:/var/run/docker.sock
      - /var/lib/docker:/var/lib/docker
    command: python consumer.py
    environment:
      - PYTHONUNBUFFERED=1
      - SOURCE_CONTAINER=prod1
    networks:
      - elastic
  es01:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.6.2
    container_name: es01
    environment:
      - node.name=es01
      - cluster.name=es-docker-cluster
      - discovery.seed_hosts=es02,es03
      - cluster.initial_master_nodes=es01,es02,es03
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - data01:/usr/share/elasticsearch/data
    ports:
      - 9200:9200
    networks:
      - elastic
  es02:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.6.2
    container_name: es02
    environment:
      - node.name=es02
      - cluster.name=es-docker-cluster
      - discovery.seed_hosts=es01,es03
      - cluster.initial_master_nodes=es01,es02,es03
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - data02:/usr/share/elasticsearch/data
    networks:
      - elastic
  es03:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.6.2
    container_name: es03
    environment:
      - node.name=es03
      - cluster.name=es-docker-cluster
      - discovery.seed_hosts=es01,es02
      - cluster.initial_master_nodes=es01,es02,es03
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - data03:/usr/share/elasticsearch/data
    networks:
      - elastic
volumes:
  data01:
    driver: local
  data02:
    driver: local
  data03:
    driver: local

networks:
  elastic:
    driver: bridge
```
This Docker Compose file brings up two services using the base image built from the Dockerfile above, and three Elasticsearch services to make up a three-node Elasticsearch cluster. The Elasticsearch cluster setup is the same as the one in the [official guide.](https://www.elastic.co/guide/en/elasticsearch/reference/current/docker.html) Next we'll go ahead and create ```producer.py``` and ```consumer.py```. You can download the sample data set used in this example [here.](https://download.elastic.co/demos/logstash/gettingstarted/logstash-tutorial.log.gz)

*producer.py*
``` python
import time
from pathlib import Path

path = "logstash-tutorial.log"
text = Path(path).read_text().split("\n")
while True:
    for line in text:
         print(line)
         time.sleep(0.1)
```
The producer simply reads and pushes logs to ```stdout```. But instead of opening and reading the same file repeatedly, we'll load the file into memory (it's only 24kb) once and iterate over the text. Run the program to see the output.

```
$ python producer.py
83.149.9.216 - - [04/Jan/2015:05:13:42 +0000] "GET /presentations/logstash-monitorama-2013/images/kibana-search.png HTTP/1.1" 200 203023 "http://semicomplete.com/presentations/logstash-monitorama-2013/" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_9_1) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/32.0.1700.77 Safari/537.36"
83.149.9.216 - - [04/Jan/2015:05:13:42 +0000] "GET /presentations/logstash-monitorama-2013/images/kibana-dashboard3.png HTTP/1.1" 200 171717 "http://semicomplete.com/presentations/logstash-monitorama-2013/" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_9_1) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/32.0.1700.77 Safari/537.36"
83.149.9.216 - - [04/Jan/2015:05:13:44 +0000] "GET /presentations/logstash-monitorama-2013/plugin/highlight/highlight.js HTTP/1.1" 200 26185 "http://semicomplete.com/presentations/logstash-monitorama-2013/" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_9_1) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/32.0.1700.77 Safari/537.36"
83.149.9.216 - - [04/Jan/2015:05:13:44 +0000] "GET /presentations/logstash-monitorama-2013/plugin/zoom-js/zoom.js HTTP/1.1" 200 7697 "http://semicomplete.com/presentations/logstash-monitorama-2013/" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_9_1) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/32.0.1700.77 Safari/537.36"
83.149.9.216 - - [04/Jan/2015:05:13:45 +0000] "GET /presentations/logstash-monitorama-2013/plugin/notes/notes.js HTTP/1.1" 200 2892 "http://semicomplete.com/presentations/logstash-monitorama-2013/" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_9_1) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/32.0.1700.77 Safari/537.36"
83.149.9.216 - - [04/Jan/2015:05:13:42 +0000] "GET /presentations/logstash-monitorama-2013/images/sad-medic.png HTTP/1.1" 200 430406 "http://semicomplete.com/presentations/logstash-monitorama-2013/" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_9_1) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/32.0.1700.77 Safari/537.36"
83.149.9.216 - - [04/Jan/2015:05:13:45 +0000] "GET /presentations/logstash-monitorama-2013/css/fonts/Roboto-Bold.ttf HTTP/1.1" 200 38720 "http://semicomplete.com/presentations/logstash-monitorama-2013/" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_9_1) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/32.0.1700.77 Safari/537.36"
83.149.9.216 - - [04/Jan/2015:05:13:45 +0000] "GET /presentations/logstash-monitorama-2013/css/fonts/Roboto-Regular.ttf HTTP/1.1" 200 41820 "http://semicomplete.com/presentations/logstash-monitorama-2013/" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_9_1) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/32.0.1700.77 Safari/537.36"
83.149.9.216 - - [04/Jan/2015:05:13:45 +0000] "GET /presentations/logstash-monitorama-2013/images/frontend-response-codes.png HTTP/1.1" 200 52878 "http://semicomplete.com/presentations/logstash-monitorama-2013/" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_9_1) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/32.0.1700.77 Safari/537.36"
83.149.9.216 - - [04/Jan/2015:05:13:43 +0000] "GET /presentations/logstash-monitorama-2013/images/kibana-dashboard.png HTTP/1.1" 200 321631 "http://semicomplete.com/presentations/logstash-monitorama-2013/" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_9_1) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/32.0.1700.77 Safari/537.36"
83.149.9.216 - - [04/Jan/2015:05:13:46 +0000] "GET /presentations/logstash-monitorama-2013/images/Dreamhost_logo.svg HTTP/1.1" 200 2126 "http://semicomplete.com/presentations/logstash-monitorama-2013/" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_9_1) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/32.0.1700.77 Safari/537.36"
83.149.9.216 - - [04/Jan/2015:05:13:43 +0000] "GET /presentations/logstash-monitorama-2013/images/kibana-dashboard2.png HTTP/1.1" 200 394967 "http://semicomplete.com/presentations/logstash-monitorama-2013/" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_9_1) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/32.0.1700.77 Safari/537.36"
83.149.9.216 - - [04/Jan/2015:05:13:46 +0000] "GET /presentations/logstash-monitorama-2013/images/apache-icon.gif HTTP/1.1" 200 8095 "http://semicomplete.com/presentations/logstash-monitorama-2013/" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_9_1) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/32.0.1700.77 Safari/537.36"

```
Before we start building the consumer service it's important to understand how Docker manages container logs. Because containers are ephemeral, logs emitted to ```stdout``` and ```stderr``` streams are stored on the Docker host. By default Docker annotates the logs with the log source (```stdout``` or ```stderr```) and timestamp, and stores them as JSON in the ```/var/lib/docker/containers/``` directory. Logs can take up a lot of disk space on the host over time so log management pipelines like the [ELK](https://www.elastic.co/what-is/elk-stack) stack are used to store and visualize logs. In our case we're replacing the L in ELK with our own log shipper, ```consumer.py```. First let's define some regex patterns we want to match.
```python
INT = '(?:[+-]?(?:[0-9]+))'
IP = '(?:(?:25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.){3}(?:25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)'
USER = '[a-zA-Z0-9._-]+'
MONTHDAY =  '(?:(?:0[1-9])|(?:[12][0-9])|(?:3[01])|[1-9])'
MONTH = '(?:Jan(?:uary)?|Feb(?:ruary)?|Mar(?:ch)?|Apr(?:il)?|May|Jun(?:e)?|Jul(?:y)?|Aug(?:ust)?|Sep(?:tember)?|Oct(?:ober)?|Nov(?:ember)?|Dec(?:ember)?)'
YEAR = '(?:\d\d){1,2}'
HOUR = '(?:2[0123]|[01]?[0-9])'
MINUTE = '(?:[0-5][0-9])'
SECOND = '(?:(?:[0-5]?[0-9]|60)(?:[:.,][0-9]+)?)'
TIME = f'(?!<[0-9]){HOUR}:{MINUTE}(?::{SECOND})(?![0-9])'
HTTPDATE = f'{MONTHDAY}/{MONTH}/{YEAR}:{TIME} {INT}'
VERSION = '[0-9]+(?:(?:\.[0-9])+)?'
REQUEST = f'\\"(?:\w+ \S+(?: HTTP/{VERSION})?|.*?)\\"'
TIMESTAMP = f"\[{HTTPDATE}\]"
QUOTE = f'(?:\\".+\\")'
COMMONLOG = r" ".join([f'(?P<ip>{IP})',f'(?P<ident>{USER})',f'(?P<auth>{USER})',f'(?P<timestamp>{TIMESTAMP})',f'(?P<request>{REQUEST})',"(?P<status>\d+)", "(?P<bytes>\d+|-)", f'(?P<referrer>{QUOTE})', f'(?P<agent>{QUOTE})'])
```
Then we'll write a ``reader`` and a ```worker``` function to read and ship the logs. We'll also have a queue as a message broker between the reader and workers. We can read the log files from the host asynchronously, using a non blocking stream generator and asyncio queues but if we want the reader to be able to get logs in real time, without yielding to the event loop, it must be available at all times. So instead the reader will run in a separate thread and ingest log streams using the Docker client library.  

*consumer.py*
``` python

pattern = re.compile(COMMONLOG)
queue = queue.Queue()

async def worker(name, client):
    log = logging.getLogger(name)
    while True:
        try:
            line = queue.get_nowait()
        except Empty as e:
            log.info('Queue is empty.')
            await asyncio.sleep(1)
        else:
            line = line.decode('utf-8')
            match = pattern.match(line)
            if match is None:
                log.info(f"No match found for {line.strip()}")
                await asyncio.sleep(0.001)
            else:
                async with client.post("http://es01:9200/logs/_doc/",
                               data=json.dumps(match.groupdict()).encode('utf-8'),
                              headers={'Content-Type': 'application/json'}
                          ) as resp:
                    if resp.status != 201:
                         err = await resp.text()
                         log.info(f"{resp.status}: {err}")
                    else:
                        log.info("Upload succesful.")


def reader(container):
    log = logging.getLogger('reader')
    stream = container.logs(stream=True)
    while True:
        try:
            queue.put_nowait(next(stream))
        except StopIteration as e:
            log.debug("No more logs")
            time.sleep(0.01)

async def main():
    c_name = os.getenv('SOURCE_CONTAINER')
    if not c_name:
        print("You must specify a source container name.")
        sys.exit(1)
    container = get_container(c_name)
    threading.Thread(target=reader, args = (container,), daemon=True).start()
    async with aiohttp.ClientSession() as client:
        done, pending = await asyncio.wait([asyncio.create_task(worker(f"worker{i}", client)) for i in range(5)])


if __name__ == '__main__':
    logging.basicConfig(
        level=logging.DEBUG,
        format='%(threadName)s %(name)s: %(message)s',
        stream=sys.stderr,
    )
    asyncio.run(main())

```
The ``` container.logs ``` function returns a blocking generator that we can iterate over to retrieve log outputs as they're emitted. The worker coroutine reads from the queue and sends matching logs to Elasticsearch. Let's run it using 5 worker coroutines and see the output.

```
$ docker-compose build producer && docker-compose up
.
.
.
prod1       | 50.150.204.184 - - [04/Jan/2015:05:17:06 +0000] "GET /images/googledotcom.png HTTP/1.1" 200 65748 "http://www.google.com/search?q=https//:google.com&source=lnms&tbm=isch&sa=X&ei=4-r8UvDrKZOgkQe7x4CICw&ved=0CAkQ_AUoAA&biw=320&bih=441" "Mozilla/5.0 (Linux; U; Android 4.0.4; en-us; LG-MS770 Build/IMM76I) AppleWebKit/534.30 (KHTML, like Gecko) Version/4.0 Mobile Safari/534.30"
prod1       | 207.241.237.225 - - [04/Jan/2015:05:17:35 +0000] "GET /blog/tags/examples HTTP/1.0" 200 9208 "http://www.semicomplete.com/blog/tags/C" "Mozilla/5.0 (compatible; archive.org_bot +http://www.archive.org/details/archive.org_bot)"
prod1       | 200.49.190.101 - - [04/Jan/2015:05:17:39 +0000] "GET /reset.css HTTP/1.1" 200 1015 "-" "-"
es01        | {"type": "server", "timestamp": "2020-05-18T23:21:43,366Z", "level": "WARN", "component": "o.e.c.r.a.AllocationService", "cluster.name": "es-docker-cluster", "node.name": "es01", "message": "[logs][0] marking unavailable shards as stale: [Oablme2xQHK4NJrYLLaP1Q]", "cluster.uuid": "6gFHNbH6RjuVwXEHwep55w", "node.id": "MFQQPkIsSpOgH_6prsOpYg"  }
prod1       | 200.49.190.100 - - [04/Jan/2015:05:17:37 +0000] "GET /blog/tags/web HTTP/1.1" 200 44019 "-" "QS304 Profile/MIDP-2.0 Configuration/CLDC-1.1"
prod1       | 200.49.190.101 - - [04/Jan/2015:05:17:41 +0000] "GET /style2.css HTTP/1.1" 200 4877 "-" "-"
prod1       | 200.49.190.101 - - [04/Jan/2015:05:17:48 +0000] "GET /images/jordan-80.png HTTP/1.1" 200 6146 "-" "QS304 Profile/MIDP-2.0 Configuration/CLDC-1.1"
consumer_1  | MainThread worker1: Upload succesful.
consumer_1  | MainThread worker0: Upload succesful.
consumer_1  | MainThread worker3: Upload succesful.
consumer_1  | MainThread worker2: Upload succesful.
prod1       | 66.249.73.185 - - [04/Jan/2015:05:18:48 +0000] "GET /reset.css HTTP/1.1" 200 1015 "-" "Mozilla/5.0 (compatible; Googlebot/2.1; +http://www.google.com/bot.html)"
consumer_1  | MainThread worker4: Upload succesful.
consumer_1  | MainThread worker2: Upload succesful.
consumer_1  | MainThread worker0: Upload succesful.
prod1       | 66.249.73.135 - - [04/Jan/2015:05:18:55 +0000] "GET /blog/tags/munin HTTP/1.1" 200 9746 "-" "Mozilla/5.0 (compatible; Googlebot/2.1; +http://www.google.com/bot.html)"
consumer_1  | MainThread worker1: Upload succesful.
consumer_1  | MainThread worker3: Upload succesful.
consumer_1  | MainThread worker4: Upload succesful.
consumer_1  | MainThread worker2: Upload succesful.
consumer_1  | MainThread worker0: Upload succesful.
.
.
.

```
Let's recap. So far we have a producer service that emits 10 logs every second and a consumer service that asynchronously processes and ships logs to a an Elasticsearch cluster. Here we've pre set the log frequency and the delay time in ```reader``` for illustrative purposes but in most real systems, logs are emitted at random or variable time intervals. Therefore log pipelines must be able to handle unexpected spikes in log volume or frequency. Let's add a ```controller``` coroutine that will monitor the service by creating worker tasks whenever the queue size (qsize) is increasing *and* is over a specific threshold.

``` python
async def controller(max_size=100):
    log = logging.getLogger('controller')
    async with aiohttp.ClientSession() as client:
        # start with 5 workers
        [asyncio.create_task(worker(f"worker{i}", client)) for i in range(5)]
        num = 5
        delay = 1
        curr_size = 0
        prev_size = 0
        while True:
            curr_size = queue.qsize()
            if curr_size > max_size and curr_size > prev_size:
                asyncio.create_task(worker(f"worker{num}", client))
                await asyncio.sleep(delay/100)
                delay += 1
                num += 1
            else:
                await asyncio.sleep(1)
                delay = 1
            prev_size = curr_size
            log.debug(f"Curently running {len(asyncio.all_tasks()) -2} workers. Queue size: {queue.qsize()}")

async def main():
    c_name = os.getenv('SOURCE_CONTAINER')
    if not c_name:
        print("You must specify a source container name.")
        sys.exit(1)
    container = get_container(c_name)
    threading.Thread(target=reader, args = (container,), daemon=True).start()
    done, pending = await asyncio.wait([asyncio.create_task(controller())])


```
Let's also change the delay in ```producer``` to simulate randomness of emitting logs.
```python
def main():
    while True:
        for line in text:
            print(line)
            time.sleep(random.random()*0.01)
```
Output:
```
$ docker-compose up
.
.
.
prod1       | 93.114.45.13 - - [04/Jan/2015:05:14:32 +0000] "GET /articles/dynamic-dns-with-dhcp/ HTTP/1.1" 200 18848 "http://www.google.ro/url?sa=t&rct=j&q=&esrc=s&source=web&cd=2&ved=0CCwQFjAB&url=http%3A%2F%2Fwww.semicomplete.com%2Farticles%2Fdynamic-dns-with-dhcp%2F&ei=W88AU4n9HOq60QXbv4GwBg&usg=AFQjCNEF1X4Rs52UYQyLiySTQxa97ozM4g&bvm=bv.61535280,d.d2k" "Mozilla/5.0 (X11; Linux x86_64; rv:25.0) Gecko/20100101 Firefox/25.0"
prod1       | 93.114.45.13 - - [04/Jan/2015:05:14:32 +0000] "GET /reset.css HTTP/1.1" 200 1015 "http://www.semicomplete.com/articles/dynamic-dns-with-dhcp/" "Mozilla/5.0 (X11; Linux x86_64; rv:25.0) Gecko/20100101 Firefox/25.0"
prod1       | 93.114.45.13 - - [04/Jan/2015:05:14:33 +0000] "GET /style2.css HTTP/1.1" 200 4877 "http://www.semicomplete.com/articles/dynamic-dns-with-dhcp/" "Mozilla/5.0 (X11; Linux x86_64; rv:25.0) Gecko/20100101 Firefox/25.0"
prod1       | 93.114.45.13 - - [04/Jan/2015:05:14:33 +0000] "GET /favicon.ico HTTP/1.1" 200 3638 "-" "Mozilla/5.0 (X11; Linux x86_64; rv:25.0) Gecko/20100101 Firefox/25.0"
consumer_1  | MainThread controller: Curently running 6 workers. Queue size: 325
prod1       | 93.114.45.13 - - [04/Jan/2015:05:14:33 +0000] "GET /images/jordan-80.png HTTP/1.1" 200 6146 "http://www.semicomplete.com/articles/dynamic-dns-with-dhcp/" "Mozilla/5.0 (X11; Linux x86_64; rv:25.0) Gecko/20100101 Firefox/25.0"
consumer_1  | MainThread controller: Curently running 7 workers. Queue size: 325
prod1       | 93.114.45.13 - - [04/Jan/2015:05:14:33 +0000] "GET /images/web/2009/banner.png HTTP/1.1" 200 52315 "http://www.semicomplete.com/style2.css" "Mozilla/5.0 (X11; Linux x86_64; rv:25.0) Gecko/20100101 Firefox/25.0"
.
.
.
consumer_1  | MainThread worker16: Upload succesful.
prod1       | 83.149.9.216 - - [04/Jan/2015:05:13:47 +0000] "GET /presentations/logstash-monitorama-2013/images/elasticsearch.png HTTP/1.1" 200 8026 "http://semicomplete.com/presentations/logstash-monitorama-2013/" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_9_1) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/32.0.1700.77 Safari/537.36"
consumer_1  | MainThread worker31: Upload succesful.
consumer_1  | MainThread worker24: Upload succesful.
consumer_1  | MainThread worker25: Upload succesful.
prod1       | 83.149.9.216 - - [04/Jan/2015:05:13:47 +0000] "GET /presentations/logstash-monitorama-2013/images/logstashbook.png HTTP/1.1" 200 54662 "http://semicomplete.com/presentations/logstash-monitorama-2013/" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_9_1) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/32.0.1700.77 Safari/537.36"
consumer_1  | MainThread worker10: Upload succesful.
prod1       | 83.149.9.216 - - [04/Jan/2015:05:13:47 +0000] "GET /presentations/logstash-monitorama-2013/images/github-contributions.png HTTP/1.1" 200 34245 "http://semicomplete.com/presentations/logstash-monitorama-2013/" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_9_1) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/32.0.1700.77 Safari/537.36"
consumer_1  | MainThread worker14: Upload succesful.
prod1       | 83.149.9.216 - - [04/Jan/2015:05:13:47 +0000] "GET /presentations/logstash-monitorama-2013/css/print/paper.css HTTP/1.1" 200 4254 "http://semicomplete.com/presentations/logstash-monitorama-2013/" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_9_1) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/32.0.1700.77 Safari/537.36"
consumer_1  | MainThread worker12: Upload succesful.
consumer_1  | MainThread worker19: Upload succesful.
prod1       | 83.149.9.216 - - [04/Jan/2015:05:13:47 +0000] "GET /presentations/logstash-monitorama-2013/images/1983_delorean_dmc-12-pic-38289.jpeg HTTP/1.1" 200 220562 "http://semicomplete.com/presentations/logstash-monitorama-2013/" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_9_1) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/32.0.1700.77 Safari/537.36"
consumer_1  | MainThread worker22: Upload succesful.
consumer_1  | MainThread worker38: Upload succesful.
consumer_1  | MainThread worker11: Upload succesful.
prod1       | 83.149.9.216 - - [04/Jan/2015:05:13:46 +0000] "GET /presentations/logstash-monitorama-2013/images/simple-inputs-filters-outputs.jpg HTTP/1.1" 200 1168622 "http://semicomplete.com/presentations/logstash-monitorama-2013/" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_9_1) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/32.0.1700.77 Safari/537.36"
consumer_1  | MainThread worker7: Upload succesful.
prod1       | 83.149.9.216 - - [04/Jan/2015:05:13:46 +0000] "GET /presentations/logstash-monitorama-2013/images/tiered-outputs-to-inputs.jpg HTTP/1.1" 200 1079983 "http://semicomplete.com/presentations/logstash-monitorama-2013/" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_9_1) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/32.0.1700.77 Safari/537.36"
prod1       | 83.149.9.216 - - [04/Jan/2015:05:13:53 +0000] "GET /favicon.ico HTTP/1.1" 200 3638 "-" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_9_1) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/32.0.1700.77 Safari/537.36"
consumer_1  | MainThread controller: Curently running 66 workers. Queue size: 1692
.
.
.
prod1       | 24.236.252.67 - - [04/Jan/2015:05:14:10 +0000] "GET /favicon.ico HTTP/1.1" 200 3638 "-" "Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:26.0) Gecko/20100101 Firefox/26.0"
prod1       | 93.114.45.13 - - [04/Jan/2015:05:14:32 +0000] "GET /articles/dynamic-dns-with-dhcp/ HTTP/1.1" 200 18848 "http://www.google.ro/url?sa=t&rct=j&q=&esrc=s&source=web&cd=2&ved=0CCwQFjAB&url=http%3A%2F%2Fwww.semicomplete.com%2Farticles%2Fdynamic-dns-with-dhcp%2F&ei=W88AU4n9HOq60QXbv4GwBg&usg=AFQjCNEF1X4Rs52UYQyLiySTQxa97ozM4g&bvm=bv.61535280,d.d2k" "Mozilla/5.0 (X11; Linux x86_64; rv:25.0) Gecko/20100101 Firefox/25.0"
consumer_1  | MainThread worker54: Upload succesful.
consumer_1  | MainThread worker56: Upload succesful.
prod1       | 93.114.45.13 - - [04/Jan/2015:05:14:32 +0000] "GET /reset.css HTTP/1.1" 200 1015 "http://www.semicomplete.com/articles/dynamic-dns-with-dhcp/" "Mozilla/5.0 (X11; Linux x86_64; rv:25.0) Gecko/20100101 Firefox/25.0"
consumer_1  | MainThread worker16: Upload succesful.
consumer_1  | MainThread worker54: Upload succesful.
consumer_1  | MainThread worker54: Queue is empty.
prod1       | 93.114.45.13 - - [04/Jan/2015:05:14:33 +0000] "GET /style2.css HTTP/1.1" 200 4877 "http://www.semicomplete.com/articles/dynamic-dns-with-dhcp/" "Mozilla/5.0 (X11; Linux x86_64; rv:25.0) Gecko/20100101 Firefox/25.0"
consumer_1  | MainThread worker56: Upload succesful.
prod1       | 93.114.45.13 - - [04/Jan/2015:05:14:33 +0000] "GET /favicon.ico HTTP/1.1" 200 3638 "-" "Mozilla/5.0 (X11; Linux x86_64; rv:25.0) Gecko/20100101 Firefox/25.0"
prod1       | 93.114.45.13 - - [04/Jan/2015:05:14:33 +0000] "GET /images/jordan-80.png HTTP/1.1" 200 6146 "http://www.semicomplete.com/articles/dynamic-dns-with-dhcp/" "Mozilla/5.0 (X11; Linux x86_64; rv:25.0) Gecko/20100101 Firefox/25.0"
consumer_1  | MainThread worker16: Upload succesful.
prod1       | 93.114.45.13 - - [04/Jan/2015:05:14:33 +0000] "GET /images/web/2009/banner.png HTTP/1.1" 200 52315 "http://www.semicomplete.com/style2.css" "Mozilla/5.0 (X11; Linux x86_64; rv:25.0) Gecko/20100101 Firefox/25.0"
consumer_1  | MainThread worker56: Upload succesful.
consumer_1  | MainThread worker16: Upload succesful.
prod1       | 66.249.73.135 - - [04/Jan/2015:05:15:03 +0000] "GET /blog/tags/ipv6 HTTP/1.1" 200 12251 "-" "Mozilla/5.0 (iPhone; CPU iPhone OS 6_0 like Mac OS X) AppleWebKit/536.26 (KHTML, like Gecko) Version/6.0 Mobile/10A5376e Safari/8536.25 (compatible; Googlebot/2.1; +http://www.google.com/bot.html)"
prod1       | 50.16.19.13 - - [04/Jan/2015:05:15:15 +0000] "GET /blog/tags/puppet?flav=rss20 HTTP/1.1" 200 14872 "http://www.semicomplete.com/blog/tags/puppet?flav=rss20" "Tiny Tiny RSS/1.11 (http://tt-rss.org/)"
prod1       | 66.249.73.185 - - [04/Jan/2015:05:15:23 +0000] "GET / HTTP/1.1" 200 37932 "-" "Mozilla/5.0 (compatible; Googlebot/2.1; +http://www.google.com/bot.html)"
consumer_1  | MainThread worker56: Upload succesful.
prod1       | 110.136.166.128 - - [04/Jan/2015:05:16:11 +0000] "GET /projects/xdotool/ HTTP/1.1" 200 12292 "http://www.google.com/url?sa=t&rct=j&q=&esrc=s&source=web&cd=5&cad=rja&sqi=2&ved=0CFYQFjAE&url=http%3A%2F%2Fwww.semicomplete.com%2Fprojects%2Fxdotool%2F&ei=6cwAU_bRHo6urAeI0YD4Ag&usg=AFQjCNE3V_aCf3-gfNcbS924S6jZ6FqffA&bvm=bv.61535280,d.bmk" "Mozilla/5.0 (Windows NT 6.2; WOW64; rv:28.0) Gecko/20100101 Firefox/28.0"
consumer_1  | MainThread worker16: Upload succesful.
prod1       | 46.105.14.53 - - [04/Jan/2015:05:16:17 +0000] "GET /blog/tags/puppet?flav=rss20 HTTP/1.1" 200 14872 "-" "UniversalFeedParser/4.2-pre-314-svn +http://feedparser.org/"
consumer_1  | MainThread worker49: Upload succesful.
prod1       | 110.136.166.128 - - [04/Jan/2015:05:16:22 +0000] "GET /reset.css HTTP/1.1" 200 1015 "http://www.semicomplete.com/projects/xdotool/" "Mozilla/5.0 (Windows NT 6.2; WOW64; rv:28.0) Gecko/20100101 Firefox/28.0"
consumer_1  | MainThread worker56: Upload succesful.
consumer_1  | MainThread worker56: Queue is empty.
.
.
.
```
The ```controller``` starts to spin up more workers when the queue has about 325 logs, and the number of workers peaks at around 1692 logs before the queue size slowly starts to decrease all the way down to 0. We can optimize the service further by using the bulk API to index multiple log documents in a single API call and using [aiologger](https://github.com/b2wdigital/aiologger) for non blocking logging. That's it for this post. Thank you for reading, I hope it was helpful :)
