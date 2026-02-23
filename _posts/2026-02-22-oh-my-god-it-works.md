---
layout: post
title: "oh my god it works"
date: 2026-02-22
---

By "it" I mean shared state concurrency! 

To make my Rust learning project an asynchronous multiplayer game, I needed to learn some concurrent programming in Rust. Michael suggested that I practice on a simpler game "number guessing". He showed me two different concurrent programming methods 
- shared state with box, unbox, and box-cas
- message passing 
in Racket, which was very helpful.

Here is Michael's Racket code:

```scheme
;; ============================================================================
;; SERVER AND MULTI-THREADING CODE
;; ============================================================================

;; Main server function - sets up TCP listener and spawns threads for each player
(define (server)
  ;; Create a TCP listener on port 8000
  ;; Parameters: port, max-queue-size, reuse-address?, hostname
  (define listener (tcp-listen 8000 4 #t))

  ;; Create a BOX containing the game state
  ;; Box = mutable container that can be atomically updated
  ;; This is the SHARED STATE that all player threads will access
  (define game-state (box (start-game PLAYERS)))

  ;; For each player (0, 1, 2), accept a connection and spawn a thread
  (for ([player-id (range PLAYERS)])
    ;; Accept a TCP connection - BLOCKS until a client connects
    ;; Returns two ports: in (for reading from client), out (for writing to client)
    (define-values (in out) (tcp-accept listener))
    
    ;; Spawn a new THREAD for this player
    ;; Each player gets their own thread running concurrently
    (thread
     (lambda ()
       ;; Set output buffering to 'line so messages are sent immediately
       ;; Without this, output might be buffered and delayed
       (file-stream-buffer-mode out 'line)
       
       ;; Handle this client's connection
       ;; Each thread runs this independently with its own player-id
       ;; but they ALL share the same game-state-box
       (handle-client in out player-id game-state-box)))))


;; Atomically updates the game state with an action
;; Uses Compare-And-Swap (CAS) for lock-free concurrency
;; Box GameState Action -> Boolean
(define (try-and-commit-action! game-state-box a)
  ;; Read the current state from the box
  ;; Note: Another thread might change this before we commit!
  (define current-state (unbox game-state-box))
  
  ;; If game is already over, can't apply action
  (if (game-over? current-state)
      #f  ;; Return false - action rejected
      
      ;; Try to atomically update the box using Compare-And-Swap (CAS)
      ;; box-cas! does: "if box still contains current-state, replace with new-state"
      ;; Returns #t if successful, #f if another thread changed it first
      (or (box-cas! game-state-box 
                    current-state              ;; Expected current value
                    (do-action current-state a))  ;; New value to set
          
          ;; If CAS failed (another thread updated first), RETRY
          ;; This is OPTIMISTIC CONCURRENCY - keep trying until we succeed
          (try-and-commit-action! game-state-box a))))


;; Handles one player's connection - runs in its own thread
;; InputPort OutputPort PlayerId (Box GameState) -> Void
(define (handle-client in out player-id game-state-box)
  ;; Inner recursive function for the game loop
  (define (game-loop)
    ;; THREAD-SAFE READ: Get current game state from shared box
    ;; Multiple threads can read simultaneously - this is safe
    (define st (unbox game-state-box))
    
    ;; Send the current state view to THIS player
    ;; Each player sees their own personalized view
    (displayln (state-view st player-id) out)
    
    ;; If game is not over, get this player's next action
    (when (not (game-over? st))
      ;; Read and parse player's guess from their TCP connection
      (define a (action (get-valid-input MAX_TO_GUESS in out) player-id))

      ;; CRITICAL SECTION: Try to atomically update shared state
      ;; This might fail if another player wins while we're processing
      (when (not (try-and-commit-action! game-state-box a))
        ;; Our action was rejected because game ended
        (displayln "Sorry, another player won in the meantime!" out))
      
      ;; Continue the loop - check new state and possibly get next guess
      (game-loop)))

  ;; Send welcome message to this player
  (displayln (format "you are player ~a" player-id) out)
  
  ;; Start the game loop for this player's thread
  (game-loop))


;; Start the server!
(server)
```

