CSE535 Course Project - Byzantine Chain Replication Implementation [http://www.cs.cornell.edu/~ns672/publications/2012OPODIS.pdf]

Team : 
	Adarsh Kashyap
	Kanishta Agarwal

Professor :
	Scott Stoller

--------
PLATFORM
--------

Distribution Information

Distributor ID :  Ubuntu
Description	:  Ubuntu 16.04.1 LTS
Release  :  16.04

Kernel Version

Linux kanisht 4.4.0-96-generic
#119-Ubuntu SMP x86_64 x86_64 x86_64 GNU/Linux

DistAlgo :
https://github.com/DistAlgo/distalgo

Python implementation & version:
Implementation - 'CPython'
Version - ‘Python 3.5.2’


-----
FILES
-----

	Olympus : ~/Byzantine_chain_replication/src/olympus.da
	Clients : ~/Byzantine_chain_replication/src/client.da
	Replicas: ~/Byzantine_chain_replication/src/replica.da
	Configs : ~/Byzantine_chain_replication/configs
	Logs 	  : ~/Byzantine_chain_replication/logs

-----------------------
INSTRUCTIONS TO EXECUTE
-----------------------

Prerequisites:

Python version 3.5 or higher, which can be obtained from http://www.python.org.
Installation of DistAlgo. https://github.com/DistAlgo/distalgo

steps :

1) 	Download source code using web URL https://github.com/KashyapAdarsh/Byzantine_chain_replication.git or from blackboard.
2) 	compile :
	To compile all the files we have written a script file, you can use it by doing the following :
	
		./compile.sh

	Above should compile all the files without any errors [Might have some warnings] 

3) 	Execute any of the test cases:

   	According to our implementation every testcase is a seperate config file. As a part of our testing
   	we have used several config files and all of them are shared under config folder. Below are the steps
   	to run our code.

   	Navigate to src folder -

   		cd ~/Byzantine_chain_replication/src

   	We are running our code on multiple nodes, so we will have to start main driver program first using -

   		python -m da --message-buffer-size 640000 -n main main.da CONFIG_FILE_PATH
   		[We are using 64kb as message buffer size]

   		e.g. python -m da --message-buffer-size 640000 -n main main.da ../config/result_shuttle_change_result.config

   	Now we need to start three nodes/terminals [For all our configs we have spread our clients and replicas in three nodes,
   	if you are using lesser or more then you might have to edit config files accordingly, see testing.txt for more details]

   		Terminal-1 : 

   			Navigate to src folder :  cd ~/Byzantine_chain_replication/src
   			Start the Node :  python -m da --message-buffer-size 640000 -n Node0 -D main.da

   		Terminal-2 : 

   			Navigate to src folder :  cd ~/Byzantine_chain_replication/src
   			Start the Node :  python -m da --message-buffer-size 640000 -n Node1 -D main.da

   		Terminal-3 : 

   			Navigate to src folder :  cd ~/Byzantine_chain_replication/src
   			Start the Node :  python -m da --message-buffer-size 640000 -n Node2 -D main.da

   	Now once all terminals are started our program will execute the above specified config file and logs
   	will ge generted in log folder as log.txt.


NOTE : If we are running using multiple hosts then we need to do this :

Start olympus : python -m da --message-buffer-size 640000 -n Node_olympus -H 172.24.20.43 main.da

start in other terminals : 

python -m da --message-buffer-size 640000 -n Node0 -H 172.24.20.43 -R 172.24.21.98 -D main.da

python -m da --message-buffer-size 640000 -n Node1 -H 172.24.20.43 -R 172.24.21.98 -D main.da

python -m da --message-buffer-size 640000 -n Node2 -H 172.24.20.43 -R 172.24.21.98 -D main.da

-H : host IP
-R : olympus IP

We can show this during demo.

-------------------
WORKLOAD GENERATION
-------------------

Client workload is provided to the clients as part of initial configuration by olympus. 
Client workload may be a list of specific requests or pseudo random generated workload.

Pseudorandom client workload generation :

Pseudorandom workload is specified in configuration using the syntax pseudorandom(seed, n), 
which denotes a sequence containing n pseudorandom requests, with the specified seed for the 
pseudorandom number generator. On client initialization in main, generate_random_workload 
method is invoked with arguments as (seed, n) for clients for whom pseudo random workload 
needs to be generated. In generate_random_workload 'random.seed(seed)' gives a value to random
value generator ('random.randint()') which generates these values on the basis of this seed. 
Here once we put same seed we get the same pattern of random numbers. To get pseudo random 
operations we loop ‘n’ times and in each loop we generate ((int)operation_type, 
(int)offset_operation_type) tuple and append it to operations list.‘Operation_type’ corresponds
to one of the fours valid operation type. We obtain this by doing modulus of randomly generated 
with 4(total operation type). ‘Offset_operation_type’ corresponds to the random index in list of
operations for that particular operation.Further ‘Populate_workload_operations’ is called which 
uses the operation list and generates the pseudorandom workload list using operation_type and 
offset_operation_type information from each tuple.

