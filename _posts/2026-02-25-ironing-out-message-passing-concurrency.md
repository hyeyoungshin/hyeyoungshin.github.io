---
layout: post
title: "ironing out message passing concurrency"
date: 2026-02-25
---
Eureka!!! Too dramatic? Absolutely not.  

After talking with Michael, the wrinkles in my brain finally smoothed out.  

I’m writing this down to keep it that way. And if my brain gets wrinkled again, I’ll have something to come back to—an iron for the next round.  

A client on a thread send a message to the "state actor" &mdash; I only have one, but there can be more &mdash; who and only who has access to "state" and ability modify it. The client's message is wrapped in a struct called `Request` along with a response channel, because the state actor's main job is to receive and handle clients' requests and respond them, but it doesn't keep track of who sent what, so it needs a way to send back its response to the right client.

### Data Types
**Racket code**
```scheme
(struct request [msg resp-ch])
```

**Rust code**
```rust
enum Msg {
    DisplayState(PlayerId),
    ProcessAction(Action)
}

struct Request {
    msg: Msg,
    reply_to: Sender<Response>,  // response channel
}

enum Response {
    DisplayState(GameState),
    OtherPlayerWon(GameState),
    ActionCommitted,
}
```

### Sync Messages

With these well thought out data types, I need a way to use them wisely. 

A client (on a thread) representing a player needs a way to send request and wait for a response. `sync-message` does this. The wrinkly part for this function was that I thought each client has its own channel to communicate with the state actor. This is possible, but I can also create a one-time response channel for each request. A (temporary) channel **Not per client**, **but per request**.

**Racket code**
```scheme
(define (sync-message state-update-channel msg)
  ;; Create a one-time response chanel for this specific request
  (define resp-ch (make-channel))
  ;; Send the request (msg + response channel) to the actor
  (channel-put state-update-channel (request msg resp-ch))
  ;; (BLOCK) Wait for the actor to send back a response
  (channel-get resp-ch))
```
The word "BLOCK" in the comment bothers me. What does it block? I will get back to it.

**Rust code**
```rust
fn sync_message(state_update_channel: &Sender<Request>, msg: Msg) -> Response {
    // Create a temporary reply channel
    let (resp_tx, resp_rx) = mpsc::channel();
    // Wrap request with reply sender
    let request = Request {msg, reply_to: resp_tx};
    // Send message to actor
    state_update_channel.send(request).unwrap();
    // Wait for response and return it
    resp_rx.recv().unwrap()
}
```


### State Actor

Now, it gets tricky. In the Racket code Michael does something cool. 

`make-object-actor` creates the generic actor by combining a communication channel and a constructor. The cool part is `ctor` which is the functional constructor (or a factory function) for an "object" as in object-oriented programming, but all encoded in a lambda. It has private fields and methods like an object!

`(define o (ctor))` actually runs `ctor` (notice a pair of parenthesis around `ctor`). The function `make-state-actor` uses the generic `make-object-actor` and passes a lambda as `ctor` to create the state actor. 

