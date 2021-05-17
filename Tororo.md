## [Contest Multiplayer Link - Pending...](https://www.codingame.com/multiplayer/bot-programming/spring-challenge-2021)

Cheers once again for the great contest CG team & fellow Codingamers!

It's been a while since I've devoted as much time to a codingame contest and written a PM :P Looks like I'll finish around **#90-100** in Legend. I quite liked the simplified design of the game and appreciate the discrete state space. I'll just refer to this contest as Tororo, it sounds better that way. Also, kudos to reCurse for an absolutely dominating performance! Looking forward to your PM. *Edit: really interesting PMs all around! Love it :) thanks for sharing!*

Some of you may associate me with a particular high-level interpreted language I like to contest in, but sometime in the middle of the contest I decided python just wasn't fast enough to simulate enough moves into the future for a search-based strategy to work decently well (squeezed out \~1K simulations with evals). I also haven't been programming much over the past year so the additional C++ practice was much welcomed.

Even though it was easier to translate over to C++ with the python bot as reference, it still took me a couple days to get it working. Once that was done, I managed to beat the Gold Boss just by increasing the search breadth/depth (\~50-60K simulations with evals). Below I'll briefly describe a couple things I learnt during this contest.

## Bitboarding

Tororo's game state quite intuitively translated over to a bitwise notation where the presence of a tree is just 0/1 on the i-th bit. This means we can represent any state as the following (depth & actions are for book-keeping during search):
```c++
struct State{
    uint64_t seed;          // | first 37 bits CELL has SEED        | --11 bits free-- |       16 bits Depth       |
    uint64_t sm;            // | first 37 bits CELL has size 1 TREE | --11 bits free-- |       16 bits Actions     |
    uint64_t md;            // | first 37 bits CELL has size 2 TREE | -----------------27 bits free----------------|
    uint64_t ta;            // | first 37 bits CELL has size 3 TREE | -----------------27 bits free----------------|
    uint64_t me;            // | first 37 bits CELL has OWN tree    | --11 bits free-- | 8 bits SCORE | 8 bits SUN |
    uint64_t opp;           // | first 37 bits CELL has OPP tree    | --11 bits free-- | 8 bits SCORE | 8 bits SUN |
    uint64_t dormant;       // | first 37 bits CELL is DORMANT      | --11 bits free-- | 8 bits NUTRI | 8 bits DAY |
};
```

With this, we can shorten some of our operations into a few bitwise operators which should speed things up significantly compared to storing and iterating over a size-37 (or more) array/vector:

```c++
// State-based macros
/* Patterns:
 * #define TREES 0x1FFFFFFFFFUL // First 37 bits
 * #define i64 ((unit64_t)1)
 *  get_(me/opp)_trees          : s.(seed/sm/md/ta)&TREES&s.(me/opp)                //Order does not matter
 *  get_(me/opp)_active_trees   : (~s.dormant)&s.(me/opp)&TREES                     //mask with inverted dormant bits
 *  cnt_(me/opp)_trees          : cntBits(s.(seed/sm/md/ta)&TREES&s.(me/opp))       //function call + first macro pattern
 *  can_seed_locations          : ((~(s.seed|s.sm|s.md|s.ta|SOIL_ZERO))&TREES)      //merge the first 4 bitmasks and places
 *                                                                                  //where SOIL is 0, invert them and
 *                                                                                  //mask again with TREES
 *  accumulating shadows        : if (state.sm&idx) // idx = (i64<<i)
 *                                    shadow.sm |= SHADOW_CELLS[d][i][1];           //For day d, tree i and size 1
 */ 
```
*`cntBits`* I adapted the solution from https://stackoverflow.com/a/109025 to do this in constant time.

### Some Shadowing/Seeding tricks
```
// bitmask of adjacent cells for idx i at length l [i][l]
const u64 ADJ_CELLS[37][4]; // [0][0]: {0UL,126UL,524286UL,137438953470UL},
// bitmask of shadowed cells on day d for idx i at length l [d][i][l]
const int SHADOW_CELLS[6][37][4]; // [0][0]: {0UL,2UL,130UL,524418UL},
```

