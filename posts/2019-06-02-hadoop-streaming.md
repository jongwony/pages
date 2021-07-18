date: 2019-06-02 15:21:34
title: Hadoop streaming
tags: ['hadoop', 'streaming', 'map', 'reduce', 'python', 'docker']
layout: post

## Hadoop Docker

[sequenceiq/hadoop-docker](https://hub.docker.com/r/sequenceiq/hadoop-docker)

## Build the image

```
docker pull sequenceiq/hadoop-docker
```


```
docker run -it sequenceiq/hadoop-docker /etc/bootstrap.sh -bash
```

```bash
cd $HADOOP_PREFIX
# run the mapreduce
bin/hadoop jar share/hadoop/mapreduce/hadoop-mapreduce-examples-2.7.0.jar grep input output 'dfs[a-z.]+'

# check the output
bin/hdfs dfs -cat output/*
```

## Python3 with hadoop

- sequenceiq/hadoop-docker Dockerfile

```
FROM sequenceiq/pam:centos-6.5
```

- python install in centos 6.5

```
yum install -y centos-release-scl
yum install -y rh-python36
scl enable rh-python36 bash
ln -s `which python3` /usr/bin/python3
```

## Map reduce principle

```
bin/hdfs dfs -cat input/* | ./mapper.py | sort -k1 | ./reducer.py
```

## Python3 hadoop streaming example

- mapper.py

```python
#!/usr/bin/env python3
import sys

def main(argv):
    for line in sys.stdin:
        line = line.rstrip()
        words = line.split()
        for word in words:
            sys.stdout.write(word + '\t1\n')

if __name__ == '__main__':
    main(sys.argv)
```

- reducer.py

```python
#!/usr/bin/env python3
import sys
from operator import itemgetter
from itertools import groupby


def output():
    for line in sys.stdin:
        yield line.rstrip().split('\t', 1)

def main(argv):
    for cur_word, group in groupby(output(), key=itemgetter(0)):
        total_count = sum(int(count) for _, count in group)
        sys.stdout.write('{0}\t{1}\n'.format(cur_word, total_count))

if __name__ == '__main__':
    main(sys.argv)
```

```
chmod +x mapper.py reducer.py
```

- hadoop streaming

```
bin/hadoop jar share/hadoop/tools/lib/hadoop-streaming-2.7.0.jar \
    -files $HADOOP_PREFIX/mapper.py,$HADOOP_PREFIX/reducer.py
    -input input \
    -output output_python \
    -m mapper.py \
    -reducer reducer.py
```

- check output

```
bin/hdfs dfs -cat output_python/*
```

# References

- [How to install Python 3 on Red Hat Enterprise Linux](https://developers.redhat.com/blog/2018/08/13/install-python3-rhel/)
- [Hadoop Streaming Docs](https://hadoop.apache.org/docs/r1.2.1/streaming.html)
- [sequenceiq/hadoop-docker](https://hub.docker.com/r/sequenceiq/hadoop-docker)