**Racket code**
```scheme
;; Generic actor framework: runs an object in its own thread
;; The object receives messages and sends back responses
;; Channel (-> Object) -> Void
(define (make-object-actor state-update-channel ctor)
  ;; Create the object (a closure that handles messages)
  (define o (ctor))
  
  ;; The actor's message loop
  (define (loop)
    ;; BLOCK waiting for next request from any client
    (match-define (request msg resp-ch) (channel-get state-update-channel))
    
    ;; Process the message using the object
    ;; Send response back through the response channel
    (channel-put resp-ch (o msg))
    
    ;; Loop forever, handling requests one at a time
    (loop))
  
  ;; Start the loop (this never returns - runs forever in thread)
  (loop))


;;
;; State actor - THE SINGLE SOURCE OF TRUTH
;; All game state lives here, accessed only via messages
;;

(define (make-state-actor state-update-channel)
  (make-object-actor
   state-update-channel
   (lambda ()  ;; ----------------------> functional object constructor (ctor)
     ;; object fields: 1) state and 2) last-displayed-state-for-players 
     (define state (start-game PLAYERS))
     (define last-displayed-state-for-players (hash))
     
     ;; object methods
     (lambda (msg)
       (match msg
         ;; Message: "give me current state for this player"
         [`(display-state ,player-id)
          ;; Remember what state we showed this player
          (set! last-displayed-state-for-players 
                (hash-set last-displayed-state-for-players player-id state))
          ;; Return the current state
          state]
         
         ;; Message: "process this player's action"
         [`(player-action ,a ,player-id)
          (cond
            ;; If game already over, reject the action
            [(game-over? state)
             `(other-player-won ,state)]
            
            ;; Otherwise, update state and confirm
            [else
             ;; CRITICAL: State update happens HERE, atomically
             ;; No locks needed - only this thread touches 'state'
             (set! state (do-action state a))
             'committed])])))))
```

**Rust code**
```rust
fn handle_request(request: &Request, state: &mut GameState, last_displayed: &mut HashMap<PlayerId, GameState>) -> Response {
    match &request.msg {
        Msg::DisplayState(player_id) => {
            last_displayed.insert(*player_id, state.clone());
            Response::DisplayState(state.clone())
        },
        Msg::ProcessAction(a) => {
            if game_over(&state) {
                Response::OtherPlayerWon(state.clone())
            } else {
                *state = do_action(&state, &a);
                Response::ActionCommitted
            }
        }
    }
}

fn server() {
    //...
    // state actor
    thread::spawn(move || {
        let mut state = start_game(NUM_PLAYERS);
        let mut last_displayed = HashMap::new();
        // Loop receiving messages
        for request in state_rx {
            let response = handle_request(&request, &mut state, &mut last_displayed);
            request.reply_to.send(response).unwrap();
        }
    });    
    //rest of server
}
```

### Client Actor

A client (on a thread) 
- sends a request and waits for a response via `sync-message`
- reacts appropriately to the responses it gets from the state actor

**Racket code**
```scheme
(define (handle-client in out player-id state-update-channel)
  (define (game-loop)
    (define st (sync-message state-update-channel `(display-state ,player-id)))
    (displayln (state-view st player-id) out)
    (when (not (game-over? st))
      (define a (action (get-valid-input MAX_TO_GUESS in out) player-id))

      (match (sync-message state-update-channel `(player-action ,a ,player-id))
        [`(other-player-won ,end-state)
         (displayln "Sorry, another player won in the meantime!" out)
         (displayln (state-view end-state player-id) out)]
        ['committed
         (game-loop)])))

  (displayln (format "you are player ~a" player-id) out)
  (game-loop))
```

The tricky part here was to get the owning and borrowing of `reader` and `writer` right. (I had it wrong.) Basically, a client doesn't need to own them, which makes sense.

**Rust code**
```rust
fn handle_client(reader: &mut BufReader<TcpStream>, writer: &mut LineWriter<TcpStream>, player_id: u32, state_update_channel: &Sender<Request>) {
    writeln!(writer, "You are player {}", player_id).unwrap();
    
    loop {
        match sync_message(state_update_channel, Msg::DisplayState(player_id)) {
            Response::DisplayState(state) => {
                writeln!(writer, "{}", state_view(&state, &player_id)).unwrap();
                if !game_over(&state) {
                    let a = Action { player_id, guess: get_valid_input(MAX_NUM_TO_GUESS, reader, writer)};
                    match sync_message(state_update_channel, Msg::ProcessAction(a)) {
                        Response::OtherPlayerWon(end_state) => {
                            writeln!(writer, "Sorry, another player won in the meantime!").unwrap();
                            writeln!(writer, "{}", state_view(&end_state, &player_id)).unwrap();
                        },
                        Response::ActionCommitted => { break; }
                        _ => { panic!("response mismatch"); }
                    }
                }
            }
            _ => { panic!("response mismatch"); }
        }
    }
}
``` 

### Server: where everything fits together

I can see the corresponding rust code for racket code and it's pretty cool how two languages differ and yet carry on the same tasks.

**Racket code**
```scheme
(define (server)
  ;; Create TCP listener
  (define listener (tcp-listen 8000 4 #t))
  ;; Create THE channel for all state updates
  ;; All client threads will send messages through this channel
  ;; Only the state actor reads from this channel
  (define state-update-channel (make-channel))
  
  ;; Start the state actor in its own thread
  ;; This actor has exclusive ownership of game state
  ;; It's the ONLY thread that can modify state
  (thread (lambda () (make-state-actor state-update-channel)))

  ;; Accept players connections and start a thread for each
  (for ([player-id (range PLAYERS)])
    (define-values (in out) (tcp-accept listener))
    ;; Each client gets its own thread
    (thread
     (lambda ()
       (file-stream-buffer-mode out 'line)
       ;; Each client thread communicates with state actor via the SAME channel
       (handle-client in out player-id state-update-channel)))))
```

**Rust code**
```rust
pub fn server() {
    let listener = TcpListener::bind("127.0.0.1:7878").unwrap();
    let (state_tx, state_rx) = mpsc::channel::<Request>();

    // state actor
    thread::spawn(move || {
        let mut state = start_game(NUM_PLAYERS);
        let mut last_displayed = HashMap::new();
        // Loop receiving messages
        for request in state_rx {
            let response = handle_request(&request, &mut state, &mut last_displayed);
            request.reply_to.send(response).unwrap();
        }
    });

    for player_id in 0..NUM_PLAYERS {
        let (stream, _addr) = listener.accept().unwrap();
        let mut reader = BufReader::new(stream.try_clone().unwrap());
        let mut writer = LineWriter::new(stream);
        let state_tx = state_tx.clone();
        // Each client gets a cloned of state_tx
        thread::spawn(move || { 
            handle_client(&mut reader, &mut writer, player_id, &state_tx);
        });
    }
}
```