# Using Akka with F# in (almost) production
------------

_Go [back](index)_

__THIS IS CURRENTLY WORK IN PROGRESS__

### Preamble

_Skip preamble and go to the [interesting part](#app-service-implemented-in-akkanet-and-f-development-diary)_.

I work at __[Precast Software Engineering](http://www.precast-software.com)__. We develop two products for the precast concrete industry, mostly for the factories themselves.


#### PLANBAR
Planbar is a CAD specialized for precast. It is mostly developed in C++, but like many older applications out there it has amassed a large number of languages and technologies. C transpiled from FORTRAN, actual C, C++ used like C, actual C++, C#, MFC, Winforms, WPF, ... You can see its history in its different layers, beautiful like the rings in an old tree stump.


#### TIM
The Technical Information Manager is a SQL Server based work-preparation program.
It is mostly written in C#, with small pieces of C++, and lately some small new pieces of F#.  
I work mostly on TIM.


#### Status Quo
TIM is a client program that stores the data in the database. It is (supposedly) multi-user capable; that is, multiple users can connect to the same database and work in parallel.

So that multiple users don't trample on each other, a user needs to lock the data he actually works with. These locks are on a project level, and acquired when you start editing. They are released when you unload the project, or quit the application.


##### Issues

1) The locks are saved in the database, causing additional churn.  
2) The locks are saved together with other data in a common table, making transactions a headache. If you lock something inside a transaction, the lock is not visible outside.  
3) If the application crashes, the locks stay in the database. If the same user reconnects, his locks are re-acquired and he can release them.  
4) If you go on vacation, but kept the program open, the locks are held until you return (or someone purges them from the database manually).  
5) The projects are locked way too long, anyway. Effectively, only one user can work with one project at a time. Which is not actually multi-user.


##### Solution

Currently, each client directly connects to the database. We need something in-between, which can handle such mundane tasks as locking and user-management (which was also in the DB. Fun fact: Since there was only TIM and the DB, the clients "authenticated" themselves).

So we need an App-Service. There is one database, one App-Service, and many TIMs.

What the service should do was relatively clear, so the question was which technology to use.

[[todo: maybe elaborate more on the new locking, and user session handling]]

1) WCF  
2) REST over http, with JSON serialized data  
3) .Net Remoting  
4) MessageQueue, for example RabbitMq  
5) Actor, Akka.net  


Requirements
 * RPC style communication over a network between .Net processes.
 * smallish number of clients (n < 100; most of the time < 10 in actual use)
 * easy setup, each customer deploys his own instances on his own servers (and they are paranoid, no Azure)
 * small number of ops/sec (probably less than 10ops/sec/client)
 
Non-Requirements:
 * interoperability with external systems
 * version tolerance


So why choose Akka.net and not ...

1) WCF

We already use WCF to provide Webservices that external systems can call. So we had some knowledge about it.

WCF for internal communication aint actually that bad. You define your Service Contracts (interfaces), your Data Contracts (POCOs) and WCF handles the serialization / autogenerates the proxies.

But WCF is _really_ complex. You need to configure it with an arcane syntax (Bindings?) in xml in your app.config. If something goes wrong, good luck.

Also, __the future of WCF does not seem that clear.__ Microsoft focuses on REST; they still provide a client library for UWP, but I haven't heard any news about WCF Servers.

2) REST

REST is (was? is there already something hotter? I heard a lot of buzz about GraphQL lately) the current trend.  
REST is JSON, which is hip. WCF is Xml, which is uncool.  
But REST looks relatively manual - no autogenerated proxies, instead you go back and manually call string-concatenated uris with hand-serialized json strings and then try to make sense of the response.  
Sure there are libs to take away some of the pain points, but __I don't think REST fits our needs__ - we don't need WebScale, we just need something for .Net <-> .Net inter-network communication.

3) .Net Remoting

Dead. Interestingly, it would probably have been the best fit.

4) MessageQueue

To be honest, I didn't actually do much research on this technology. I would be interested in usage-stories.

5) Akka.net

