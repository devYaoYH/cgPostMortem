## [Contest Multiplayer Link](https://www.codingame.com/multiplayer/bot-programming/mean-max)

# Mean Physics

Great job to the creators of the first (of many I hope) community contests! :smiley: Really well executed contest with interesting mechanics.

Couldn't spend as much time on this contest as before (on vacation most of the week :stuck_out_tongue:) and only had the weekends to work on my bot resulting in my abysmal ranking haha :upside_down: was a great platform for some physics revision though \o/

After the previous contest, I figured out my bot was actually doing something pretty far from design. So this time round, I decided to devote the first weekend to writing a proper visual tool for me to debug my algos.

### Turtle

Used the default tkInter package in python (turtle graphics library) to draw each vehicles' state which made life much easier breaking down the actions my vehicles took each turn.
 
<img src="https://forum.codingame.com/uploads/default/original/3X/4/1/411c178b707d06b196c0d237b7421c2bb5268a73.gif" width="500" height="500">

Perhaps something like having two simple options to draw circles and lines to the debug mode visual screen be possible for the current game engine? Could be an additional regex pattern that accepts input like 
```
"DEBUG [-circle | -line] [-pos] [-dist]..." 
```
in the referee so our bots could have a visual 'output' esp for physics-heavy games like this where it's pretty hard to imagine the vectors and it's a lot easier to have a visual feedback on what your bot is doing.

## Navigation

Most of my programming time was spent on implementing different navigation strategies for my Reaper which I focused on thinking that it would be the most important factor in a winning strategy:

1. **Collision Avoidance**

    + The nearest obstacle intersecting the line from vehicle to target will exert a force onto the vehicle to push it away from a collision.
    + Got my bot stuck many times alternating between diverting left and right as the obstacle moves...

2. **Bug Pathing**

    + https://www.cs.cmu.edu/~motionplanning/lecture/Chap2-Bug-Alg_howie.pdf
    + Used for my Reaper to go around multiple congregated obstacles
    + Since there isn't an issue of limited vision, I simulated either direction and picked the fastest. (without enemy movement prediction)

3. **Gravity Pathing (For Destroyer)**

    + Each body within a certain effective range exerts a 'gravitational force' on my Destroyer, empty Tankers repel, full ones attract, enemy Reapers attract, own Reaper repels etc...
    + Sum all forces to get resultant vector (current turn's thrust)
    + Worked somewhat but needs a lot more tuning with how the state of each vehicle changes it's force on the Destroyer.

Even with these pathing strategies, they were not as important to the overall performance of the bot. It became clear in Gold League that the complexities introduced by the many collisions and skills quickly made such an approach untenable. Even with a strong navigation algo (goes to wrecks fast and manages to brake in time to stop) won't ensure your bot will win. Many times, the optimal pathing isn't followed due to enemy collisions, grenades and oils messing up your prediction.

## Deciding Wrecks

Moving from a distance-based selection to a time-based selection (simulate Reaper pathing with no collisions and sorting based on the number of turns instead of distance to wreck) was sufficient to go from Silver to Gold, thereafter I stalled :/ 

Tried improving the selection of wrecks by taking into account vehicles present on and around wrecks, hence blocking it, as well as nearby doofs with the potential to block with oil. Neither of these strategies gave that much improvement to the bot's overall performance and instead ended up with a lower ranking.

One thing that improved my bot's performance slightly was the addition of 'simulated' wrecks. I scanned through the current wrecks and added wrecks at the intersection of overlapping wrecks with the sum of overlapping wrecks' water. This allowed my Reaper to treat this as just another wreck to consider going to in it's selection algo; allowing it to target collecting from multiple wrecks at once.

## Deciding Oil Spills

Only added in heuristics for using skills on the last day of contest. Oil target selection was done by running my Reaper's wreck selection algo for each enemy and making my Doof to go towards this target (Oil it when within range).

Haven't gotten to tars/grenades before the time ran out for contest :(

## Multiplayer!
Navigation turned out to still be slightly buggy (hurhur) and my Destroyer simply spam grenades at wrecks within range with nearby enemies (very sophisticated heuristics). Doof also isn't aggressive enough towards enemy Reapers denying them from collection water on wrecks.

There's still quite some work cut out for me to make my bot competitive enough to be promoted into Legend.

## Game Design

One thing I loved in this game is the 1v1v1 mechanic, first 3p multiplayer contest I've participated in (GitC does seem suitable to be extended into 3 players). Too much complexity, however, resulted in the demise of heuristic bots in this contest :confused:. 

The state of the game cannot be easily predicted without running costly simulations, hence limiting the amount of 'metrics'? a heuristic bot can take into account. Overall the game isn't suited for heuristics to prevail in the higher leagues.

Still this is an AI-programming competition and I think this just pushes me to properly implement search-based techniques (more widely applicable) instead of relying on heuristics.

:thumbsup: Thanks again Agade, pb4, Magus, reCurse for the amazing contest! Can't wait to see what comes of the next one :slight_smile: 

####P.S. *Some blatant advertising* :stuck_out_tongue: 

The annual [battlecode](https://www.battlecode.org) competition will be running in January next year. It's a RTS-styled bot programming tournament (think programming a StarCraft II AI player). Feel free to join me while we wait for the next contest :smiley: hehe.