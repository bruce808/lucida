# Lucida

Lucida is a speech and vision based intelligent personal assistant inspired by
[Sirius](http://sirius.clarity-lab.org). Visit the provided readmes in
[lucida](lucida) for instructions to build Lucida and follow the instructions to
build [lucida-suite here](http://sirius.clarity-lab.org/sirius-suite/).  Post to
[Lucida-users](http://groups.google.com/forum/#!forum/sirius-users) for more
information and answers to questions. The project is released under [BSD
license](LICENSE), except certain submodules contain their own specific
licensing information. We would love to have your help on improving Lucida, and
see [CONTRIBUTING](CONTRIBUTING.md) for more details.

## Overview

- `lucida`: back-end services and command center (CMD). 
Currently, there are 4 categories of back-end services:
speech recognition (ASR), image matching (IMM), question answering (QA),
and calendar (CA). By default, Lucida uses the following ports:
3000, 8080 for CMD; 8888 for ASR (web socket listener as part of CMD) ; 
8082 for IMM; 8083 for QA; 8084 for CA.

- `tools`: dependencies necessary for compiling Lucida.
Due to the fact that services share some common dependencies like Thrift,
all services should be compiled after these dependencies are installed.

## Lucida Local Development

- From this directory, type: `make local`. This will run scripts in `tools/` to
  install all the required depedencies. After that, it compiles back-end services
  in `lucida/`. Note: if you would like to install the
  packages locally, each install script must be modified accordingly. This will
  also build `lucida-suite` and `lucida`.
- Similar to what is set in the Makefile, you must set a few environment
  variables. From the top directory:
```
export LD_LIBRARY_PATH=/usr/local/lib
export LUCIDAROOT=`pwd`/lucida
```
- Start all the services:
```
make start_all
```
- To test, open your browser, and go to `http://localhost:3000/`.

## Lucida Docker Deployment

- Install Docker: refer to
  [https://docs.docker.com/engine/installation/](https://docs.docker.com/engine/installation/)
- Install Docker Compose: use `pip install docker-compose` or refer to
  [https://docs.docker.com/compose/install/](https://docs.docker.com/compose/install/)
- Pull the Lucida image. There are several available:  
`docker pull claritylab/lucida:latest # add your own facts` (TODO!!!!!)
- Pull the speech recognition image (based on
  [kaldi-gstreamer-server](https://github.com/alumae/kaldi-gstreamer-server)):  
`docker pull claritylab/lucida-asr`
- From the top directory of Lucida:  
`docker-compose up`
- In Chrome, navigate to `localhost:8081`

Note: Instructions to download and build Sirius can be found at
[http://sirius.clarity-lab.org](http://sirius.clarity-lab.org)

## Design Notes -- How to Add Your Own Service into Lucida?

### Start with Thrift

Thrift is an RPC framework with the advantages of being efficient and language-neutral. 
It was originally developed by Facebook and now developed by both the open-source community (Apache Thrift) and Facebook.
We use both Apache Thrift and Facebook Thrift because Facebook Thrift has a fully asynchronous C++ server but does not support
Java very well. Also, Apache Thrift seems to be more popular.
Therefore, we use Apache Thrift for services written in Python and Java,
and Facebook Thrift for services written in C++.

One disadvantage about Thrift is that the interface has to be pre-defined and implemented by each service. 
If the interface changes, all services have to re-implement the interface. 
We try to avoid changing the interface by careful design, but if you really need to adapt the interface for your need,
feel free to modify, but make sure that all services implement and use the new interface.

### Does it mean I only need to implement the Thrift interface?

The short answer is no, but you only need to configure the command center (CMD) besides implementing the Thrift interface
in order to add your own service into Lucida. Let's break it down into two steps:

1. Implement the Thrift interface jointly defined in `lucida/lucidaservice.thrift` and `lucida/lucidatypes.thrift`.

 1. `lucida/lucidaservice.thrift`:

    ```
    include "lucidatypes.thrift"
    service LucidaService {
       void create(1:string LUCID, 2:lucidatypes.QuerySpec spec);
       void learn(1:string LUCID, 2:lucidatypes.QuerySpec knowledge);
       string infer(1:string LUCID, 2:lucidatypes.QuerySpec query);
    }
    ```
    
    The basic funtionalities that your service needs to provide are called `create`, `learn`, and `infer`. 
    They all take in the same type of parameters, a `string` representing the Lucida user ID (`LUCID`),
    and a custom type called `QuerySpec` defined in `lucida/lucidatypes.thrift`. 
    The command center invokes these three procedures implemented by your service,
    and services can also invoke these procedures on each other to achieve communication.
    Thus the typical data flow looks like this:
    
    ```Command Center (CMD) -> Your Own Service (YOS)```
    
    But it also can be like this:
    
    ```Command Center (CMD) -> Your Own Service 1 (YOS1) -> Your Own Service 2 (YOS2) -> Your Own Service 3 (YOS3)```
    
    In this scenario, the command center sends a request to YOS1, YOS1 processes the query
    and sends the reqeust to YOS2. 
    
    Make sure to implement the asynchronous Thrift interface.
    If YOS1 implements the asynchronous Thrift interface, which it should,
    it won't block on waiting for the response from YOS2. Rather, a callback function is registered
    and the current thread can resume execution, so that when the reponse from YOS2 gets back to YOS1, the callback
    function is executed, in which YOS1 can either further process the response or immediately return it to CMD.
    If YOS1 implements the synchronous Thrift interface, the current thread cannot continue execution until
    YOS2 returns the response, so the operating system will suspend the current thread and perform a thread context switch
    which incurs overhead. The current thread sleeps until YOS2 returns. 
    Hopefully you see why we prefer asynchronous implementation of the thrift interface to synchronous implementation.
    See (3) of step 1 for details.
    
    `create`: create an intelligent instance based on supplied LUCID
    
    `learn`: tell the intelligent instance to learn based on data supplied in the query. 
    Although it must be implemented, you can choose to do nothing in the function body or simply print some message
    if your service cannot learn new knowledge. For example, it may be hard to retrain a DNN model, so the facial recognition
    service simply print a message when it receives a learn request. You can tell the command center not to send a learn request
    to your service, which will be explained soon.
    If your service can handle new knowledge in the form of either image or text, you should tell the command center, which will be explained soon.
    
    `infer`: ask the intelligence to infer using the data supplied in the query.
    This is the most important functionality in the sense that it receives a query in the form of either image or text and
    returns the response in the form of a string. As will be explained soon, a string can be either plain text or image data,
    but usually human readable plain text is returned and this is what is assumed in the command center.

 2.  `lucida/lucidatypes.thrift`:

    ```
    struct QueryInput {
        1: string type;
        2: list<string> data;
        3: list<string> tags;
    }
    struct QuerySpec {
        1: string name;
        2: list<QueryInput> content;
    }
    ```
    
    A `QuerySpec` has a name, which is `create` for `create`, `knowledge` for `learn`, and `query` for `infer`. 
    A `QuerySpec` also has a list of `QueryInput` which is the data payload. 
    A `QueryInput` consists of a `type`, a list of `data`, and a list of `tags`. 
    
    * If the function call is `learn`:
    
    Only one `QueryInput` is sent to your service currently, but you shouldn't assume this. Instead,
    you should iterate through all `QueryInput`s and grab all data to learn.
    The `type` can be `text` for plain text, `url` for url address, `image` for image,
    or `unlearn` (undo learn) for the reverse process of learn.
    A service can handle either text or image, and if it can handle text, the type will be
    `text`, `url`, or `unlearn`, 
    and if it can handle image, the type will be `image` or `unlearn`.
    The command center guarantees that a service that can only receive text won't receive image
    as long as the command center is configured correctly.
    See step 2 for more details about this and how to define your own types (e.g. `video`).
    If `type` is `text` or `url`, `data[i]` is the `i`th piece of text as new knowledge
    and `tags[i]` is the id of the `i`th piece of text generated by a hash function in the command center;
    if `type` is `image`, `data[i]` is the `i`th image as new knowledge
    (notice that it is the actual string representation of an image and thus can very long),
    and `tags[i]` is the label/name of the `i`th image received from the front end;
    if `type` is `unlearn`, `data` should be ignored by your service (usually a list of an empty string),
    and `tags[i]` is the id of the text to delete or the label of the image to delete
    based on whether the service can handle text or image.
    See step 2 for more details on how to specify the type of knowledge that your service can handle.
    
    * If the function call is `infer`:
    
    Each `QueryInput` in `content` corresponds to one service (CMD is not considered to be a service)
    in the service graph, a connected directed acyclic graph (DAG) describing all services that are needed for the query.
    Thus, for the following service graph, only one `QueryInput` is generated by the command center`:
    
    ```Command Center (CMD) -> Your Own Service (YOS)```
    
    The `type` can be `text` for plain text, or `image` for image. 
    Similar to `learn`, if `type` is `text` or `url`, `data[i]` is the `i`th piece of text as query;
    if `type` is `image`, `data[i]` is the `i`th image as query.
    However, `tags` have the following format:
    
    ```
    [host, port, <size of the following list>, <list of integers>]
    ```
    
    , which indicates a node in the service graph. The `host:port` specifies the location of the service,
    and second list specifies the indices of nodes that the service points to.
    The size matches the length of the list, and thus must be a non-negative integer (0 indicating an empty list).
    If the list is empty, the node does not send any further request.
    
    Therefore, the above service graph results in a `QuerySpec` like this:
    
    ```
    { name: "query", 
    content: [ 
    { type: "text",
    data: [ "What's the speed of light?" ],
    tags: ["localhost", "8083", "0"] }
    ] }
    ```
    
    . We can define arbitrarily complicated service graphs. For example, for the following service graph:
    
    ![Alt text](Service_Graph.png?raw=true "Service Graph")
    
    , the resulting `QuerySpec` may look like this, assuming YOSX is running at 909X:
    
    ```
    { name: "query", 
    content: [
    { type: "image",
    data: [ "1234567...abcdefg" ],
    tags: ["localhost", "9090", "1", 2"] },
    { type: "image",
    data: [ "1234567...abcdefg" ],
    tags: ["localhost", "9091", "2", "2", "3"] },
    { type: "text",
    data: [ "Which person in my family is in this image?" ],
    tags: ["localhost", "9092", "0"] },
    { type: "image",
    data: [ "1234567...abcdefg" ],
    tags: ["localhost", "9093", "0"] }
    ] }
    ```
    
    . Notice that if the order of `QueryInput` in `content` is rearranged, the `QuerySpec` still corresponds to the same graph.
    In fact, there are `2^(N)` valid `QuerySpec`s for a given graph, and you need to define only one of them in
    the configuration file of the command center. Notice that the starting nodes, YOS0 and YOS1, need to be specified separately,
    so that the command center knows where to send the request to.
    See more detais in step 2.
    
    The command center guarantees to send a valid `QuerySpec` as long as the user defines the service graph correctly,
    but your service is responsible for parsing the graph, sending the request(s) to the service(s) it points to,
    and returning the response(s) to the service(s) it is pointed to by.
    Suppose in the above example, YOS0 does not send the request to YOS2,
    and YOS2 is written in a way that it must process requests from both YOS0 and YOS1 before returning to YOS1.
    Then YOS2 cannot make progress, which leads to YOS1 waiting for YOS2, which leads to the command center waiting for YOS1.
    Each service is also allowed to ignore or modify the graph if that is necessary, but that should be done with caution.
    
    Although the service graph can be very complicated, it is usually very simple. At least for the current Lucida services,
    the most complicated graph looks like this:
    
    ```Command Center (CMD) -> IMM -> QA```
    
    . Thus, most current services can ignore the `tags` without any problem.

 3. Here are the concrete code examples that you can use for your own service:

    If it is written in C++, refer to the code in `lucida/lucida/imagematching/opencv_imm/server/`, especially `IMMHandler.h`,
    `IMMHandler.cpp`, and `IMMServer.cpp`.
    
    If it is written in Java, refer to the code in `lucida/lucida/calendar/src/main/java/calendar/`, especially `CAServiceHandler.java` and `CalendarDaemon.java`.
    
    Here are what you need to do concretely: 
    
    * Add a thrift wrapper which typically consists of a Thrift handler
      which implements the Thrift interface described above, and a server daemon which is the entry point of your service.
    
    * Modify your Makefile for compiling your service and shell script for starting your service.
    
    * Test your service.
    
    * Put your service into a Docker image, and add Kubernetes `yaml` scripts for your service.

2. Modify the command center. 

 [This is the only file you need to modify.](lucida/commandcenter/controllers/Config.py)
 
 ```
 MAX_DOC_NUM_PER_USER = 30 # maximum number of texts or images per user

TRAIN_OR_LOAD = 'train' # either 'train' or 'load'

# Pre-configured services.
# The ThriftClient assumes that the following services are running.
# Host IP addresses are resolved dynamically: 
# either set by Kubernetes or localhost.
SERVICES = { 
	'IMM' : Service('IMM', 8082, 'image', ['image']), 
	'QA' : Service('QA', 8083, 'text', ['text']),
	'CA' : Service('CA', 8084, 'text', None),
	'IMC' : Service('IMC', 8085, 'image', None),
	'FACE' : Service('FACE', 8086, 'image', None),
	'DIG' : Service('DIG', 8087, 'image', None),
	'ENSEMBLE' : Service('ENSEMBLE', 9090, 'text', None) 
	}

# Map from input type to query classes and services needed by each class.
CLASSIFIER_DESCRIPTIONS = { 
	'text' : { 'class_QA' :  Graph([Node('QA')]) ,
			   'class_CA' : Graph([Node('CA')]) },
	'image' : { 'class_IMM' : Graph([Node('IMM')]),
				'class_IMC' : Graph([Node('IMC')]),
				'class_FACE' : Graph([Node('FACE')]),
				'class_DIG' : Graph([Node('DIG')]) },
	'text_image' : { 'class_QA': Graph([Node('QA')]),
					 'class_IMM' : Graph([Node('IMM')]), 
					 'class_IMM_QA' : Graph([Node('IMM', [1]), Node('QA')]),
					 'class_IMC' : Graph([Node('IMC')]),
					 'class_FACE' : Graph([Node('FACE')]),
					 'class_DIG' : Graph([Node('DIG')]) } }
 ```







