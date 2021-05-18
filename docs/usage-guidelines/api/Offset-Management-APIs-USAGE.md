---
title: "Offset management"
date: 2019-03-26T08:47:11+01:00
draft: true
---
# Spark-Structured Streaming Offset Management APIs  
  
spark-structured streaming has it's own offset management it does not  
support storing consumed offset information in consumer group in kafka.  
It uses checkpoint location which you provide. we are using s3 as  
checkpoint location. Currently there is no APIs in spark-structured  
streaming using which we can get information of currently consumed  
offsets or reset offsets for current pipeline. We created APIs which  
will do all of this work.  
  
## Offset Management APIs  
  
**1.get currently consumed offsets by pipeline:**  
     The API gives information of currently consumed offset by pipeline. 

  **Request:**

    curl -X GET -H "Accept:application/json" -H "Content-Type:application/json" localhost:9090//v1/getCurrentOffsets/?pipelineName=orion_transaction  

   **Response:**

    {  
      "Orion.Transaction.info": {  
        "0": 873340,  
        "1": 884664,  
        "2": 879160,  
        "3": 866873,  
        "4": 894115,  
        "5": 880421,  
        "6": 868078,  
        "7": 885777,  
        "8": 876230  
      }  
    }  


**2.get topic details:**  
  The API gives detail of given topic like starting offset and ending offset of each partition.

  **Request :**

        curl -X GET -H "Accept:application/json" -H "Content-Type:application/json" localhost:9090//v1/getTopicDetails/?topicName=Orion.Transaction.info  

   **Response:**

    {  
      "name": "Orion.Transaction.info",  
      "partitions": {  
        "0": {  
          "firstOffset": 873340,  
          "lastOffset": 898067  
        },  
        "1": {  
          "firstOffset": 884664,  
          "lastOffset": 909803  
        },  
        "2": {  
          "firstOffset": 879160,  
          "lastOffset": 904819  
        },  
        "3": {  
          "firstOffset": 866873,  
          "lastOffset": 892826  
        },  
        "4": {  
          "firstOffset": 894115,  
          "lastOffset": 917581  
        },  
        "5": {  
          "firstOffset": 880421,  
          "lastOffset": 905385  
        },  
        "6": {  
          "firstOffset": 868078,  
          "lastOffset": 893108  
        },  
        "7": {  
          "firstOffset": 885777,  
          "lastOffset": 908980  
        },  
        "8": {  
          "firstOffset": 876230,  
          "lastOffset": 902482  
        }  
      }  
    }  


**3.reset offset of all partitions to earliest:**  
The API reset’s offset of the consumer to the earliest offset of each partition.

   **Request:**

    curl -X GET -H "Accept:application/json" -H "Content-Type:application/json" localhost:9090//v1/resetOffsetOfAllPartitionsToEarliest/?pipelineName=orion_transaction  

   **Response:**

    {  
      "Orion.Transaction.info": {  
        "0": 873340,  
        "1": 884664,  
        "2": 879160,  
        "3": 866873,  
        "4": 894115,  
        "5": 880421,  
        "6": 868078,  
        "7": 885777,  
        "8": 876230  
      }  
    }  

**4.reset offset of all partitions to latest:**  
The API reset’s offset of the consumer to the latest offset of each partition.

   **Request:**

    curl -X GET -H "Accept:application/json" -H "Content-Type:application/json" localhost:9090//v1/resetOffsetOfAllPartitionsToLatest/?pipelineName=orion_transaction  

   **Response:**

    {  
      "Orion.Transaction.info": {  
        "0": 898071,  
        "1": 909810,  
        "2": 904827,  
        "3": 892828,  
        "4": 917590,  
        "5": 905389,  
        "6": 893126,  
        "7": 908987,  
        "8": 902492  
      }  
    }  

**5.reset offset of partition:**  
The API reset’s offset of the consumer to the specified offset at partition level. We specify all number of partitions that exist in the topic. This api is useful when we want to want to reset offset to a custom pre-calculated offset for each partition.

   **Request:**
   
    curl -X GET -H "Accept:application/json" -H "Content-Type:application/json" localhost:9090//v1/resetOffsetOfPartitions/?pipelineName=orion_transaction  -d @offset.json  
  
    offset.json file content  
    {  
      "Orion.Transaction.info": {  
        "0": 0,  
        "1": 895788,  
        "2": 890201,  
        "3": 877647,  
        "4": 904420,  
        "5": 891558,  
        "6": 879249,  
        "7": 896014,  
        "8": 887096  
      }  
    }  
  

   **Response:**
   
    {  
      "Orion.Transaction.info": {  
        "0": 0,  
        "1": 895788,  
        "2": 890201,  
        "3": 877647,  
        "4": 904420,  
        "5": 891558,  
        "6": 879249,  
        "7": 896014,  
        "8": 887096  
      }  
    }
