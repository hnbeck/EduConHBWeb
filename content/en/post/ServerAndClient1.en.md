+++
date = "2020-01-16T18:00:00+01:00"
publishdate ="2020-01-19"
highlight = true
math = false
tags = ["Programming"]
title = "Server - Client - Replication"

[header]
  caption = "(c) Hans N. Beck"
  image = "flipper.png"

+++

### Motivation

Designing web applications leads always to the question where the data should be located. Because of common practice we have some paradigms in place narrowing our view what can be. We had ages of thin clients where the server owns all data in most applications. Then there was a time when developers put more data on the client. In addition, the speed of network connections and JavaScript influenced the common idea of how a web application has to look like. 

Beside this mainstream there was the idea to build up a computer network with nearly symmetric nodes. The OpenCroquet project for example was build around this idea. See [OpenCobalt](http://www.opencobalt.net/) as one of the successors or please refer to the actual project [Croquet.studio](https://croquet.studio//). 

For my PROLOG based games I think about variants to store the main game data. I always liked the OpenCroquet idea of replicated computing. But it is important to use existing libraries and frameworks in order to limit the amount of work. The power of PROLOG to handle data as code puts an additional light to the scene. Here I wrote down some thoughts about this topic. Before I'll start the comparison of options I want to look at replication, what it is and what it could be in the PROLOG world.


### What Replication is

Let's look at the basic idea of replication as described in the [PHD thesis of David P. Reed](http://publications.csail.mit.edu/lcs/pubs/pdf/MIT-LCS-TR-205.pdf) of 1978. This basic idea is as follows. A system is understand as a collection of objects receiving messages. Any object is defined by its history of received and executed messages since creation. If a message want to access an selected object of a selected version it has to name the object and a time.  These messages can be received from every where in the network.  

Now the big thing of David Reeds work is to ensure that this history is well defined for all objects. Naming all possible states (objects and their points in history) is not trivial. Replicated objects in this approach are that of same type and history. If objects of a type are created at different nodes in the network then these objects can be kept in sync by just ensure equal history on every object. In consequence, not data has to be send over the network but messages with their reference to a object of a selected time. For more details of an implementation - the OpenCroquet system - please refer to [this presentation from David Reed](https://dl.acm.org/doi/10.1145/1094855.1094861)


### What this means for PROLOG

For the discussion please refer to the architecture below which all my PROLOG games will base on. 

{{< figure src="/src/PrologGamesBasic.png" class="myimg" title="Base architecture" >}}

The PROLOG part implements the game rules. My thinking is about structures and effects. Structure can be levels, obstacles, game objects and are coded as predicates. When I say *"structure"*, I refer to the logic structure of the (model) world which can defined in predicates of mathematical logic. 

Rules describe the transformation of structure: how objects move on or what happens if 2 objects will be combined - for a collision for example. Every structure element is also source of effects as light, a field, a push or what ever. The effect part is what graphics and interactivity are realized in. JavaScript or libraries based on JavaScript encode this dynamic aspects of the world. 

Back to the replication which is described in objects and messages. PROLOG has no objects and messages. It is built around facts, rules and queries. So how to do the mapping of pseudo time to this? One thing is that different versions of the knowlegde base in PROLOG could be associated to the history of queries applied. In the procedural view of PROLOG an application of a rule is like a method call. Introducing a pseudo time as reference to a query history may be possible. But that is not the only thing that the query has to happen in a certain sequence. The facts which are the parameter of the queries should be of a selected version or time point. So maybe every predicate gets an new argument: the pseudo time reference.

 What about the objects? Assume we have space ships in a game as well as asteroids. It seems natural to implement ships and asteroids as object. What would be the equivalent in PROLOG? Knowledge is encoded and stored in predicates. Facts about the ships and asterodis are given as predicates. A message send to an object in the OpenCroquet world I would interpret as a query on predicates with pseudo time. This may require that the rules for transforming the related structures also have to be stored in connection to this predicates. [Logtalk](https://logtalk.org) has a much more envolved view on this point, maybe I'll switch to it for later work.


### And now - the comparison

My approach for finding a judgement about the different Client-Server-Replication options is to examine the questions below. In order to state the questions precise let me define this basic operations needed in the PROLOG setting:

S 	==  Structure, a subset or the full game  
S\* == Structure of a next pseudo time  
A 	== Answer to a query 
t 	== Real time  
R  	== Rules  
F 	== Facts as result of effects
n 	== Amount or number of  

Then we have this abstract functions:

| Command              | Description              | Example |
| ---------------------|--------------------------|--------|
| ?(S, A) | Query about the structure, returning A | Are there spaceships on the playfield? A would then contain all the ships with this predicate |
| !(S, S\*) | Applying all applicable rules R on S, resulting in S\*| This would be one tick of the pseudo time clock. For example take the rule that on every place only ship can be. If in the structure S 2 ships will be on the same place that must be a collision |
| +(S, F) | Calculate all effects out from structure S, resulting in new facts F | A spaceship has velocity v. The effect would be that the ship has moved after on tick of pseudotime according its v. The new fact is the new position |
| >(F, S, S\*) | Translate facts which are result of effects into new structure S\*| Look at the ship example above. Because S is a logical structure it may be or not that the new ship positions change something in the logical structure. If the new position induces a collision, then this will be a structural thing. If one ship free in space moves a little bit it will be still a free ship in space.|
| O(X) | Evaluate complexity related to amount of storage or time of X) | O(S) may count the predicates stored in S |


Keeping the interpretation of objects and messages and the operations described above in mind, I'll look at:

1. Data load (the amount of data) on a node. 
2. Communication load between nodes
3. Implementation effort of the effect (graphics, UI) on a node

For the PROLOG world and the definitions above these questions translate to

1. Storage and processing load of the structure S and the rules R
2. Communication load between nodes 
3. Processing load for determine F and S\* out from S


### Structure on the server

**Question 1:**

?(S, A) is implemented on the server. S contains the complete game. In consequence - because the complete set of S is on the server - !(S, S\*) is also part of the server. O(S), O\(R\) are big for big games. 

S has to be implemented in PROLOG way as predicates, facts and rules. There are many ways to make data in PROLOG persistent. The developer can choose between many kinds of databases. With the *asserta* predicate any term could be add to the - persistent - knowledge database. The structure describes the game world and all game objects. 

**Question 2:**

?(), S have to be send to the server. O(?) and O(A) << O(S) and O\(R\). The pengine library of SWI Prolog sends queries as strings, so ?(), A has to be stringified. The answer A is sent back as JSON objects, which matches perfect for the JavaScript environment. Big worlds and many effects require a big A, so the traffic will increase with the structural complexity of the game.

**Question 3:**

 The client has to implement +(S, F).  O(+) depends on the rules an mechanisms describing the effect. S is received by the server. Related to the function >(F, S, S\*) there are 2 possibilites: if >() is executed on the client then the client has to need rules how F transforms to elements of S. That may require edditional communication and storage on client side. For a lightwight client it would be better to set >() on the server. In this case the server has to now also something about F, it cannot described in terms of S alone. The best option may depend on the game structure and the complexity of >()


### Structure on the client

**Question 1:**

The server only holds R and performs !(S, S\*). In general O\(R\)<<O(S) holds saying that the storage requirements will be low.

**Question 2:**

Because the rules on the server performing !(S, S\*) they have to receive all necessary elements of S from the network. In general this could be a lot - in worst case all of the full set S. The server performs !() and sends S\* back, which could also many data. So in general O(S), O(A) with respect to communication would be big.

**Question 3:** 

?(S, A), +(S, F) and >(F, S, S\*) are done on the client. The advantage may be that the encoding of S and F could be as needed and usefule for the client in its JavaScript world. The downside is that O(?)+O(+)+O(>) >> O(!). So the in fact, the server is nearly useless, all load is on client and network.


### Replication

**Question 1:**

Two variants are possible:

1. The SWI Prolog server is a node as all other. According to the understanding of replication here ?(S, A) and !(S, S\*) are implemented on every node. Every node has its replicated R, S and A and can encode and process S and A in the best way suited to the environment: SWI Prolog, Tau-Prolog or JavaScript (or type systems). In general, every node needs also the effect functions +(S, F) and >(F, S, S\*). A server needs no effects and GUI, so this functions may not be necessary.

2. The SWI Prolog server is a pure rules server. It provide the rule pattern !() without application of rules,  !(S, S\*) has to be executed on the client.  But I have to admit that I have no idea what this would looks like, what are the tasks of such a gateway. Therefore I'll take version 1. as initial approach. 

**Question 2:** 

Because every node has R, S and A, only the querys have to be transmitted. In opposite to the protocol the Pengine library of SWI Prolog uses replication would allow to transmit only references to the pseudo time and the predicates ("objects"). Including A and a subset of S would not be required resulting in a very fast and light communication. 

**Question 3:**

As a full node, the "client" implements ?(S, A), !(S, S\*), +(S, F) and >(F, S, S\*). In principle coding could be in JavaScript as well as in Tau-Prolog. The nature of replication allows to keep the encoding of the package  S, R, F JavaScript friendly. 

Again here are 2 variants possible: 

1. Queries and rules are subject of replication. In this case the interpretation of replication described above will hold. Pseudo times and naming of predicate versions has to be implemented in some way in PROLOG. Some code will required in SWI Prolog and some in Tau-Prolog.

2. Only events and effects will be replicated. In this setting it may be reasonable to use the  [Croquet.studio](https://croquet.studio//) SDK (which is JS) to implement the this kind of replication. The SDK is easy to use. Because it is JavaScript it should be integrable with the graphics JavaScript stuff realizing all graphics and interactions.


### Summary

Replication and PROLOG looks interesting and I will investigate further in this option for "client-sever" applications.  There are many questions to solve and also a bunge of things to implement. But nevertheless for designing multi-user games which utilitize PROLOG as a rule engine this all looks attractive to me. Some code experiments will follow, and I will report about the results.


 