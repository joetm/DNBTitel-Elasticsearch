# DNBTitel-Elasticsearch

(Update · May 25, 2016 · We just published a blog post with background info and some first results: ["Empirical Data on Over-Length Books"](http://weltliteratur.net/Empirical-Data-on-Over-Length-Books/). – *FF*)

This Docker project will create a container running Elasticsearch/Kibana. It also features shell scripts for downloading [the current version of the German National Library (DNB) title catalogue](http://datendienst.dnb.de/cgi-bin/mabit.pl?userID=opendata&pass=opendata&cmd=login). Some selected data fields from every book in that catalogue are then transformed into JSON and pushed to the Elasticsearch instance. After that you will be able to query the DNB catalogue data with Elasticsearch to create nice outputs with Kibana. The data fields we're focusing on are mainly the number of pages per book and some book metadata (author, title, year, publisher, etc.) for identification.

# Docker

## Build the Docker Image

Clone this repo to your hard drive and build a Docker image by typing: 

    docker build -t dnb-titel .

This will do the steps defined in the Dockerfile: create a clean Debian container and install some stuff we need (curl, java, wget, etc.). Elasticsearch and Kibana will be downloaded and set up with according environments. Once this is done we can go on and …

## Create Persistent Volumes

    docker volume create --name dnb2es
    docker volume create --name esindex

## Start Docker Image

    docker run --rm -ti -p 9200:9200 -p 5601:5601 -v dnb2es:/home/elasticsearch/dnb2es -v esindex:/home/elasticsearch/elasticsearch dnb-titel:latest

After this we have a working Docker container and can query Elasticsearch with Kibana/Sense via http://localhost:5601. However, our Elasticsearch index is still empty. It will be filled with DNB catalogue data during the next step:

## Download Data from German National Library (DNB) Catalogue, Transform It to JSON and Fill Elasticsearch

After typing …

    docker ps

… you will have to look for the Docker instance in question and execute the following:

    docker exec [docker-instance-name-from-ps] dnb2es/doItAll.sh

This will call all sub scripts assembled in our "doItAll" file:
 * getDnbTitel.sh
 * split.sh
 * convert.sh
 * createDnbIndex.sh
 * json2es.sh

Now we're ready to go and carry out some queries.

# Using Kibana

Now head to the Kibana dashboard: `http://localhost:5601`. If you never worked with Kibana, check out their guide, for our purposes it's perfectly okay to start with the chapter *[Defining Your Index Patterns](https://www.elastic.co/guide/en/kibana/4.3/tutorial-define-index.html)*.

Next, in the Kibana dashboard, go to "Settings", "Indices", enter "dnb" as name for the index pattern and don't forget to uncheck the box "Index contains time-based events". Then click on "Create".

Just to give you an example at this point, click on "Visualize". Then "Vertical bar chart". Select "From a new search" (index "dnb"). Now you can graphically create your query. For example, to find the 5 authors with the most titles in the catalogue, do this: "X-Axis", Aggregation "Terms", choose the field "creator.untouched". Then press the green Play button. There you will see the 5 authors who are attributed the most titles/books in the DNB catalogue. They appear as their GND numbers that easily be resolved: 1. Goethe, 2. Rudolf Steiner, 3. Thomas Mann, 4. Hermann Hesse, 5. Heinz G. Konsalik.

# Using Sense

Go to the Sense plug-in in Kibana and set the server to `http://localhost:9200/dnb`, then write your query:

    GET _search?search_type=count
    {
       "aggs" : {
           "suhrkamp_pages" : {
               "filter" : { "term": { "publisher.untouched": "Suhrkamp" } },
               "aggs" : {
                   "page_stats" : { "stats" : { "field" : "pages_norm" } }
               }
           }
       }
    }

This will produce the following output:

    {
      "took": 110,
      "timed_out": false,
      "_shards": {
        "total": 5,
        "successful": 5,
        "failed": 0
      },
      "hits": {
        "total": 11373859,
        "max_score": 0,
        "hits": []
      },
      "aggregations": {
        "suhrkamp_pages": {
          "doc_count": 22325,
          "page_stats": {
            "count": 20209,
            "min": 3,
            "max": 3980,
            "avg": 268.5873125835024,
            "sum": 5427881
          }
        }
      }
    }

Another query will answer our initial question:

    GET _search?search_type=count
    {
       "aggs" : {
          "page_stats" : { "stats" : { "field" : "pages_norm" } }
       }
    }

Output:

    {
      "took": 228,
      "timed_out": false,
      "_shards": {
        "total": 5,
        "successful": 5,
        "failed": 0
      },
      "hits": {
        "total": 11373859,
        "max_score": 0,
        "hits": []
      },
      "aggregations": {
        "page_stats": {
          "count": 5874504,
          "min": 0,
          "max": 2711111,
          "avg": 165.09413730929452,
          "sum": 969846170
        }
      }
    }


