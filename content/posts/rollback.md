---
title: "Making a GGPO-style rollback networking multiplayer game"
date: 2023-01-14T07:33:11-08:00
draft: false
tags: ["gamedev", "ggpo", "multiplayer"]
ShowToc: false
---

## Intro

We will be making a simplified multiplayer clone of the arcade game, Mario Bros. It will have no deaths and a single infinite level. It will have rollback networking and should provide a smooth experience for people even with higher latencies.

![Game Screenshot](/images/rollback-game-screenshot.png "Screenshot")

The source code is [here](https://github.com/lostlevels/rollback-networking). The game is playable [here](https://rollback-networking.pages.dev).

### Rollback Networking

Rollback networking is an elegant form of netcode that can be simple and effective to implement. It works for games of all types from twitchy fighting games to slow-paced turn based ones. Players send their inputs to peers and each player will simulate the game with these inputs. Each player should arrive at the same game state. A local player's input will be applied immediately. Once a remote player's input arrives, the game will need to rewind from the time of the remote input, apply the remote players input for that frame, and resimulate up to the current frame. This means it must keep track of previous game states, not just the current moment.

![Rollback Local Input](/images/rollback-local.svg "Local Input")
![Rollback Remote Input](/images/rollback-remote.svg "Remote Input")

Rollback networking has many advantages. Because there is no central server to send and receive a response from, the latency is effectively halved. Rather than having a round trip from player A to a central server and awaiting a response, player A can send their inputs to player B and continue simulating. Because inputs are the only thing that are sent, bandwidth is also reduced as the state of the world never needs to be continuously sent other than the initial state.

However, rollback networking can be more computationally expensive. Because it needs to continuously apply peer inputs, it may end up simulating many frames for each peer input. This is because a peer's input will be from a past time. The system will need to apply the peer's input in the past and resimulate all the way up to the present time. You can watch more of the idea behind this in [this GDC talk](https://www.youtube.com/watch?v=7jb0FOcImdg). Rollback networking can also be more difficult to implement if the game was not designed for it as the simulation must be deterministic. It can also be very difficult to debug when and why a simulation diverges.

### Determinism

As stated previously, in order for the simulation to work properly, your game **must** be deterministic. For any given input and state, the resulting state must be the exact same for all players. This will allow us to only send inputs without having the simulation diverge. This means keeping the following things in mind while developing:

- Fixed point numbers over floating point
- Reproducible randomness
- Same update ordering
- Fixed simulation timestep

### Fixed point numbers

Floating point numbers are not exact. Imprecisions on different platforms may lead to diverging simulations, e.g., a collision being detected on one device but not another. Because of this we will use [fixed point numbers](https://en.wikipedia.org/wiki/Fixed-point_arithmetic). The gist of this is all our numbers will be "integers" with a fractional and decimal part. The math behind this is simple and is implemented in `js/fixed.js`. All floating point numbers, such as object positions and velocities, will be represented as fixed point instead of floating point. Division and multiplication will be done through fixed point functions:

```javascript
player.x += mulFixed(dt, player.speed);
// as opposed to
player.x += player.speed * dt;
````

### Randomness

We want (pseudo) randomness in our game for certain events and actions. For this, I have used a mersenne twister implementation found on [github](https://gist.github.com/banksean/300494). The current game frame will be the seed for our mersenne twister. Usage will be as so:

```javascript
const rand = new MersenneTwister(currentGameFrame);
if (~~(rand.random() * 100) < 50) {
  // do something
}
````

### Update Ordering

Objects must update in the same order. Player A and Player B must update their characters in the same order. This is to avoid situations such as both players touching a coin at the same time and having different player's scores increase. In this game, we will always update the player with the lexicographically smaller id first for simplicity.

### Fixed simulation timestep

Each player must simulate things with the same simulation [timestep](https://gafferongames.com/post/fix_your_timestep/). This is for determinism. For this game, we will use a simulation timestep of 1/60 a second. This is represented as a fixed point number `DT` and can be seen in `js/constants.js`. The simulation frame rate would be `1/DT` = `60 fps`. Note, this is the *simulation* frame rate, **not** the *rendering* frame rate. The simulation frame rate is independent of any rendering. 

### Code

The source code is available [here](https://github.com/lostlevels/rollback-networking). I will give a high level overview of each component.

#### General components

- `fixed.js` - Fixed point arithmetic.
- `mersenne-twister.js` - Javascript seedable mersenne twister implementation taken from [here](https://gist.github.com/banksean/300494).
- `connectionhandler.js` - wrapper around [PeerJS](https://peerjs.com/), a WebRTC library. Wraps PeerJS to handle connection lifecycle and dropped packets.
- `constants.js` - Some configurable options around screen resolution and shared constants used by all objects. Rather than make this dynamic or configurable, for simplicity these will be predefined.
- `collision.js` - Collision detection and response math functions.

#### Game specific components

- `block.js` - A Block is the main collidable thing in the game. Players can jump and headbutt them to flip an enemy. If it is a question block, it will also create an object when headbutted.
- `enemy.js` - An enemy is the main thing that players interact with. Touching an enemy will either cause the player to lose points and be temporarily frozen or gain points depending on whether the enemy is flipped or not.
- `player.js` - Playable character that responds to input and interacts with the world.
- `projectile.js` - Anything the player can kick to activate. Hurts players and enemies alike if collides.
- `pickup.js` - Coins to pickup for increased score. Could be a star, weapon, etc in the future.
- `spawner.js` - Spawns enemies at periodic intervals.
- `map.js` - The initial game state with enemies, blocks, and players.
- `gamestate.js` - Holds the entire game state. Simulation can be advanced forward one frame by calling `.update`
- `world.js` - Manages the GameState history for rewinding and simulation. Each GameState is saved in it's entirety for simplicity, as opposed to having a compressed "SaveState" or only recording delta changes from the previous state.
- `gamescene.js` - An extended `Phaser.Scene` that uses `ConnectionHandler`, and manages game state. It also syncs the sprites/views with the models/game state.

#### Potential improvements

- Collision partitioning - Currenty, players and enemies check collisions against all other objects. In a game where there are many more blocks or enemies, partitioning the objects can improve performance if less objects are checked per entity.
- Reuse GameState - Each frame a new GameState is created from the previous frame's GameState. This is for simplicity. Rather than creating a new GameState every frame, GameStates can be reused. This may reduce memory allocations. Rather than saving the entire GameState, you could also save a smaller object - like an actual save - as well.
- Alternating player updates - as mentioned previously, one player always updates before the other. This means if both hit a flipped enemy, player one always gets the score. Instead we can either randomize who goes first or alternate the player for fairness.
- Adaptive buffering of remote inputs - Right now in `World.applyRemoteInput`, the world always maintains a fixed sized buffer of inputs to ensure there is always an input from the remote player to apply each frame. This has worked well enough for me in practice, but a smarter algorithm could take into account latency and average time and standard deviation between packets received.
- Input prediction - Some rollback implementations use the remote player's previous input as their next predicted input. I personally find it leads to a more jarring snapping back of a player's position when the actual input arrives and differs so that was not implemented here, but it may work well for you.
- Collision - the bounding boxes at the moment have a lot of extra space. A tighter bounds can be added.
