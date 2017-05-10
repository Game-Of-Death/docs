# The Game of Death

Specificiations for the Game of Death, version 1.1. Author shardulc, copyright 2015--17; permission is given to copy, distribute, modify, or otherwise use this document in any manner provided that this copyright and author notice is preserved and that these same permissions are granted to subsequent users.


## High-Level Overview

The Game of Death is a real-time multiplayer strategy programming capture-the-flag game. The specific meaning of each of those adjectives is:

 - capture-the-flag: The primary objective of each team in the game is to capture or take possession of a 'flag', or within-game token, that belongs to an opponent team, and then to perform a specfied action with that flag. For the purposes of this game, all teams have a 'base' where the team's flag is kept, and a captured flag of an opponent team has to be brought to the capturing team's base to be counted. The game environment is given in more detail in the section "Game Environment".
 
 - programming: Players do not directly interact with flags, opponents, or other game objects. Instead, players give instructions in a specified language to virtual units under their control before the game begins. In the game, multiple (hundreds) of copies of these units will be made, all bearing and following the same instructions, which are immutable until the end of the game. The exact nature of these units and their instructions is given in the section "Virtual Units".
 
 - strategy: Playing the game successfully primarily requires strategy and not dice-rolling luck, fancy equipment, expensive purchases, etc.
 
 - multiplayer: More than 1 players play the game. Each player is responsible for one team of virtual units in the game. (Real-world people may work together to design a team, but they will be collectively counted as '1 player' to keep language simple.)
 
 - real-time: All teams play simultaneously; the game does not involve turns. However, space and time in the game is discrete, as described in the section "Game Environment".


## Game Environment

The game takes place in a grid of squares, each of which can at any point be occupied by a unit, occupied by an object that is not a unit, or empty. The difference between squares occupied by units and occupied by non-units is that units can move (and thus change the set of their occupied squares) on their own, while non-units stay in a location indefinitely. The grid is subjectively very large compared to the units that occupy it: a rough estimate is that the grid is 1000 to 5000 times as long as the average unit.

Spread within the grid are the bases of the teams. Each team has one base, occupying one square and counted as a non-unit. The bases are randomly scattered, but care is taken to not have two bases too close to each other (how close is allowed is a tweakable parameter). As mentioned, each team has an infinite number of flags which are contained in the base and can be taken by opponent teams.

Just as space is divided into discrete squares, time is divided into discrete 'ticks'. A specified portion of a unit's instructions are executed at every tick, for all units on the grid. Any actions a unit performs in a tick can only have effects in later ticks, not the tick in which they were performed. Note: if the game is being observed by humans, ticks would have relatively long durations such as 0.2 seconds, but otherwise the game can progress very fast with microsecond ticks. Obviously, there will be an upper bound to how long a unit is allowed to execute its instructions to ensure reasonable speed.

## Virtual Units

A unit is a collection of squares on the game grid that acts as a single entity. Each unit has a set of instructions given to it by the player who controls it. These instructions have a 'birth' part, which is executed exactly once when the unit is created; a 'life' part, which is executed at each tick as specified in the section "Game Environment"; and a 'death' part, which is executed when the unit is destroyed.

We will call the number of squares a unit occupies its 'size', and the size in bytes of its instructions 'weight'.

