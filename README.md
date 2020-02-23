# tonyjoblin.github.io

## Using dynamodb local with AWS SAM

23 February 2020

Setup the initial SAM app using the SAM cli

```
sam init
```

When I did this I used the nodejs10.x quick start and selected the web backend. This gives you an app with 3 lambda functions
* getAllItemsFunction
* getByIdFunction
* putItemFunction

and a single dynamodb table
* SampleTable

I am also assuming you are running on some brand of Linux.

### Connecting dynamo local to sam local api

In order to get the lambdas to be able to talk to the local dynamodb, we need to put them both in the same docker network. Then update the endpoint used by the dynamodb client in the lambdas. The minimal steps required to do this are

1. create docker network for your app
    ```
    docker network create sam-demo
    ```
2. Run the dynamodb local docker container. You need to run it in your app network using the ```--network``` option. Also give it a useful name like ***dynamo***, this name will be used for networking to it from the lambdas.
    ```
    docker run -d --name dynamo \
        -p 8000:8000 --network sam-demo \
        amazon/dynamodb-local
    ```
    Then you can create your table in dynamo
    ```
    aws dynamodb create-table \
        --table-name SampleTable \
        --attribute-definitions \
            AttributeName=id,AttributeType=S \
        --key-schema AttributeName=id,KeyType=HASH \
        --provisioned-throughput ReadCapacityUnits=1,WriteCapacityUnits=1 \
        --endpoint-url http://localhost:8000
    ```
3. Update the document client to use a local endpoint, eg
    ```
    const options = { endpoint: "http://dynamo:8000" };
    const docClient = new dynamodb.DocumentClient(options);
    ```
    ***Note***: if you are running on Mac or Windows then I think the endpoint url will be slightly different here.

4. When you start your api locally or invoke one of the lambdas locally you need to specify the docker network using the ```--docker-network``` option.
    ```
    sam local invoke getAllItemsFunction \
        -e events/event-get-all-items.json \
        --docker-network sam-demo
    ```

The key points here are putting the dynamodb local and the sam lambdas or api on the same docker network. Outside docker you can connect to dynamo using 127.0.0.1:8000 or localhost:8000, but within docker space you need to refer to it using the name you gave it, eg ***dynamo***.
