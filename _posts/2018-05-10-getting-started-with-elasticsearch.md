---
layout: post
title: "Getting started with ElasticSearch"
category: ElasticSearch
tags: [linux, elasticsearch]
---

## Introduction

I manage an internal build system that creates a simple text file with
key value pairs with build statistics. These statistics are then
processed using a fairly gnarly shell script. When I first saw this
years ago I thought it looked like the perfect candidate to use
ElasticSearch and finally had time to look into this.

## ElasticSearch and Kibana

ES is a text database and search engine which is useful, but it also
has a neat frontend called Kibana which can be used to query and
visualize the data. Since I manage the system, there was no need to
setup Logstash to preprocess the data since I could just convert it to
json myself.

## Official Docker Images

The documentation for ElasticSearch covers installation using Docker,
but there is one gotcha. The webpage that lists all the available
docker images at https://www.docker.elastic.co/ only lists the images
that contain the starter X-Pack with a 30 day trial license. I ended
up using the `docker.elastic.co/elasticsearch/elasticsearch-oss` image
which is only Open Source content. Same for the Kibana image.

## Docker-compose

I wanted to run ES and Kibana on the same server, but if you do that
using two separate docker run commands, the auto configuration of
Kibana doesn't work. I also wanted to make a local volume to hold the
data and so I created a simple docker-compose file:

    ---
    version: '2'
    services:
      kibana:
        image: docker.elastic.co/kibana/kibana-oss:6.2.4
        environment:
          SERVER_NAME: $HOSTNAME
          ELASTICSEARCH_URL: http://elasticsearch:9200
        ports:
          - 5601:5601
        networks:
          - esnet

      elasticsearch:
        image: docker.elastic.co/elasticsearch/elasticsearch-oss:6.2.4
        container_name: elasticsearch
        environment:
          discovery.type: single-node
          bootstrap.memory_lock: "true"
          ES_JAVA_OPTS: "-Xms512m -Xmx512m"
        ulimits:
          memlock:
            soft: -1
            hard: -1
        ports:
          - 9200:9200
        volumes:
          - esdata1:/usr/share/elasticsearch/data
        networks:
          - esnet

    volumes:
      esdata1:
      driver: local

    networks:
      esnet:

Now I have ES and Kibana running on ports 5601 and 9200.

## JSON output from Bash

I have a large collection of files in the form:

    key1: value1
    key2: value2
    <etc>

Converting this to JSON should be simple, but there were a few
surprises:

- JSON does not like single quotes so wrapping the key and value in
quotes requires either using echo and \" or printf which I found
cleaner.
- JSON requires that the last element does not have a trailing
comma. I abused bash control characters by using backspace to erase
the last comma.
- ElasticSearch would fail to parse if the JSON contained \( or \). I
used `tr` to delete all backslashes from the JSON.

The final code looks like this:

    convert_to_json()
    {
        local ARR=();
        local FILE=$1
        while read -r LINE
        do
            ARR+=( "${LINE%%:*}" "${LINE##*: }" )
        done < "$FILE"

        local LEN=${#ARR[@]}
        echo "{"
        for (( i=0; i<LEN; i+=2 ))
        do
            printf '  "%s": "%s",' "${ARR[i]}" "${ARR[i+1]}"
        done
        printf "\b \n}\n"
    }

    for FILE in "$@"; do
        local JSON=
        JSON=$(convert_to_json "$FILE" | tr -d '\\' )
        echo $JSON
    done

## ElasticSearch type mapping

The data was being imported into ES properly and I tried to search and
visualize the data, but I found it really hard to visualize the
data. Every field was imported as a text and keyword type which meant
that all the date and number fields could not be visualized as
expected.

The solution was to create a mapping which assigns types to each field
in the document. If the numbers had not been sent as strings, ES would
have converted them automatically, but I had dates in epoch seconds
which is indistinguishable from a large number. Date parsing is its
own challenge and ES supports many different date formats. In my
specific case, epoch_seconds was the only date format required.

I took the default mapping and added the type information to each
field.I tried to apply this mapping to the existing document, but ES
does not allow the mapping of a field to be changed because that would
change the interpretation of the data. The solution is to create a new
index and reindex the old index to the new one with types. This worked
and I was now able to visualize the data much more easily.

## Curator

I now had one index and it was growing quickly. I remembered from
previous research that LogStash uses indexes with a date suffix. This
allows data to be cleaned up regularly and also allows a new mapping
to applied to new indexes. Creating and deleting indexes is handled by
the Curator tool.

I created two scripts: one for deleting indexes that are 45 days old
and another for creating tomorrows index with the specified
mapping. Running these from cron daily will automate the creation and
cleanup. Last piece is to have the JSON sent to the index that matches
the day.

## Conclusion

Every new piece of software has its learning curve. So far the curve
has been quite reasonable for ElasticSearch. I look forward to working
with it more.
