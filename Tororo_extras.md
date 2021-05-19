# Tororo Extras

Broken search strategy, for those interested in not repeating my mistakes...

## Search Strategy (pretty broken...)

Finally, some comments on my search strategy although I think [Euler's implementation](https://forum.codingame.com/t/spring-challenge-2021-feedbacks-strategies/190849/2) of beam search may be more accurate than the version I ended up with which feels a bit wonky now that I look at it more...

Roughly I *think* it is a beam search where I limited the width of progagation at each search layer but for Tororo we needed to consider two 'layers', actions and days. So I just gave priority in a BFS way for least actions in closest day then break ties by state evaluation function. Not comparing the actions taken gave me quite bad results since a COMPLETE action would always certainly be weighted heavier in my heuristic evaluation (but thinking back, maybe prioritizing by more SUN may be better than length of actions?). **BUT** after looking at Euler's post, I'm sure my bot's search is pretty broken... like, really, really, only-simulates-3-actions-deep broken... lol

The tricky part comes in when we are searching within the 'beam' of a day. When should we prune off the remaining actions and begin a new day? I'm not sure if my algo handles this part in a principled manner, but for this I have a smaller 'beam' for each action length that I defer over to the next day by appending a WAIT action on top of it (without calling `generate_actions` to propagate the state, just truncate the action sequence). Otherwise, the beam size for a day constrains how many nodes we expand within each day (number of actions simulated within a day). Actions thus deferred into the next day will be sorted for us when we reach to expand them later (we similarly only expand top-'beam' sorted action sequences in the next layer).

I have a cheaper heuristic state evaluation and a more expensive playout policy to determine priority in queue and final action selection respectively. This allows for increased evals/states visited in exchange for a more approximated solution.

Roughly in python (incomplete sketch):
```python
def simul(init_state, playout=s_playout_static, max_depth=5, beam=100, t_left=0.07):
    init_t = time.time()
    search_layer, last_layer = [], []
    pq, v = [(init_state, 0)], {}
    search_depth = 0
    while(len(pq) > 0 and time.time()-init_t < t_left):
        current_state, cur_depth = heappop(pq)
        if (cur_depth > max_depth):
            break
        if (search_depth > cur_depth):
            # defer this node into next day (if still have budget within small actions 'beam')
            # after adding a WAIT action on top of it
            continue
        if (len(search_layer) >= beam or search_depth < cur_depth): # We stepped into a new day
            last_layer = search_layer   # Swap & store this layer
            search_layer = []
            search_depth += 1
        search_layer.append(current_state)
        for action, state in generate_action(current_state): # Expand this node
            # ... Typical BFS Algo stuff ... #
            heappush(pq, (state, cur_depth+(1 if action == "WAIT" else 0))) # next layer only when WAIT
            v[state] = v[current_state] if current_state in v else action   # store the top-level action
    # Finally, we pick the action from the last layer we completed our search in
    return v[sorted(last_layer, key=lambda s: playout(s), reverse=True)[0]] # highest-scoring
```
*note* Queue and PriorityQueue in python is the thread-safe version and I find heapq faster if we don't require that constrain.

## Mistake...

However, we note that the deferred actions will only come from the `k` and `k+1` actions taken during day `d` where `k` is the **full-width BFS** action depth we break at once we hit our large beam constraint. I think this was my major mistake not noting that it thereby limited me to only a few actions-deep each day... Too wide but not deep enough :/

### Fix (#90 -> top #50)

A relatively simple fix is to restart our search at each day and constrain the beam over each action layer instead. Applying this fix results in a huge performance jump up to top 50 from the roughly top 100 final contest submission...

Fix for the above-described search (once again, just a rough sketch):
```python
def simul(init_state, playout=s_playout_static, max_depth=5, beam=100, t_left=0.07):
    init_t, timeout = time.time(), False
    search_layer, last_layer = [], []
    day_pq, v = [(init_state, 0)], {}
    actions_depth, expanded_nodes, day = 0, 0, init_state.day
    for d in range(day, min(24, day+max_depth)): # Depth limited by DAYS still
        if (timeout):
            break
        action_pq = []  # Restart with a fresh pq for each day
        actions_depth = 0
        # Push in our actions accumulated from previous day
        while(len(day_pq) > 0 and len(action_pq) < beam):
            heappush(action_pq, heappop(day_pq))
        day_pq = []     # Clear remaining nodes (pruned)
        while(len(action_pq) > 0):
            if (time.time() - init_t > t_left):
                timeout = True
                break
            current_state, cur_actions = heappop(action_pq)
            if (actions_depth > cur_actions): # Beam Pruning here over actions instead
                continue
            if (expanded_nodes > beam): # Move onto next layer of actions, skip remaining nodes
                actions_depth += 1
                expanded_nodes = 0
            expanded_nodes += 1
            # ... Generate actions, update visited etc... Same as before #
            # Layers are now for each depth of actions instead of day as before
            # Make sure to push into the correct pq (day_pq on WAIT, action_pq otherwise)
    return v[sorted(last_layer, key=lambda s: playout(s), reverse=True)[0]] # highest-scoring
```

That's it! Can't help but be slightly disappointed I didn't work this out during contest time and lost \~40 positions due to this inappropriate application of the beam over entire DAYS instead of ACTIONS.

*note* We still have comparator for State object sort by actions then tie-break with evaluation function. This allows for the single pq within each day over multiple action depths, otherwise, we would require a new pq each layer which might be faster still since we avoid popping out the excess pruned nodes...