My task is to achieve the same safe concurrency in Rust.

There is no `box` or `box-cas!` in Rust, but using `Arc` and `Mutex` (and of course Rust's type system and ownership rules) I could achieve safe shared state concurrency. 

Of course, this was not too straightforward.

Some of the challenges I faced are 
1. deciding to own or borrow 
2. multithreading with a closure
3. useful macros that I didn't know I needed
4. (smart) lock releasing pattern


### Deciding to own or borrow
This is something I am still getting used to.

I had this function `do_action` which takes `GameState` and `Action` and returns an updated state.  
The arguments, old state `st` and action `a`, are no longer useful after a call to `do_action` so I gladly let the function own them.

```rust
// Process a player's action and returns the new game state
fn do_action(st: GameState, a: Action) -> GameState {
    match st {
        GameState::Over(_) => st,
        GameState::InProgress(secret_num, hash) => {
            if secret_num == a.guess {
                GameState::Over(a.player_id)
            } else {
                let new_hash = hash.update(a.player_id, a.guess);
                println!("{:?}", new_hash);
                GameState::InProgress(secret_num, new_hash)
            }
        }
    }
}
```

However, I questioned this decision while implementing `try_and_commit_action` function.

```rust
// Atomically updates the game state with an action
fn try_and_commit_action(game_state: &Arc<Mutex<GameState>>, action: &Action) -> bool {
    let mut current_state = game_state.lock().unwrap();

    if game_over(&current_state) {
        false // action rejected
    } else { 
        *current_state = do_action(&*current_state, action); // PROBLEM
        true
    }
}
```

The updating code was problematic with `do_action` trying to own `current_state`. So I was stuck with the decision to own it or borrow it here or there.   
Well, the answer was clear after I learned that *`GameState` is owned by `Mutex` the only way to use it is to borrow it*. This mutex rule also allows me to use the shared game state more safely. I welcome the restriction for extra guarantee. 


### Multithreading with a closure

I wasn't bit hard by Rust on this one, because I read the book first. So I was prepared. 

The following example is from the Rust book.

```rust
use std::thread;

fn main() {
    let v = vec![1, 2, 3];

    let handle = thread::spawn(move || {
        println!("Here's a vector: {v:?}");
    });

    handle.join().unwrap();
}
```

I needed multiple threads to handle multiple players sending their guesses to the server simutaneously. I learned that I need `thread::spawn` with a closure, suspended computation, that will eventually run on the (guest) thread (as opposed to the main thread). The thing I think would made my job harder is the `move` keyword in front of the closure. It's something I probably would have never though of using if I didn't read the book. (Rust compiler errors are usually very helpful. Perhaps, it would have suggested that I use it.). 

The purpose of `move` is for the spawned thread to take the ownership of the value `v`, so it is guaranteed to be available for the thread. This prevents a possibility of the main thread to drop `v` or modify it.  

Rust’s ownership rules make me think a scenario that I might have missed otherwise!  

### Useful macros that I didn't know I needed: writeln!

For my server to take player's request (guesses) I learned about `BufReader<TcpStream>` and `LineWriter<TcpStream>`. Since I am trying to learn to use the Rust documents, I looked up `LineWriter` and its methods. The method I thought I needed to write to `TcpStream` was [`write_all`](https://doc.rust-lang.org/std/io/trait.Write.html#method.write_all). 

I used it and the compiler didn't complain. 

```rust
// ...
   if !try_and_commit_action(current_st, action) {
       writer.write_all(String::from("Sorry, another player won in the meantime!").as_bytes());
    }
// ...
```

But this wouldn't have worked exactly mainly because my client reads with `read_line()` which waits for `\n`.  

The same can be correctly achived by `write_all` combined with `format!` or just `writeln!`. The latter is "the Rust way" for network communication.  

```rust
writer.write_all(format!("Sorry, another player won in the meantime!\n").as_bytes()).unwrap();
writeln!(writer, "Sorry, another player won in the meantime!").unwrap();
```

### (smart) Lock releasing pattern

Before getting to (smart) lock releasing pattern, I want to write a little bit about Box CAS and how to mimic that in Rust.

**Racket Box CAS:**

```scheme
(define box (box 5))

(box-cas! box 5 10)  ; If box contains 5, set to 10
; Returns #t if successful, #f if value changed
```

**Rust Equivalent with `Arc<Mutex<T>>`:**

```rust
use std::sync::{Arc, Mutex};

let shared = Arc::new(Mutex::new(5));

// Compare-and-swap logic
fn cas(shared: &Arc<Mutex<i32>>, expected: i32, new: i32) -> bool {
    let mut data = shared.lock().unwrap();
    
    if *data == expected {
        *data = new;  // Swap
        true          // Success
    } else {
        false         // Failed - value changed
    }
}

// Usage
cas(&shared, 5, 10);  // true - changed 5 to 10
cas(&shared, 5, 15);  // false - no longer 5
```

**Key difference:** Rust uses Mutex locks and Racket uses lock-free CAS. Both of these are aiming for thread-safe updates. 

There is the "true CAS" method `compare_exchange` offered by Atomics, but it only works with limited types: `AtomicBool`, `AtomicI32`, `AtomicU32`, `AtomicUsize`, `AtomicPtr`. It doesn't work for Structs.


```rust
use std::sync::atomic::{AtomicU32, Ordering};
use std::sync::Arc;

let shared = Arc::new(AtomicU32::new(5));

// Arc's true CAS - lock-free!
let result = shared.compare_exchange(
    5,   // expected
    10,  // new value
    Ordering::SeqCst,
    Ordering::SeqCst
);

match result {
    Ok(_) => println!("Success!"),
    Err(actual) => println!("Failed, actual value: {}", actual),
}
```

If I want CAS like update for complex types, I need `Muitex` and implement CAS logic myself:

```rust
fn cas_gamestate(
    shared: &Arc<Mutex<GameState>>,
    expected: &GameState,
    new: GameState,
) -> bool {
    let mut state = shared.lock().unwrap();
    
    if *state == *expected { 
        *state = new;
        true
    } else {
        false
    }
}
```

Moving on to the main topic of this section "(smart) lock releasing pattern", I needed to write a function `handle_client` that will be run on spawned threads. 

```rust
thread::spawn(move || {
    handle_client(reader, writer, player_id, shared_game_state)
});
```

`handle_client` takes a reader and writer, a player id, and a **shared** game state. The "client" is a player. So `handle_client` handles each player on its own thread while sharing the same game state with other threads. It runs in a big loop getting an access to the shared state and updates it based on the client's input. For it to use the shared state safely I used Mutex lock mechanism like so

```rust
// ...
    loop {
        let current_st = shared_game_state.lock().unwrap();
        
        // send current_st view to the player
        // check if game over
        // construct an action based on player input
        // try and commit action
    }
/// ...
```

The problem with the way I used lock here is that the **lock is held too long**.  

The (smart) lock releasing pattern I learned is creating a local scope to lock the shared state and cloning it and binding the copy to a variable!

```rust
    loop {
        // THREAD-SAFE READ: Get current game state from Mutex (shared box)
        // Multiple threads can read simultaneously 
        let current_st = {
            let state = shared_game_state.lock().unwrap();
            state.clone() // copy the data so I can use it after the lock is released
        }; // lock released here
    // ...
    }
```

It is helpful to note that `current_st` is for **reading safely** (without holding lock). For updating the shared state, I need to use the actual `shared_game_state`.  

Next, I am going to try message passing concurrency.  
