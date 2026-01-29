---
layout: post
title: "client-server game design"
date: 2026-01-28
---

After finishing my single-player, text-based game, I started thinking about how client–server game design works—and got overwhelmed pretty fast. Taking a step back and focusing on a few simple design principles, along with answering some key questions, ended up helping a lot.

### Separation of Concerns
**Question to consider:** What information does each entity actually _need_ to know vs. what they _shouldn't_know?
- What does the **server** need to know to run the game?
  + the solution (mine locations)
  + each player's request 
  + the tile status of all tiles on the board at all time (before and after each player's request)
  + winning condition
- What does a **client** need to to know to display the game state?
  + the tile status of all tiles on the board 
- What should client **never** know (until appropriate)?
  + the solution

### "Never Trust the Client" 
Any data sent to the client can be inspected, modified, or used to cheat. 
**Questions:**  
- If a client knows where all the mines are, what can they do?
  + they can cheat and win by revealing all but mines.
- Should game logic (like "did I hit a mine?") run on the client or server?
  + server
- Who should be the "source of truth" for the game state? 
  + server

### Data Models for Different Contexts
Different parts of the system often need different _views_ of the same data.
- Is there a "full" version of the board with all information?
- Is there a "filtered" or "public" version of the board?
- Could I have multiple structs representing the same logical entity? 


### Example 
```rust
// Server's view (complete)
struct UserAccount {
    id: u32,
    username: String,
    password_hash: String,  // Secret!
    email: String,          // Secret!
}

// Client's view (filtered)
struct PublicProfile {
    id: u32,
    username: String,
    // No password or email!
}
```

### Authority and Validation
**Who decides what?**  
- Should clients tell the server "I revealed this tile and it was safe"?
- Or should client say "I want to reveal this tile" and the server decides the outcome?


Imagine I have

```rust
struct ServerBoard {
    // Everything - including secrets
}

struct ClientBoard {
    // Only what players should see
}
```

**Questions:**
- What fields go in each?
- How do I create a `ClientBoard` from a `ServerBoard`?
- When a player clicks a tile, what gets sent over the network?
- What does the server send back?