First we can just precompute all neighboring cells for each locations at a given distance with a 37-bit bitmask. Similarly, we can do so for any cell on a given (day%6) where the shadow would be cast. So to check whether we can seed to a location, we need only require a check for whether our tree bits intersect with the neighbors bitmask of the tree height distance around the seed_to location (an inverse check as I iterate through possible seed locations instead of own trees). However, iterating through 0-36 for which index to seed_from still took my seeding actions generation to O(n^2) :(. For shadows, we can combine all the bitmasks together in one iteration over the tree indices to get the overall shadow 'map' for the day.

## Zobrist Hashing

Another trick was [Zobrist Hashing](https://en.wikipedia.org/wiki/Zobrist_hashing) which reduced the collisions for my hashmap and gave me a boost of \~5K more simulations (10%) even though it was more expensive to compute than the prime-multiply and add naive strategy used before. The Zobrist hashing table is statically pre-populated with random 64-bit unsigned long long integers beforehand. *All in all, around 1/4 of my codebase was constants o.O*

The general strategy here for Zobrist hashing is to xor each component of the board state with each other. Each cell can be in one of 9 unique states {empty,(seed,small,medium,tall)\*(me,opp)}, in addition to a limit of 24 days, 21 nutrients and I assume a max of 255 possible sun/score. Chuck them all into an xor and we get back our collision-reduced hash.

## Optimization tricks

As per Magus https://www.codingame.com/forum/t/c-and-the-o3-compilation-flag/1670 and \_CPC\_Herbert https://forum.codingame.com/t/c-and-the-o3-compilation-flag/1670/15.

1) `#pragma GCC optimize "O3,inline,omit-frame-pointer"` at the top of your file!
    - Declare functions explicitly as inline.
2) Reuse global variables
    - `struct State[]` for the search and `struct Shadow[]` for computing shadow maps. To recover from an early-exit scenario (timeout when searching a layer), I keep pair of State arrays and just alternate between them by indexing with `depth%2`. So when a layer is incomplete, we can revisit the old completed layer to evaluate those nodes instead.
3) `std` is slow
    - Unfortunately I did not implement my own heapq or hashmap, but the only two `std` datastructures I really used was a `std::priority_queue` and `std::unordered_map` in the search. It was faster to declare the pq as a function-local variable while reusing a global map and calling `.clear()` (heap log(n) removals I presume - but is there no way to wipe it in O(1)?).

## Search Strategy (pretty broken...)

Finally, some comments on my search strategy although I think [Euler's implementation](https://forum.codingame.com/t/spring-challenge-2021-feedbacks-strategies/190849/2) of beam search is far more accurate than the version I ended up with which definitely feels a bit wonky now that I look at it more...

Roughly I *thought* it was a beam search where I limited the width of propagation at each search layer but for Tororo we needed to consider two 'layers', actions and days. So I just gave priority in a BFS way for least actions in closest day then break ties by state evaluation function. Not ordering by the number of actions taken gave me quite bad results since a sequence with COMPLETE would always certainly be weighted heavier in my heuristic evaluation (thinking back, maybe prioritizing by more SUN may be better than length of actions?). **BUT** after looking at Euler's post, I'm sure my bot's search is pretty broken... like, really, really, only-simulates-3-actions-deep-a-day broken... lol, so I shall skip describing it further... (see [extras](/Tororo_extras.md))

Only thing of note here was that I had a cheaper heuristic state evaluation (which I cached as a 'score' field in my State) to determine priority in queue and a more expensive playout policy for the final layer to select the best action. This allowed for increased evals/states visited in exchange for a more approximated solution.

### Heuristic Evaluations

My heuristic evaluation was a cheap next-day sun income and score (adding bonus for shading opponent trees worked worse...) + a small weighted bonus for SOIL score. The final playout function uses a naive static WAIT policy... which works... surprisingly alright:

1) Wait each turn to generate income. Compute shadow map. Since we're always WAIT-ing, we can optimize slightly by unrolling the loop to only iterate through at most 6 days.
2) We have a multiplicative decay of \*0.95 for each day on sun income.
3) Pessimistically assume nutrients will decrease by `opponent_tall_trees` when we harvest our trees. These additions were so that my bot didn't hold onto trees for too long and prunes the board earlier.
4) Add a bonus score for seeds that are in various levels of shade/no-shade. Afterall, an actual MCTS playout would have us grow these trees and produce some additional sun each turn.
5) Return the simulated final score based on game rules + `num_trees/100` (tie-breaker).

## Remarks

Thanks again for the wonderful contest! Hopefully the things I learnt and shared about above may be helpful to some, definitely switched me back into programming mode from reading/writing philosophy all semester... Looking forward to the Fall Contest!