---------------
RECONFIGURATION
---------------

As a part of phase 3 we have implemented reconfiguration, here the main idea is :
1) Wedge request is sent by olympus
2) All replicas become immutable and send their wedged data
3) As soon as replicas receives t+1 wedged response, it makes its first quorum and starts quorum check.
4) In the background more combination of quorums are made.
5) Checkpoint, history and slots consistency is checked from the wedged response of replicas in the quorum,
   if any of these check fails then we take next quorum
6) Longest history is identified if any.
7) catch up message is sent to all replicas which are lacking in the history
8) Replicas will catchup and also send a <client, latest result> data to olympus along with hash of datastore
9) Catch up's are verified.
10) Pending results are sent to client.
11) Running state is requested from all replicas in quorum.
12) Running state is verified.
13) all replicas are killed and new replicas are spawned.
14) required setup for replicas are performed (key generation, sharing info with clients, giving failure config info)
15) Once client realizes config is changed, it will send any pending msg or new msg to new head.


----------------------------
PERIODIC CONFIG CHANGE CHECK
----------------------------

This logic is a part of client where client periofically checks if replica config is changed in olympus.
We use a background thread to do this, we have a call back function which will be invoked at a particular
interval and in this method we exchange messsages with client to check if configuration is updated. If configuration
is updated then first get latest confi information and we send any pending/new requests to new head.

-------------
CHECKPOINTING
-------------
We have implemented checkpoints based on the slot count that head receives, at every interval of say 5 slots we 
invoke checkpointing where head will start a new shuttle with the slot details and also hash of datastore. When 
this checkpoint shuttle travels up the chain all replicas will verify it and also save the checkpoint details
which will have to be sent to olympus on request.

--------------------
BUGS AND LIMITATIONS
--------------------

These are certain bugs and limitations that we have for the current implementation.

1)	Our code does not generate seperate log files for seperate configurations, it overwrites old log files if any.
2)	Our implementation requires that config file has node details for replica and clients seperated by ';'
3) There might be some issue because of message timing, all the configurations provided have been tested thoroughly
   sometimes the behaviour is slightly different because of message delays.
4) Few combinations of triggers and failures did not make sense to use and hence our system design does not support 
   them e.g. using checkpoint(n) with change_operation/change_result.
5) Few operations such as catchup, new config, runningstate works with n=1, this has been clarified by professor as 
   well

-------------
CONTRIBUTIONS
-------------

We together developed the design for the different modules and entities used in the project.
We then split the implementation of the modules amongst us.

Combined effort on the following:

-	Overall design of the product
-	Supported features
-	Interface between different components
-	Making config files for testing.
-	Overall testing and debugging
-	classes.da

Adarsh implemented the following features:

-	Replica complete Implementation
-	Olympus complete Implementation
-	Part of main driver code.
-	Few triggers and failure injection
-	Logging implementation
-  Checkpointing
-  Reconfiguration

Kanishta implemented the following features:

-	Client complete implementation
-	Parsing and evaluating the configurations
-	Pseudorandom workload generation
-	Client data verification at the end of testcase
-	Few trigger and failure injection
-  Re-implemented logic to read the intial conig file
-  Implemented all triggers and failures required for phase3

---------
CODE SIZE
---------

Numbers of non-blank non-comment lines of code (LOC):

	Algorithm : 963
	Other : 650
	Total : 1613

Used CLOC https://github.com/AlDanial/cloc

For Algorithm loc we have - 350 core algorithm related lines, 190 lines related to interleaved functionality.

----------------------
LANGUAGE FEATURE USAGE
----------------------

list comprehensions - 15
dictionary comprehensions - 22

Distalgo features :

1) send 	- To send message between nodes
2) receive	- receive handlers to receive the messages
3) await	- Awaits to wait for necessary condition
4)	timeout	- To check if a particular operation timedout
5)	setup	- To setup an instance of any class

--------------
OTHER COMMENTS
--------------

Our implementation is mostly inline with our pseudocode design. We might have changes few minor things
like using a different or addition data structors, extra receivers, helper and util functions.

Thank you