Actors are a cool concept. They take care of threading issues and enable easy concurrency. It seemed easy to transform the actor model (mailboxes and messages) into a RPC framework.
The biggest upside was that Actors are inherently peer-to-peer. The locking stuff is client-server, where all requests are initiated by the client, but we already had ideas where two-way communication would be nice to have or required. 

I had a little bit of experience with Actors, from when I played around with MBrace (which I would like to also write a small post about) and [Thespian](https://github.com/mbraceproject/Thespian). I would have used Thespian again, but the Nessos guys stopped working on it and recommend Akka.net.

Further reasons we decided to use Actors and Akka:

* the actor model seemed useful for other parts of the application
* no explicit serialization, you can (with caveats) simply pass complete object graphs. Of couse we don't just blindly pass around our domain objects, but not having to manually define a protobuf schema (or having to annotate everything with an attribute) is a blessing.
* [peer-to-peer](#peer-to-peer), location transparency. (With caveats) you don't care where the service is located, and whether it is in-proc or out of proc.
* open source, with companies actually using it.

##### To expand on the open-source part:

If for some reason development of Akka itself stopped, we would not continue using it and would probably migrate off it as fast as possible.

But it being open source was already valuable; the documentation is not always as explicit as you would want, and just being able to look at the implementation helps with understanding.

If you find a trivial issue, you can just send a PR (three got already merged :) ), if it is non-obvious, you can at the least create a public issue and pray someone answers. 

Being in use by a company hopefully means that it is actually usable in the real world.

#### Development Status
We are polishing up the current release and will officially ship it soon™.


# App-Service implemented in Akka.Net and F#: Development Diary

### First Try - nothing works

I coded up an actor and a message type, started two processes and .... - nothing happened.

Long story short, Akka.net uses a Json.net based serializer by default.  
That serializer doesn't handle unions.  
Instead of throwing, it just silently swallowed the messages.  
``¯\_(ツ)_/¯``

### Custom FsPickler based serializer

I'm not the first to notice that, go and read https://github.com/akkadotnet/akka.net/issues/1534 and a few other issues.
So I need a custom serializer? FsPickler it is. Almost.

I didn't want to change the root serializer, but only the serializer for my messages. It would probably have worked, but I didn't want another case of mysteriously disappearing messages.

You can configure your serializer in the HOCON config per message-type. I didn't want to have to explicitly include all my different message types, so I just created a tagging interface and attached that to all my top-level messages. Yes, that means if I create a new message type and forget that interface, the message silently disappers. But it would be boring if there was nothing to improve for the future.

So I just subclassed the Akka Serializer base class, implemented Serialize and Deserialize with FsPickler, and - well it didn't work.

I need to pass around Akka objects (mostly IActorRef) nested inside my types. FsPickler complained that the actor ref is not serializable.  
So time to look up how the default Json serializer handles that.

Short story short, there are two interfaces, ``ISurrogated`` and ``ISurrogate``. At serialization time you call ``ISurrogated.ToSurrogate``, which returns an ``ISurrogate``. This can be trivially serialized. At deserialization time, you do the same dance the other way around with ``FromSurrogate``.

So I created a pickler, which does that, and registered it to ``ISurrogated``. When I then restarted my app, FsPickler complained that the Akka object is not serializable. What?

Turns out, FsPickler only takes concrete types into account when trying to find a pickler. So me registering the interface did nothing. I did the logical thing and just forked FsPickler. Currently I am using a custom built version compiled off my pr https://github.com/mbraceproject/FsPickler/pull/81. This version also takes pickler factories registered for interfaces into account when trying to find a pickler for a type. It works. Would be interesting to see how Akkling handles that.


### Akka and Dispatchers and Deadlocks, great

I may be the first to actually use Akka.net in a GUI application.

C# has __pioneered__ a great feature to make working with concurrency and asynchronous APIs easier - async/await.

Async/Await has a concept of a SynchronizationContext. The main use is for GUI applications, where you can only update the GUI from one Thread, so you need to make sure to go back to that thread after the async call returns.  
The SynchronizationContext handles that for you.  
Unfortunately, async/await automatically captures that context, unless you explicitly opt-out _each time_ with ``.ConfigureAwait(false)``.

Which caused a nice Deadlock. Because Akka internally did not opt out. And so wanted to execute their internal code on my gui thread. Which was waiting for the Actor System.  
``¯\_(ツ)_/¯``

A workaround (move the outer call into the ThreadPool, from where it then calls into Akka) and my first PR against Akka (https://github.com/akkadotnet/akka.net/pull/2550) later, it worked.

There is still an open issue, where Akka schedules continuations on the dedicated IO thread, which means that if you do ``await Ask()``, the code _after_ the await is executed on the Thread that is supposed to push the bits through the network pipe. If you want to deadlock the whole actor system, just do something like

```C#
var x = await actorRef.Ask(...);
MessageBox.Show();
```

That will stop _any_ message from being processed.

``¯\_(ツ)_/¯``

Patient: > Doctor, it hurts if I do that.

Doctor: > Then don't do that.


### Types or GTFO

> <insert something philosophical about types>

If you use F#, you want to use types. Akka is objectly-typed.

This is a basic C# example:


```C#
    public class Greet
    {
        public Greet(string msg)
        {
            Msg = msg;
        }
        public string Msg { get;private set; }
    }

    public class GreetingActor : ReceiveActor
    {
        public GreetingActor()
        {
            Receive<Greet>(greet =>
               Sender.Tell($"Hello {greet.Msg}.", Self);
        }
    }

    class Program
    {
        static void Main(string[] args)
        {
            // Create a new actor system (a container for your actors)
            var system = ActorSystem.Create("MySystem");

            // Create your actor and get a reference to it.
            // This will be an "ActorRef", which is not a
            // reference to the actual actor instance
            // but rather a client or proxy to it.
            IActorRef greeter = system.ActorOf<GreetingActor>("greeter");

            // Send a message to the actor
            string response = await greeter.Ask<string>(new Greet("World"));
            Console.WriteLine($"Response from actor: {response}");
            // This prevents the app from exiting
            // before the async work is done
            Console.ReadLine();
        }
    }
```

Before, my only actor experience was with Thespian. If you don't know it, its API is similar to the F# MailboxProcessor.

You have typed ``ActorRefs<'T>``, and a really beautiful mechanism ``ReplyChannel<'T>``. The first time I saw that concept, it took a little bit to wrap my brain around it, but it is really simple, and statically type safe!

So let's find the issues with the C# api:
How many can you count?

...  
...  
...  


1) Q: What messages can my IActorRef handle? The actor type is just ``ReceiveActor``, the ref is just ``IActorRef``.  
   A: Who knows? Look at the source code of the actor. It can either handle all messages (``obj``), or register for specific ones with ``Receive<T>``. Also, the messages one concrete actor instance handles may change over time.

2) Q: If I receive a message, and want to send something back, how do I know _where_ to send it back?  
   A: That's totally easy! There is a really convenient property ``Sender``, provided by Akka.  
   Q: Great! Wait, how do I know what Messages I can send back?  
   A: ... let's go back to Q1  
   Q: How do I even know whether there is an actor on the other side? What about ``Tell``, which is fire and forget?  
   A: ``¯\_(ツ)_/¯``

