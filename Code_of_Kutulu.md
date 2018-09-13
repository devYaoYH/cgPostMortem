Hello fellow codingamers! Ended up **#25** with a simulation approach in Python3 :stuck_out_tongue:

First off, congratulations to the community team JohnnyYuge, Nmahoude, and EulerscheZahl (Toad_of_Kutulu haha) for another great contest I really enjoyed! Stunning performance by Blasterpoard in his first contest as well, holding pole position for most of the contest! :clap: 

##Kutulu

I enjoy games with discrete movement (grids) and no collisions!!! (phew) Meanmax/FantasticBits were a headache unfortunately. CodeRoyale had a much better handling of collisions (resolving up to a finite 5 sets of collisions via discrete movements in 0.2 frames). This left me much more time to explore decision strategies rather than coding boilerplate code for physics simulations...

Seems like EulerscheZahl ranked right above me at #24 running 1ms on c#. A 50x speed boost from Python3->C# sounds about right (likely another magnitude higher). Gonna to try porting my bot over into C++ and add enemy simulation to get higher performance when multi comes out. Not too familiar with the language so didn't attempt it in contest time-frame.

Movement using UP, RIGHT, DOWN, LEFT should be explicitly stated in the rules. I only found this one out by watching replays. Rules should be kept up-to-date with any changes/errors (Slasher state transition from SPAWNING->RUSHING immediately caused quite some confusion but wasn't updated in the rules).

##My Strategy

###Scoring Moves
As many of you have pointed out, the initial Wood1 boss was *really* strong... In past contests, rushing a simple heuristic would get me past into Bronze but in this contest I had to compute Wanderers' next move and search to depth 1 movement before getting promoted. In the end, this code got me into Silver as well (this bot was around top50 when Silver opened).

I assumed all wanderers move towards me (pessimistic) and scored adjacent cells, picking the highest scoring to travel in. If I can afford to WAIT, use some delicious spaghetti to decide which skill to use.

While my heuristic bot was climbing it's way into Silver, I spent the time implementing a simulation to process current state into the next. The finite state automata of Wanderer/Slashers were pretty easy to understand and I was able to get a sim working with sufficient accuracy (wanderer spawns as well) by Tuesday.

Unlike past contests where I had to really dig into the referee code, the rules were clear enough and simple enough to implement without looking through the referee. Some of the things like path priority cycling through URDL was not stated in the rules and I only knew about that looking at the chat haha (thx!) and Magus noticing the 4/2 bugged duration of PLAN/LIGHT instead of the stated 5/3 duration.

Overall I found the simulation to be much easier to implement than in recent contests.

###Optimizations

Python3 is obviously slow, so to prevent myself from using too much time calculating the paths of wanderer/slashers, I instead ran a BFS from each valid cell to exhaustion, recording which cell it propagated from, then flipped that direction and stored it in a huge dictionary. Something I picked up doing Battlecode :stuck_out_tongue: 

<img src="https://forum.codingame.com/uploads/default/original/3X/c/3/c373c0562e173190bd8225bd979e92e0a43a6d83.png" width="300" height="300">
Screen-cap from an old [Battlecode video](https://www.youtube.com/watch?v=JYLhcUpdKIE)

So here you can see in order to path to the cell circled in Purple, all we have to do is query the direction recorded in the cell we are on and execute that move! And this works from anywhere else on the map towards this targeted cell. :smiley: 

Unfortunately, even with 1000ms, python3 can't compute this exhaustive BFS 4x* in each of the priority directions (for me to then cycle through as the rounds progress) so I only stored and used the URDL ordered one, some inaccuracies but works sufficiently well (was running into the occasional wrongly simulated wanderer but this was relatively rare).

*Tried doing precomp offline, identifying maps using their string and copying a hardcoded dictionary over but with that many maps and essentially storing 4\*(numCells^2) values per map, I ended up with somewhere around 1m characters...*

Another  optimization was to precompute all valid adjacent directions on each cell rather than cycling through URDL and checking for validity every time we want to propagate.

I still computed Slasher LoS cells every turn tho, should've precomputed as well! (realized this after reading MSmits PM)

###A*Star (or BFS with pq and heuristics)

I was inspired by the A*Star search after searching around and finding this nifty tutorial on [maze pathing](http://bryukh.com/labyrinth-algorithms/). It seemed to me that we have a pretty straightforward fit for this game into A*Star pathing. Instead of a distance heuristic, we optimize for length of survival and end sanity score.

```python
t_bfs = time.time()
pq = queue.PriorityQueue()
while (not pq.empty() and time.time()-t_bfs < time_left):
    priority_score, move, state, path = pq.get()
    new_state = procState(move, state)
    # Evaluate state
    # potentialScore() -> gives a heuristic score to board position to tie-break sanity
    new_score = new_state.sanity + potentialScore(new_state)
    # Propagate
    for adj in adj_map[new_state.position]:
        # min-heap so we inverse our score
        pq.put((-new_score, adj, new_state, path[:]+[adj]))
```
And that's the gist of my A*Star search algo used to path through possible moves. The sanity loss per turn provided a nice dropoff in priority_score so that I'll start to explore previously 'bad' paths after going a few rounds being isolated or when I get Spooked some time into the future. This isn't ideal but it works decently, allowing me to find a path which will allow me to have the highest sanity longest.

I have YELL and PLAN simulation but no LIGHT simulation other than a heuristic to use it if I can impact wanderer's targeting and WAIT was the best option. YELL/PLAN was seeded into the pq at the start with some spaghetti and allowed to play out and evaluated alongside initial moves.

Whilst exploring these paths, I'll record the highest-scoring deepest path thus far for each available initial moves (a comparison of len(path) then priority_score). Once time ran out, I'll look through the stored moves and select the one with the highest score.

**Things that worked:**

- Changing my scoring from end_state.sanity into an accumulative scoring of state.sanity scaled by 1/x (x: depth) + end_state.sanity (cuz it's end sanity that matters - but this prioritized better immediate rewards which mitigated inaccurate predictions)

- Adding a higher weighing to nearest explorer (makes me beeline towards them on large separated maps)

- Penalize running into a dead-end heavily (avoid dodging into a corner and inevitably going insane)

**Improvements in the other direction:**

- Taking into account possible Enemy YELLS (made my bot pro-actively suicide into wanderers rather than be stunned for an extra turn)

- Simulating usage of my own Skills past the initial turn (additional branching factor notwithstanding, unsure other explorers will be in a position for me to effectively use PLAN/YELL --> heavily reliant on an unreliable enemy prediction)

I implemented this searching beyond depth 1 algo sometime around Wednesday and shot up to top20 where I stayed around ~#12 for most of the contest. This surprised the heck out of me... I mean Python3! Search?! I was getting on average 150-200 executions of **procState()** before running out of time. Debugging the paths explored, I could see that I explore up to 3/4 move depth when I don't get Spooked and sometimes reaching 9 move depth when many paths are pruned (sent to the back of pq) due to me walking into a wanderer or slasher rush. This is by no means an exhaustive exploration, merely expanding the highest-scoring path thus far, so I still miss out on some more optimal path due to time running out.

Wait, what about other Explorers? Yea, I couldn't afford to run a simulation for them as well, so I assumed others run my Silver Heuristics code and used that to simulated enemies instead :confused: This led to my performance changing quite a lot during the last weekend when everyone else in the top fixed their simulation or used a better heuristic and my bot couldn't predict accurately where other explorers would go.

Still, this approach resulted in many nice strategies my bot will use to prolong it's survival, for instance, baiting a slasher into STALKING then turning down an alley in the last second to transition it into STUNNED so I have 6 more turns of relative safety. Or baiting other explorers into following me when a slasher targets me then turning out of LoS in the last moment and landing the slasher on them instead. Pretty cool watching it 'discover' such paths on its own without me coding explicit heuristics for such behavior :slight_smile: 

Slashers were I feel, a fresher kind of minion than the classic baddie chasing you around.

##Contest Analysis

I wasn't around for Hypersonic (and for most of MeanMax) so this was my first contest other than 1v1, and boy was it fun :smiley: Loved the competitive co-operation aspect of the game but yes, this introduced much randomness to both games and ladder scoring. Many games are lost by a wrong turn (away from everyone else) at a junction and this was pretty frustrating to deal with... The increased number of players also meant an increase in time taken for games to be played and towards the end it was taking ~1hr for 50% games on submits, meaning I couldn't test my code as much as I liked to.

Once again thanks to CodinGame for hosting another great contest and our Community Creators for taking their time to craft this wonderful game! Loved the subtle graphics too (color changes, idle animations!).