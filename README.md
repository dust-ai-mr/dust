# Dust
Java 21 saw the introduction of virtual threads into Java which allows creation of Java programs 
comprised of tens of thousands (or more) of threads.

Of course a question instantly arises:
"I already have trouble managing conventional platform threads (locking, semaphores etc etc) 
which would only become worse as the number of threads increases. How does Java 21 help me?"

The answer to this question is simple: it doesn't. Java's designers took the elegant step of 
making thread management exactly the same for platform (non-virtual) and virtual threads once the thread
has been created - both implement the Thread class. While this makes the transition from a 
thread-poor to a thread-rich environment easy, it was not designed to change any aspect of 
thread management. 

That is needed is a different model for creating programs - one that exploits the ability to have
an application with one million concurrent threads, but one in which thread management (and all
that implies) sinks back into the framework where it belongs.

At least one such model exists.

## Actors
The idea of Actors is not new - indeed it was described in the 1970s (<a href="https://arxiv.org/pdf/1008.1459">link</a>).
Variations of the Actor model have been implemented as languages (Erlang, Elixir) and frameworks (Akka).
While the implementations may vary there is a common underlying model:
* Applications are built up out of _Actors_
* An Actor is cheap to create and destroy
* An Actor accepts _messages_ and sends messages to other Actors. This is the only way
an Actor communicates with the world outside of it.
* An Actor has internal state which is inaccessible outside of the Actor. 
It may mutate this state as it receives messages.
* What an Actor does in response to a message is called its behavior. Part of an Actor's behavior 
can be to change its response to subsequent message i.e. change it behavior.
* Actors process messages in the order in which they were received. 
If it has no message to process it waits. 
* An Actor has a single thread of control - receive a message, process it, 
receive the next message ...

So an Actor is single threaded and all mutable state is internal to an Actor. Therefore there is no
shared state and no synchronization issues to worry about.

Clearly, in Java 21 there can be a natural 1-1 mapping between Actors and virtual threads. Moreover
Actors often spend a lot of time waiting for messages to arrive. This causes the Actor's
virtual thread to be suspended, but by the design of Java 21 this _does not block_ 
the underlying platform thread. Consequently there is no processing cost to simply waiting. Actors can
wait for months or years if the application needs it.


## The Dust Actor Framework
Dust is a lightweight implementation of Actors for Java 21 and beyond. It manages the creation of 
millions of Actors on modest hardware. Dust Actors can be distributed across a network and since
every Actor has a unique name (basically a pathname) messages can be sent to local or remote Actors
transparently.

Dust is open-source (Apache2 license).

As a simple example here is the code for an Actor which just reports what messages it receives and who sent them.
<pre>
/**
 * Like it says on the can. Just log every message it receives
 *  @author alanl
 */
@Slf4j
public class LogActor extends Actor {
    /**
     * create Props
     * @return the Props
     */
    public static Props props() { return Props.create(LogActor.class); }
    /**
     * Default behavior - every message is logged. Most Actors will do a switch() (or branch) on message.class
     * to handle different messages differently.
     */
    @Override
    protected ActorBehavior createBehavior() {
        return message -> { log.info(String.format("%s got message %s from %s", self.path, message, sender)); };
    }
}
</pre>

Another Actor could create an instance of LogActor and send it a message: 

<pre>
    actorOf(LogActor.props()).tell("Hi there", self)
</pre>

Here actorOf(LogActor.props()) creates the Actor and returns a reference to it. tell() sends it a 
message (here a string but a message can be any serializable class) 
and 'self' is a reference to the sending Actor (so the recipient knows who the message is from).

## The Dust Repos
Dust consists of a number of different repos

<a href="https://github.com/dust-ai-mr/dust-core">dust-core</a> - the heart and soul of Dust. Actor creation, 
persistence and lifecycle management along with a large collection of Actors to enable the easy creation of 
applications in _idiomatic_ dust. Entity Actors to support digital twinning and modeling. Extensive documentation
<a href="https://github.com/dust-ai-mr/dust-core/blob/main/docs/dust-core-1.0.1.pdf">here</a>.

<a href="https://github.com/dust-ai-mr/dust-http">dust-http</a> - Actors for creating HttpClients and interacting with
web end points. Common messages for request/response and error handling.

<a href="https://github.com/dust-ai-mr/dust-html">dust-html</a> - Actors and msgs for processing html documents 
(as used in dust-http). Support for JSoup (page cleaning, parsing) and heuristics for extracting 
the _core content_ of web pages (using 'Goose' as a base).

<a href="https://github.com/dust-ai-mr/dust-feeds">dust-feeds</a> - Built on the above provides Actors for processing and
managing RSS feeds, crawling web sites and using SearXNG for web search.

<a href="https://github.com/dust-ai-mr/dust-nlp">dust-nlp</a> - Built on dust-core and dust-http provides Actors to 
interact with OpenAI and Ollama LLMs (and a generic Actor for OpenAI-like apis which can be tailored to fit 
other providers). Actors for chunking text and creating embeddings (Huggingface embedding but again can easily
be adapted to other APIs.) 

## But what is it good for?
A question that should be answered is 'what kinds of applications is Dust good for?'. Following is a list of 
rules-of-thumb for recognizing possible good fits. As it happens, dust (or its predecessor) has been used in 
such applications.
* Digital twinning and event-based simulations - according to McKinsey 
"A digital twin is a digital replica of a physical object, person, system, or process, 
contextualized in a digital version of its environment. Digital twins can help many kinds of organizations
simulate real situations and their outcomes, ultimately allowing them to make better decisions."

Dust's Entity Actors are designed to make creating a 'digital replica of a physical object, person, system, or process'
simple and straightforward. Actor messages determine how the various entities interact with each other.

A very simple example of twinning/simulation can be found at <a href="https://github.com/dust-ai-mr/dust-ville">dust-ville</a>



  
* Dynamic Focus - change the details of ongoing monitoring based on the contents of the data stream.
  * Monitor business news feeds for company mentions. If there is an uptick in mentions of a particular product
  then add this product to things to watch more closely. As the mentions fall off stop this deeper monitoring.
  A system was built using Dust's predecessor to do just this:
  
  <img src="/xray.png"/>
  
  * Network monitoring detects an anomaly at a server. Turn on deeper packet inspection on all routes into
  the server until the anomaly has been understood and mitigated.


     
* Sophisticated workflows










