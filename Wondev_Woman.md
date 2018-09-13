## [Contest Multiplayer Link](https://www.codingame.com/multiplayer/bot-programming/wondev-woman)

# Legend #46

I'm a bit late to the party...got carried away making graphics for illustration, ended up writing as much code to generate the graphics as I did during contest time :sweat_smile: Messed around with python turtle graphics library and got some interactive simulation for Wondev working :stuck_out_tongue:  One in which you play as the enemy! \o/ 

Still figuring out how to isolate it to read/write to other processes tho...the version I have now has my bot's python code embedded inside it XD I will however, upload it once it's ready :slight_smile: 

# Wood -> Gold
Only had time to work on this contest during the weekends this time round...almost didn't make it to legend >.< 
The [~200 lines of python](https://github.com/devYaoYH/WdevW/blob/master/woodHeuristics.py) I rushed out on the first weekend surprisingly got me from Wood I to Gold :astonished:

## Unsophisticated but somewhat effective
Some observations I made watching replays:

#### MOVE&BUILD
- Try to climb higher
- Prefer move to 3
- Build higher
- Don't waste time building higher than you can climb to
  - E.g. On 1, building 2 -> 3 would incur a penalty
- When moving from 3 -> 3, build lower (maximize time spent jumping around at height 3)
- When I have to build a 3 -> 4, I pick the one that gives maximum floodfill area
  - That is: max accessible area when I run a BFS on the resulting state (allowing my bot to play a decent end-game)

#### PUSH&BUILD
- Push enemy lower

My early focus was on point-scoring and MOVE&BUILD action, not paying much attention to PUSH&BUILD. Even so, by scoring moves according to the above naive rules, my bot climbed straight from Wood to Gold league...

# Gold -> Legend
Climbing to legend required some more sophisticated code haha :stuck_out_tongue:  Tried my hand coding a minimax for this but by the end of Saturday, I was so swamped with bugs, I made the switch back to my heuristic code from the previous weekend.

Made the shift from point-scoring to area control like many others and finally reached Legend on Sunday evening :smiley:  with 2 major improvements to my code.

## Enemy Tracking
After some debugging whilst generating the graphics, I now hesitate to call it 'enemy tracking'...more like scoring for 'best' enemy position.

The approach my bot took to tracking enemies was simply to overlay possible moves (from previous turn's predictions) against cells from which the enemy is able to build at the detected grid.

<img src="https://forum.codingame.com/uploads/default/original/3X/7/9/79301e7ebd57289cfc75baaf3c8f5d30b23b86d8.PNG" width="519" height="263">

So in the above image, enemy was at (3, 6) then moved to (2, 7) and build at (1, 7). Pardon the turtle resting in the corner, she's tired from drawing all those squares...

- **Green** are the cells the enemy needs to be to build at (1, 7).
- **Red** are possible positions the enemy could've moved to from possible locations tracked the previous turn.
- **Yellow** are the overlapping cells (predicted enemy locations this turn).
- **Darker shades** of yellow and red are previous turn's predictions.

It becomes obvious that this won't work after a few moves where all 8 adjacent cells around the build location could've been moved to from the previous turn's guessed locations. However, with some filtering and cases of enemy pushes, the possibilities can be narrowed down.

- When (-1, -1) is read for enemy position, it provides information on where the enemy **is not**. Namely, the cells adjacent to your units.
- When pushed, the enemy could be in 1 of 3 locations that gives the valid push-angle.

So with these two additional methods of filtering in conjunction with the above 'tracking', my bot was able to decrease the number of possible enemy locations. Using this set of possible locations, I then did a positional scoring on each of them and picked the top as the guessed enemy location to be used for the rest of my turn.

## Voronoi Diagram
The secret to area control and why we want to track enemy positions. For each move, I generate a voronoi diagram on the resultant state and a large portion of the score for that move is based on the ownership of the cells.

<img src="https://forum.codingame.com/uploads/default/original/3X/e/8/e8d56febb3479bf20cd5ae36de295073d4223e0d.PNG" width="519" height="263">

In this example, **Red** is where my enemy is, **Green** is where my unit is. Based on these locations, I run a stepwise BFS alternatively on my units first, then the enemy's, taking into account the validity of each move (cannot climb higher than 1, cannot move onto other units). I then split the cells into 3 categories, *self_control* -> green, *en_control* -> red, *contested* -> yellow.

*Contested* cells are cells that take the same number of moves to get to by either player. Even though I can reach these cells first (you always make the first move), I scored them lower than cells in *self_control* since I could be pushed from these cells by the adjacent enemy.

To make my bot more aggressive, I gave a bigger penalty to enemy controlled cells, so it will try to cordon off the enemy and trap them. With this addition into my scoring function, my bot finally made the jump from Gold into Legend :slight_smile: 

# Thoughts

First off, congratulations to the top 3!!! The score difference in this contest is drastic...even though minimax was the way to go, seems like they nailed the implementation details and scoring functions just right :smiley:

Thanks to CodinGame team for another great contest! This contest was more to the likes of Code4Life, maybe intentionally so based on positive feedback? :slight_smile:  Nonetheless, the boardgame-like nature of this contest is at least similar to that in c4l.

Looking forward to when multiplayer comes out, would be good to properly implement minimax and enemy tracking on this :stuck_out_tongue: