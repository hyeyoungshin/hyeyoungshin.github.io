---
layout: post
title: "message passing concurrency - an attempt"
date: 2026-02-23
---
I wrote this note while trying to make sense of a few concepts and tie together the loose ends in my understanding. It captures a moment in that process &ndash; questions, uncertainties, and all &ndash; and I’m choosing to share it as it is.


I thought I understood the concept of message passing concurrency in Rust from the book. However, I still had this question while trying to implement the client API function, `sync_message`. 

```scheme
;; Racket code written by Michael
(define (sync-message state-update-channel msg)
  ;; Create a one-time response chanel for this specific request
  (define resp-ch (make-channel))
  ;; Send the request (msg + response channel) to the actor
  (channel-put state-update-channel (request msg resp-ch))
  ;; Receive
  ;; BLOCK waiting for the actor to send back a response
  (channel-get resp-ch))
```

" How do actors and channels work to make message passing concurrency to work in Rust? I thought that there is one special actor that contains business logic (updates state) and multiple actors (maybe representing players?) sending request to this special actor. So I thought there is only one Receiver (the special actor channel's receiver).  
I am not sure how to implement "sync_message".For example, who calls it, what arguments should be passed to it, does it need to create channels, etc.? "

Here is what I've learned to untangle a couple of conceptual layers.

- What actors really are in Rust
- How channels enable message passing
- How a `sync_message` API usually works

### What an "actor" really is in Rust

Rust does not have built-in actors like Erlang or Akka. An "actor" in Rust is just
- A task/thread
- That owns some private state
- And processes messages in a loop
- through a channel receiver

```code
Actor = 
    owns state
    has a Receiver
    loops:
        wait for message
        update state
        maybe send response
```

### My mental model: One business actor + many clients
The way how I think my multiplayer game's concurrency should work describes the following architecture:
- One "speical" actor with business logic 
- Multiple actors (players) sending requests
- Only one Receiver for the special actor

This is a valid mental model.  

### But how do responses work?

If messages only go one way, you don't need anything else.  
But if a client wants a response, then the message must carry a way to reply.  

There are two models:

1. **Model A: Fire and forget (async message)**
Client sends:

```code
"increment score"
```

Business actor:
- Updates state
- No reply

Very simple.  
Only one channel needed.  

2. **Model B: Request-Response (synchronous message)**

Now the client may want:

```code
"what is my score?"
```

This requires a reply. But the business actor does not know who sent the message. So how does it apply?

The client **includes a response channel inside the message**.


### The Important Pattern: Message Contains a Response Channel

Instead of sending `get_score(player_id)`, send 

```code
GetScore {
  player_id,
  reply_to: some_channel_sender
}
```

Now the flow is
1. Client creates a temporary channel for the reply

2. Client sends message containing:
 - request data
 - reply sender

3. Business actor:
 - processes request
 - sends response through reply sender

4. Client waits on the reply receiver

This is how "sync_message" usually works.

### So Who Creates Channels?

There are two types of channels involved here:

1. The Main Actor Channel &mdash; created wgeb the system starts and lives for the entire lifetime of the system
- Business actor owns the Receiver
- Clients hold cloned Senders

2. Temporary Reply Channel &mdash; created per request for sync calls
- Client creates it
- Sends the sender part inside the mssage
- Waits on the receiver part
- Channel gets dropped afterward