3) Q: I have an IActorRef, and want to ``Ask`` it something, how do I know what it will send back, or if it sends something at all?  
   A: take a look at Q2 again.

#### Time to type this shit up

The goal is to create a strongly typed abstraction layer above Akka. This layer should be as thin as possible; hiding Akka itself is an explicit non-goal.

I took a quick look at both the official Akka F# api and [Akkling](https://github.com/Horusiath/Akkling), but my issue at that time was that they both hide too much.
Both are almost complete replacements for the Akka Api, which means the C# tutorial and the C# code samples are basically useless.  
Also, both use operators in place of ``Ask`` and ``Tell``. At first that sounds really great, and _functional_, but in my limited experience, operators are almost never a good idea. Methods can have optional parameters, and can be overloaded. Operators? How do I set a timeout? How do I pass a CancellationToken?

Anyway, I decided to roll my own wrapper. Even now, this whole wrapper has less than 200 LOC.  

Looking back, I still think this was a good decision. This way I could get my feet wet with Akka, without being confused by the differences between the C# Api and Akkling. For a new project, I would probably take a second look at Akkling, because now I am (a little bit) familiar with Akka itself, so it is easier to understand the additional abstractions Akkling offers. If it is a good fit, great, if not, maybe I can improve it and send a few PRs back.


