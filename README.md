This project is based on Nathan Sweet's [KryoNet](https://github.com/EsotericSoftware/kryonet)  
[![Build Status](https://travis-ci.org/Edarke/kryonet.svg?branch=master)](https://travis-ci.org/Edarke/kryonet)  

This project is not backwards-compatible with KryoNet, but the following guide expects you to be familiar with it. This project aims to provides higher abstraction over KryoNet with stronger type safety and better performance by reducing redundant serialization of messages.


Key changes from KryoNet
- connection.sendToAll(msg) serializes msg exactly once, rather than once per connection
- Support for caching pre-serialized forms of commonly used messages (see CachedMessage<T>)
- Adds support for 'queries' -- Messages that expect a reply.
- Supports registering callbacks for message types. O(1) dispatch without use of instanceof
- Uses Jackson for json serialization rather than jsonbeans.
- Supports compressed message length prefix. (Saves two bytes/message with default config)
- Requires all message types to implement MessageToServer or MessageToClient
- Uses ConcurrentUnit for unit tests which catch many failures that are ignored in KryoNet
- Removes support for RMI


Examples:

- [Creating a Server and Client](#creating-a-server-and-client)
- [Defining a message Type](#defining-a-message-type)
- [Registering Callbacks](#registering-callbacks)
- [Using Queries](#queries)
- [Pre-serialized Messages](#pre-serialized-messages)




## Creating a Server and Client
Getting started is identical to kryonet if you intend to use the default server/client
```java
	Server server = new Server();
	server.start();
	server.bind(tcpPort, udpPort);
	
	Client client = new Client();
	client.start();
	client.connect(timeOut, "localhost", tcpPort, udpPort);
```



## Defining a message type
- Messages should implement either MessageToServer, MessageToClient, or both.
- All messages inherit an isReliable method that indicates whether they should be sent over TCP or UDP by default. The intent is that you first design your game using TCP for everything, and then optimize later by selecting messages types that can be sent over UDP by just overriding this method.

```java
	/** This message indicates that the client has requested their player to move once in a particular direction.
	* This message is send over TCP by default when using server.send(MessageToSever).
	* If UDP is desired in specific instances, you can still use sever.sendUDP(MessageToServer)*/
    public class MovementMessage implements MessageToSever {
    	
    	public Direction dir;
    	
    	MovementMessage() {
    		// Used by Kryo deserializer
    	}
    	
    	public MovementMessage(Direction d){
    		dir = d;
    	}
    }
```

To define a message type that defaults to UDP:

```java
	/** This message updates the client's knowledge of the player's position. Since this message is sent
	* frequently and the info is quickly outdated, we will send this over UDP by default.
	* the method player.getConnection().sendTCP(MessageToClient) will override this behavior though.
	*/
    public class PositionUpdateMessage implements MessageToClient {
    	
    	public int x, y;
    	
    	PositionUpdateMessage() {
    		// Used by Kryo deserializer
    	}
    	
    	public PositionUpdateMessage(int xx , int yy){
    		x = xx;
    		y = yy;
    	}
    	
    	@Override
    	public boolean isReliable(){
    		return false; // False indicates that messages of this type should be sent over UDP
    	}
    }
```


## Registering Callbacks
RegisteredListeners support mapping Message types to callbacks that can be invoked in constant time. 
Here's an example that demonstrates adding callbacks for messages and queries.
```java
	RegisteredServerListener listener = new RegisteredServerListener();
	
	// Delegate to appropriate handler
	listener.addHandler(MovementMessage.class, (msg, sender) -> movementHandler.handle(msg, sender)); 

	listener.addQueryHandle(LoginQuery.class, (query, connection) -> {
		//Reply should be called once on each query to send back a result
		if(query.username.equals("John Smith") && query.password.equals("1234")) {
			query.reply(LoginStatus.SUCCESS);
		} else {
			query.reply(LoginStatus.FAILURE);            
		}
    });
    
    server.addListener(listener);
```



## Queries
Queries are types of messages that are intended to invoke a reply from the other endpoint. Queries and their results are always sent over TCP. You can define a query that returns a type T by extending QueryToServer<T> or QueryToClient<T>. Additionally timeouts for queries can be specified by overriding the Query#getTimeout() method.


Let's say that you're developing a turn-based strategy game.
In your game client, you probably have code that involves logging into your game server, since every request of this type will require a response, it may be appropriate to create a LoginQuery class that extends QueryToServer<T>.
```java
    public void attemptLogin(String username, String password) {
	    Optional<LoginStatus> response = server.sendAndWait(new LoginQuery(username, password)); // blocks until response recieved.
	    response.ifPresent(reply -> {
		    if(reply == LoginStatus.SUCCESS) {
			loadGame();
		    } else {
			showDialog("Username/Password combination is incorrect.")
		    }
	    }
    }
```


Queries can also be handled asynchronously with futures. Let's say its player1's turn to make a move, and the game server needs to indicate to the client that it's time for the user to select an option. 
```java
    player1.sendAsync(new RequestSelection()).thenAccept(new Consumer<Selection>(){
		@Override
		public void accept(Selection reply){
		   // Handle selection. This code will run in another thread. 
		}
     }); 
    
```

It's very easy to define new query types. For example, the RequestSelect query used above would be defined as:
```java
	public class RequestSelection extends QueryToClient<Selection> { }
```

We can optionally, specify a default timeout for this type of query by overrriding getTimeout()
```java
	public class RequestSelection extends QueryToClient<Selection> {
		@Override
		public Duration getTimeout() {
			return Duration.ofMinutes(5);
		}
	}
```
A timeout can also be specified for individual requests:
```java
    player.sendAsync(new RequestSelection(), Duration.ofSeconds(100))
    	.exceptionally(timeoutError -> getDefaultSelection())
    	.thenAccept(new Consumer<Selection>(){
		@Override
		public void accept(Selection reply){
		   // Handle selection. This code will run in another thread. 
		}); 
    
```

## Pre-serialized Messages
Identical messages that are sent frequently can be serialized once ahead-of-time and sent more efficiently later. A quick benchmark suggests that pre-serialized messages can be sent 10x faster for simple objects.  

```java
	CachedMessageFactory msgFactory = server.getCachedMessageFactory(); 
	MyMessage msg = new MyMessage(); 	// Message that will be cached.
	CachedMessage<MyMessage> cached = msgFactory.create(msg);
	connection.send(cached);  	// Equivalent of connection.send(msg), but faster.
```