Instructions are specified in a format similar to a basic programming language with conditionals (if-statements), loops (for- and while-statements), variables, and function calls. Apart from user-specified functions, certain functions (all of which have no return value) are provided by the environment to all units to interact with the game:

 - `move (direction)`: Move one square in a given direction. Life on a grid implies the direction can either be forward, backward, leftward, or rightward (relative to the unit's orientation).

 - `fire (direction, strength)`: Fire a projectile in a given direction with a given strength. Unlike `move`s, projectiles can be fired in any two-dimensional direction, specified by an angle relative to the unit's orientation, because projectiles do not actually occupy any grid space. Larger strength values correspond to greater speeds, greater ranges after which the projectile vanishes, higher damages if the target is hit, and higher costs to the firing unit (details about speed, damage, and cost below). Since a unit consists of multiple squares, one square has to be designated as the firing origin when the unit is designed. Note that firing at a base with any strength captures one flag from that base.
 
 - `broadcast (message)`: Broadcast a message to all units---friend and foe---at all locations on the grid.
 
 - `die ()`: Commit suicide on the next tick, executing the death instructions if given. Takes no arguments. May be a legal offense in some countries.

In addition the environment provides some variables to each unit:

 - `integer x` and `integer y`: Current absolute grid coordinates of the unit as integers. 'Absolute' means that if two units are at the same location (at different times) they will have the same coordinates whether they are on the same team or not.
 
 - `square env[integer][integer]`: A view of the grid currently surrounding a unit, up to a certain visibility (e.g. 10 squares; this is a tweakable parameter). This variable is a list of `square`s indexed by an integer for a unit-relative `x` and an integer for a unit-relative `y`: `square`s are a data type which can either be `empty`, `unit` (occupied by a unit), or `nonunit` (occupied by a non-unit).
 
 - `(integer, integer) bases[text]`: A list of the absolute coordinates `(x, y)` of all the teams in the game, indexed by the name of the team as text. (Units are expected to know the name of their own teams.)
 
 - `hp`: Current HP or health points of the unit as a real number not less than 0 and not greater than 1. This is indicative of the health of a unit. By default, HP is incremented once in a tweakable number of ticks depending on the unit's size and weight, unless the unit is hit by a projectile, in which case the HP is decreased according to a tweakable formula combining the projectile's speed and the unit's size and weight.
 
 A unit dies when its HP reaches or goes below 0. When a unit dies, the squares it occupied are marked as `nonunit`. These squares will then block any projectiles from passing through them and will be opaque to other units trying to access their `env` variables beyond them.
 
 - `power`: Current firing power, a real number very similar to HP. This indicates the firing power of a unit, is incremented like HP with optionally different parameters, and is decremented when a unit fires a projectile depending on a tweakable formula combining the projectile's speed and the unit's size and weight. When the power reaches or goes below 0, a unit cannot fire more projectiles until it regenerates to a tweakable value.
 
 - `flag`: Which team's flag the unit currently holds. This is the name of the team, or empty text if the unit does not hold a flag. A unit can only hold one flag at a given time.

A special note must be made about the `move` function. When a unit calls the `move` function, it of course does not move on that very tick, but when it does actually move is a tweakable function of the unit's size and weight. For example, an implementation might decide that a unit with double the weight of another unit takes twice as many ticks to move by one square. Thus, calls to `move` can be thought of as pushing directions into one end of a pipe: they will take some time to move to the other end and when they come out, they will be executed. For example, these ticks may occur for a pipe of length 2:

 1. `move(forward)` The pipe is now `F `.
 2. `move(rightward)` The pipe is now `RF`.
 3. `move(leftward)` The pipe is now `LRF`, but the pipe is of length 2 so the right end falls off: `F` is executed and the unit moves forward one square.

## Gameplay

At the start of the game, all of a team's units are arranged around the team's base. Their initial formation can be specified by the player, or it can be randomly assigned. In typical play, each team will have a few types of units (say 5) and up to hundreds of units of those types (say 50 of type 1, 100 of type 2, etc.). Bases must obviously be far enough apart.

Each unit's birth instructions are run. Then, the game begins and the ticks are started, with the units' life instructions being repeatedly executed. When a unit captures a flag, it *must return alive* to its own base to have the flag counted. The game's end can be some tweakable combination of:

 - a certain number of ticks
 - a certain number of dead units, possibly an entire team
 - a certain number of captured flags for a team
 - anything else the players/implementations provide for

The high-level format of the game, such as a tournament system, awarding points to players, giving up the code controlling units as trophies, etc. is up to the players to maximize fun. However, it is recommended that players have a long enough time span to design their units and test the instructions before competing against other players. It should be evident from the structure of the game that the emphasis is on designing units to collaborate based on simple rules and not to design a super-unit with thousands of lines of code.

## Implementation

The game implementation requires three major parts: the backend simulator for maintaining the grid structure, the units, running the instructions, managing HP, power, move calls, etc.; the design front-end, in which players have an interface to visually design and arrange units, write instructions, and be able to test them in a 'live' simulation debuggable and controllable by the player; and a simulation front-end, for players to set up and optionally view games in real-time.

All instances of the word 'tweak' in the specification are to be concretely implemented by the implementation such that they fulfill the objectives adequately: for example, greater size should increase the time it takes for a unit to move. Tweaking can even be up to the players but the same rules and functions must be applied to all teams' units. These details cannot be tweaked during the game, only before it.

## A Personal Note

I had this idea two years ago, but have been too lazy to actually write the code. The motivation was to design a game that required only strategy and not luck, equipment, etc. such that humans would---at least for a few decades from now---be better than computers at playing. In 2016, this increased in relevance as AlphaGo beat world champion Lee Sedol at the game of Go, one of the hardest challenges for AI.

I was inspired by Conway's Game of Life for the name of the game and more importantly, for the simplistic design and the idea that complex behavior can be the result of simple rules in a grid-based two-dimensional world. I encountered many games like Screeps, Battlecode, and various AI tournaments that had almost what I had in mind, but not quite, hence this document.

I hope this is fun!