This wrapper fulfills two conditions:

* ActorRefs are strongly typed
* There are no implicit Sender references; everything is obvious from the Message type signature

That means if you want to send a response, the sender must include his own address in the message he sends.

The C# sample above could look like this then:

```F#
type ActorMessage = string * IActorRef<string>

type GreetingActor() =
    inherit ReceiveActor()
    do base.Receive<ActorMessage>(fun ActorMessage(msg,sender) -> sender.Tell(sprintf "Hello %s" msg))
...

let greetingActorRef : IActorRef<ActorMessage> = ...
let receivingActor : IActorRef<string> = ...
greetingActorRef.Tell (ActorMessage("World!", receivingActor)

let response = ... // now you need to somehow get the response out of the receivingActor
```

Well, it is typed, but is it easier? First of all, how do you spawn the ``receivingActor : IActorRef<string>``? And how do you get the response out of it?

Akka handles this with the Ask pattern (``Task<T> Ask(this IActorRef, object msg)``. Under the hood, it spawns a temporary actor, then takes that actors ref, and sets it as the sender. When the temp actor receives an answer, it completes the underlying Task.


Fortunately, for the typical Query-Response pattern, where you send a message, and want to receive one answer, you don't need this whole ceremony. There is already an established mechanism, the ``ReplyChannel``. Our code sample now becomes

```F#
type ActorMessage = string * IReplyChannel<string>

type GreetingActor() =
    inherit ReceiveActor()
    do base.Receive<ActorMessage>(fun ActorMessage(msg,rc) -> rc.ReplyWithValue(sprintf "Hello %s" msg))
...

let greetingActorRef : IActorRef<ActorMessage> = ...
let! response = greetingActorRef.Ask (fun rc -> ActorMessage("World!", rc))
```

Why the fun? In Akka, you can directly pass a message object to ``Ask``. But the reply channel is part of our message object! So we need the actual channel, before we can construct the message. Since the reply channel is constructed inside ``Ask``, we pass a callback, which is passed the channel and then constructs the message.

The same pattern is used in the F# MailboxProcessor.

This is the public Api that is implemented by my abstraction layer:


```F#
type IActorRef<'T> =
    abstract member Tell : 'T -> unit
    abstract member Raw : IActorRef

type IReplyChannel<'T> =
    abstract member ReplyWithValue : 'T -> unit
    abstract member ReplyWithException : exn -> unit
    
let asTyped<'T> (actor:IActorRef) : IActorRef<_> = ...

type IActorRef<'T> with
    
    member self.Ask<'TResult> (cb:IReplyChannel<'TResult> -> 'T, ?ct, ?timeout) : Task<'TResult> = ...
```

As I said before, I wanted a thin wrapper. That means I create the Actor using the normal Akka C# api, then pass it through ``asTyped<'T>``. From there on, it is type safe, but the asTyped method itself doesn't do any checks. You can easily get the underlying ActorRef back out, which is neccessary if you want to interact with any Akka functionality.

Like Thespian, and unlike the F# mailbox, the reply channel can handle exceptions. That is great for RPC scenarios; if the request failed on the server, you can just pass back the exception, and the Task returned by Ask will fail. A little bit more about that [later](#message-types-unions-and-exceptions).

#### Task vs Async

Speaking about tasks, you may have noticed that I use Task, not Async in my version of Ask.
The unfortunate fact is that .Net uses Tasks. Async is great in F#, but if you want (or need) to interact with the BCL, it is a pain.

So instead of converting back and forth every time, I use [the awesome ``task`` ce from rspeele](https://github.com/rspeele/TaskBuilder.fs) and use
Task everywhere. It's actually better than async/await in C#!

#### Message Types, Unions, and Exceptions

My philosophy with regards to error handling was to use union types for expected errors (``Object already locked``), but exceptions for unexpected errors (``sql statement failed``). This means regardless of what happens on the server, the client will __always__ get a response back (well, as long as the network is not interrupted, anyway).

I once had made an issue in an sql statement inside the actor on the server, but instead of the app-service process crashing, this mechanism meant that the Ask on the client threw an SqlException. __With a complete stacktrace from inside the server!__ Fixing bugs can't get easier than that.

#### Location Transparency and Leaky Abstractions

All technological abstractions are leaky.  
Entity Framework promises that you don't need SQL, but the realistic end result is that you need to lern both SQL ___AND___ all the internals of EF when you use it. Been there, done that, fuck that shit. [Dapper](https://github.com/StackExchange/Dapper) all the way. ([SqlCommandProvider](http://fsprojects.github.io/FSharp.Data.SqlClient/) for F#)

Akka promises Location Transparency, that is, there is no difference between a local actor and a remote actor, all you need to know is the ``ActorRef``. But the reality is, that as soon as you move the second actor onto a different server, the two ActorSystems will _disassociate_, or even _quarantine_ each other, if the _heartbeat_ times out, and you will need to think about that and handle it.

As far as leaky abstractions go, Akka really ain't that bad in my opinion. The remoting code is solid. For example, if you cut the connection, it will disassociate, but it will also automatically reassociate when the connection is restored. Messages that were sent in-between, are lost (Akka implements ``at-most-once`` semantics). Asks will time out (you did set a timeout < infinity, right?). You can detect a remote actor failure by ``Watch``ing an ActorRef. This will inform you when the remote actor either shuts down cleanly, or when the whole remote system is disassociated because of a (hopefully transient) network failure. But you ___do___ need to think about it and handle it.

But the truth is that you would need to think about it regardless of the technology. The network will fail at some point.

``[[todo: point to code samples wrt. death watch / ask / cancellation]]``


#### Debugging

If you code in F#, you don't need debugging anway, because _Types_ will solve all your issues, right? _Yeah..._

``[[todo]]``

tl;dr: debug-stepping and heartbeat timeouts don't like each other

#### Performance

If you read any post about actors, one big topic is performance. You can instantiate millions of actors, and send billions of messages per second between them!

When I started the project (around February 2017), I did a quick benchmark. With multiple local clients and one local server, I could do a few hundred ops/sec. What? I thought _billions_?

In this case, the limit was the cpu, more specifically the tcp stack used.  
By now they have replaced the old one (Helios) with a new one (DotNetty), which should improve performance a lot.

``[[todo]]: Rerun benchmark`` 

But it doesn't matter. __You need to know your requirements__, and in our case __it was enough__.

#### Service Discovery

``[[todo]]``

tl;dr: The service starts itself up, connects to the database and writes the uri of an actor into a table.
The clients read the uri, and connect to the system.

So on both the app-service side and the client side you only need to configure the database.

#### Peer-to-Peer

``[[todo]]: nice picture of a star topology``

We don't actually use p2p (yet), but we do use two-way communication between server and client. We have a NotificationHub. A client can send a message to that actor, and it will broadcast it to all other clients.

With typical client-server solutions (WCF, REST) we would either need to spin up a REST service at each client and tell the server the uri of the client, or implement polling.

With Akka, that is piss-easy. We just spawn a local actor, and ``Tell`` the notification hub the actor ref. Done.


### Reality

So how does my actual code look like?

``[[todo]]: add a fully compiling and working slightly simplified version of an actor (and client)``


### Conclusion

So, what is my conclusion? Akka sucks, is totally buggy, with a shitty api, and I should have used REST?  
Not at all. Sure, I'm picking on Akka and a big part of this post was pointing out its flaws, but I actually like the end result.
Maybe it's stockholm.

If I had to do another new development, I would probably choose REST for a public endpoint, and Akka again for an internal endpoint.  
All depending on the exact circumstances, of course.

For a prototype, I would choose a MessageBus based solution, just so that I know what that is about, and so that I can make a more informed decision next time.

_Go [back](index)_