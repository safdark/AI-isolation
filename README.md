<!--- Adding links to various images used in this document --->
[StartGame]: https://github.com/safdark/isolation/blob/master/docs/images/isolation2.png
[MidGame]: https://github.com/safdark/isolation/blob/master/docs/images/isolation4.png
[EndGameOWins]: https://github.com/safdark/isolation/blob/master/docs/images/isolation1.png
[EndGameXWins]: https://github.com/safdark/isolation/blob/master/docs/images/isolation5.png

![EndGameXWins]

# Isolation - Game Playing Agent

## Overview

Isolation is a deterministic, two-player game in which the players alternate turns moving a single piece from one cell to another, on a board.  Whenever either player occupies a cell, that cell becomes blocked for the remainder of the game.  The first player with no remaining legal moves loses, and the opponent is declared the winner.

This project is an attempt at building a game playing 'adversarial search' agent for Isolation combining different strategies. Concepts covered in this project include:
- Building a game tree
- Minimax search
- Alphabeta search (optimization of minimax)
- Iterative deepening (time-limited)
- Quiessance search
- Evaluation functions

This project uses a version of Isolation where each agent is restricted to L-shaped movements (like a knight in chess) on a rectangular grid (like a chess or checkerboard).  The agents can move to any open cell on the board that is 2-rows and 1-column or 2-columns and 1-row away from their current position on the board. Movements are blocked at the edges of the board (the board does not wrap around), however, the player can "jump" blocked or occupied spaces (just like a knight in chess).

Additionally, agents have a fixed time limit each turn to search for the best move and respond. If the time limit expires during a player's turn, that player forfeits the match, and the opponent wins.

## Design

The project is further divided into components, discussed below.

### Isolation Board

This component manages the state of the board game, which includes the location of players on the board, the positions that have already been played, the positions that each player can play, and the winning/losing player at end game. It has no logic built into it except the ability to determine the set of possible moves for the active player on each turn. The choice of the move however is delegated to the players, which is where the 'intelligence' lies.

### Visualizer

This component provides a visualization of the Isolation board, given the sequence of moves that were generated by the players. It is at present only an after-the-fact visualiation, so the entire history of moves for the game (start to finish) must be known before invoking the visualizer. At a later point, it is my hope to change that so that the visualizer is more interactive, providing the opportunity for even a human player to get involved in playing the game.

Here's an example of the visualizer being used to display various stages of the game.

Start of the game:
![StartGame]

End of the game:
![EndGameXWins]

### Game Playing Agents

These are the components, as mentioned above, wherein the game playing intelligence lies. Different game playing agents can exist, each implementing a different game playing strategy/algorithm.

The entry point into the game playing intelligence is through the 'get_moves()' method:

```
    def get_move(self, game, legal_moves, time_left):
        """Searches for the best move from the available legal moves and returns a
        result before the time limit expires.

        This function must perform iterative deepening if self.iterative=True,
        and it must use the search method (minimax or alphabeta) corresponding
        to the self.method value.

        **********************************************************************
        NOTE: If time_left < 0 when this function returns, the agent will
              forfeit the game due to timeout. The agent must return _before_ the
              timer reaches 0.
        **********************************************************************

        Parameters
        ----------
        game : `isolation.Board`
            An instance of `isolation.Board` encoding the current state of the
            game (e.g., player locations and blocked cells).

        legal_moves : list<(int, int)>
            A list containing legal moves. Moves are encoded as tuples of pairs
            of ints defining the next (row, col) for the agent to occupy.

        time_left : callable
            A function that returns the number of milliseconds left in the
            current turn. Returning with any less than 0 ms remaining forfeits
            the game.

        Returns
        -------
        (int, int)
            Board coordinates corresponding to a legal move; may return
            (-1, -1) if there are no available legal moves.
        """
```

#### Adversarial Search Agent

This game playing agent uses adversarial search to determine the next best move to return when its get_move() api is invoked. Various aspects of adversarial search are discussed below.

##### Game Tree Traversal

Adversarial search involves building a game tree for the current state of the board, given the available next moves, and evaluating each subtree (branch) recursively while 'simulating' the progress of the game for each of the move options provided, until a configurable depth has been reached.

There are two traversal mechanisms that have been implemented.

###### Minimax Traversal

With minimax, at each node, the recursive operation performed is as follows:

```
def minimax(self, game, depth, maximizing_player=True):
    ...
    ...
    ...
    legal_moves = game.get_legal_moves(game.active_player)
    if legal_moves is not None and len(legal_moves)>0:
        if depth>0: # Recursive case:
            if maximizing_player:   # MAXIMIZING ply
                score, move = None, None
                for i,m in enumerate(legal_moves):
                    newscore, _ = self.minimax(game.forecast_move(m), depth-1, maximizing_player=not maximizing_player)
                    if score is None or newscore > score:
                        score, move = newscore, m
            else:                   # MINIMIZING ply
                score, move = None, None
                for i,m in enumerate(legal_moves):
                    newscore, _ = self.minimax(game.forecast_move(m), depth-1, maximizing_player=not maximizing_player)
                    if score is None or newscore < score:
                        score, move = newscore, m
        else: # Base case (depth==0)
            score, move = self.score(game, self), None
    else:  # We are at a DEAD-END here
        score, move = self.score(game, self), (-1, -1)

    return score, move
```

###### Alphabeta traversal

Alpha-beta traversal is an optimization of minimax wherein branches are only explored if they contribute some useful information to the score determination. Branches that yield no useful information are pruned out, thereby avoiding unnecessary traversal overhead. This pruning is achieved by continuously updating an acceptable range for the game tree. Any branches that will yield a score that is outside this range are discarded. The algorithm proceeds just like minimax, with the following added logic:

```
def alphabeta(self, game, depth, alpha=float("-inf"), beta=float("inf"), maximizing_player=True, tab='\t'):
    ...
    ...
    ...
    floor = alpha
    ceiling = beta
    legal_moves = game.get_legal_moves(game.active_player)
    if legal_moves is not None and len(legal_moves)>0:
        if depth>0: # Recursive case:
            if maximizing_player:   # MAXIMIZING ply
                score, move = None, None
                for i,m in enumerate(legal_moves):
                    newscore, _ = self.alphabeta(game.forecast_move(m), depth-1, floor, ceiling, maximizing_player=not maximizing_player)
                    if score is None or newscore > score:
                        score, move = newscore, m

                    # Alphabeta bookkeeping:
                    if score > floor:
                        floor = score   # Constrains children at the next (minimizing) layer to be above this value
                    if score >= ceiling: # No need to search any more if we've crossed the upper limit at this max layer already
                        break
            else:                   # MINIMIZING ply
#                     print (tab + "MINIMIZING: (({})) {} < score < {}  ||  Moves: {}".format(depth, floor, ceiling, legal_moves))
                score, move = None, None
                for i,m in enumerate(legal_moves):
                    newscore, _ = self.alphabeta(game.forecast_move(m), depth-1, floor, ceiling, maximizing_player=not maximizing_player)
                    if score is None or newscore < score:
                        score, move = newscore, m

                    # Alphabeta bookkeeping:
                    if score < ceiling:
                        ceiling = score   # Constrains children at the next (maximizing) layer to be below this value
                    if score <= floor: # No need to search any more if we've crossed the lower limit at this min layer already
                        break
        else: # Base case (depth==0)
            score, move = self.score(game, self), None
    else: # We are at a DEAD-END here
        score, move = self.score(game, self), (-1, -1)

    return score, move
```

With alpha-beta traversal, we are guaranteed the same outcome as minimax, yet without the overhead of exploring every game tree branch.

##### Iterative Deepening & Quiessance

```
    ...
    results = deque(maxlen=3)
    for depth in range (self.search_depth, 25):
        score, move = self.dosearch(game, depth)
        results.append((score, move))
        if len(results) >=3 and all(x[1] == move for x in results):
            # Achieved quiessance here since last 3 moves recommendations were identical
            break
        if self.time_left() < 2*self.TIMER_THRESHOLD:
            break
```

##### Board Evaluation Functions & Anaysis

Due to the exponential nature of game tree exploration, memory and time constraints can be limiting. It is therefore usually infeasible to explore very deep into the tree on any turn. In this scenario, heuristics to *quantify the advantage that a given board configuration confers to the given player, relative to other board configurations*, can be helpful at approximating the best choice among possible moves for that player.

The effectiveness of the heuristic can be the differentiating factor for winning or losing the game. Different mechanisms have been explored below, with a brief analysis of the performance of each of these heuristics against a set of simpler heuristics. A `tournament.py` script is used to evaluate this effectiveness. The heuristics being evaluated are powered by alphabeta pruning and time-limited iterative deepening with quiessance, and pitted in a round-robin tournament against the simpler heuristics that in turn are powered by minimax search and alphabeta pruning (but no iterative deepening).

The performance of time-limited iterative deepening search is hardware dependent (faster hardware is expected to search deeper than slower hardware in the same amount of time). However, performance gains are still expected (and have been achieved). The script analyses these heuristics against a baseline performance of the improved_score() heuristic (termed 'Advantage score' below) when powered by iterative deepening and alphabeta pruning. The goal, obviously, is to find at least one heuristic to outperform the improved_score() heuristic. This is achieved by the net_advantage_score() heuristic, termed 'Net advantage score' further below.

The tournament opponents are listed below. (All scoring functions are present in the file: scorefunctions.py).

- Random: An agent that randomly chooses a move each turn.
- MM_Null: CustomPlayer agent using fixed-depth minimax search and the null_score heuristic
- MM_Open: CustomPlayer agent using fixed-depth minimax search and the open_move_score heuristic
- MM_Improved: CustomPlayer agent using fixed-depth minimax search and the improved_score heuristic
- AB_Null: CustomPlayer agent using fixed-depth alpha-beta search and the null_score heuristic
- AB_Open: CustomPlayer agent using fixed-depth alpha-beta search and the open_move_score heuristic
- AB_Improved: CustomPlayer agent using fixed-depth alpha-beta search and the improved_score heuristic

###### Win-Lose score (winlose_score() or null_score())

This scoring only looks at the end game -- i.e, whether the player has actually won or lost. Any other game position returns a score of 0. I wouldn't even call this a heuristic function because any useful information here would require the game tree to be expanded all the way to the end.

```
    if game.is_loser(player):
        return float("-inf")

    if game.is_winner(player):
        return float("inf")

    return 0.
```

Even though it is not very informative on its own for evaluating intermediary board configurations, it does provide the information that all other heuristics try to approximate -- i.e, whether the given board configuration is a winning or losing configuration. Therefore, it is utilized as a short-circuiting determination for every board configuration, prior to other heuristics being utilized, as will be visible in the code snippets below.

###### Open-Move score (open_move_score())

This scoring looks at the number of options available to the player in the present board configuration. It is a sufficiently useful metric and achieves a reasonable performance.

```
    score = winlose_score(game, player)
    if -INFINITY < score < INFINITY:
        score += float(len(game.get_legal_moves(player)))
    return score
```

###### Advantage score (improved_score())

This scoring achieves an even better performance than the 'open_move_score' and is used as a baseline when comparing subsequent scoring algorithms listed below. It determines a very naive determination of the relative advantage of the given player, compared to the opposing player, and returns the advantage as the # of additional move options available to the one over the other.

```
    score = winlose_score(game, player)
    if -INFINITY < score < INFINITY:
        own_moves = len(game.get_legal_moves(player))
        opp_moves = len(game.get_legal_moves(game.get_opponent(player)))
        score += float(own_moves - opp_moves)
    return score

```

**Performance**

```
*************************
Evaluating: ID_improved_score
*************************

Playing Matches:
----------
  Match 1: ID_improved_score vs   Random    	Result: 20 to 0
  Match 2: ID_improved_score vs  MM/3_Null  	Result: 19 to 1
  Match 3: ID_improved_score vs  MM/3_Open  	Result: 12 to 8
  Match 4: ID_improved_score vs MM/3_Improved 	Result: 14 to 6
  Match 5: ID_improved_score vs  AB/5_Null  	Result: 15 to 5
  Match 6: ID_improved_score vs  AB/5_Open  	Result: 13 to 7
  Match 7: ID_improved_score vs AB/5_Improved 	Result: 12 to 8


Results:
----------
ID_improved_score     75.00%
```


###### Net Advantage Score (net_advantage_score())

This score, as with the 'advantage score' above, is also based on the difference in number of moves between the opponent and oneself. However, it also takes into account who the active player is. The score is even higher if an advantage is assessed when the player is active (as opposed to when it is inactive), and similarly, even lower if a disadvantage is assessed when the player is inactive.

```
    score = winlose_score(game, player)
    if -INFINITY < score < INFINITY:
        own_moves = game.get_legal_moves(player)
        opp_moves = game.get_legal_moves(game.get_opponent(player))
        if game.active_player == player:
            score += float(len(own_moves) - len(opp_moves))
        else:
            # It' the opponent's turn, so downgrade the advantage a bit
            score += float(len([x for x in own_moves if x not in opp_moves]) - len(opp_moves))
    return score
```

**Performance**

```
*************************
Evaluating: ID_net_advantage_score
*************************

Playing Matches:
----------
  Match 1: ID_net_advantage_score vs   Random    	Result: 20 to 0
  Match 2: ID_net_advantage_score vs  MM/3_Null  	Result: 18 to 2
  Match 3: ID_net_advantage_score vs  MM/3_Open  	Result: 13 to 7
  Match 4: ID_net_advantage_score vs MM/3_Improved 	Result: 17 to 3
  Match 5: ID_net_advantage_score vs  AB/5_Null  	Result: 17 to 3
  Match 6: ID_net_advantage_score vs  AB/5_Open  	Result: 12 to 8
  Match 7: ID_net_advantage_score vs AB/5_Improved 	Result: 13 to 7

Results:
----------
ID_net_advantage_score     78.57%
```
**This approach yields the best performance, among the score functions evaluated.**


###### Net-mobility Score (new_mobility_score())

Another version of the advantage-based scores above is to return a fixed score (+/-1 or +/-2), regardless of the actual magnitude of variation, with 1 reflecting a minor advantage/disadvantage, and 2 reflecting a more significant one. The logic here is that any momentary advantage on the present board configuration will likely not be the same on a subsequent move, which might yield a very different mobility score. So, it assumes that the extent of the any advantage might not be as high as with the previous mechanisms, particularly since the adversarial search will likely settle on the highest scoring (but likely transient) alternative from among other alternatives that are better from a longer-term standpoint. In practice, however, the performance of this heuristic falls short of expectations, as seen below.

```
    score = winlose_score(game, player)
    if -INFINITY < score < INFINITY:
        own_moves = len(game.get_legal_moves(player))
        opp_moves = len(game.get_legal_moves(game.get_opponent(player)))
        if player == game.active_player:
            if own_moves > opp_moves:
                score += 2              # Active player so extra adv
            elif own_moves < opp_moves:
                score -= 1
        else:
            # Having fewer moves is worse if we're not the next player
            if own_moves > opp_moves:
                score += 1
            elif own_moves < opp_moves: # Not active, so worse-off
                score -= 2
    return score
```

**Performance**

```
*************************
Evaluating: ID_net_mobility_score
*************************

Playing Matches:
----------
  Match 1: ID_net_mobility_score vs   Random    	Result: 19 to 1
  Match 2: ID_net_mobility_score vs  MM/3_Null  	Result: 13 to 7
  Match 3: ID_net_mobility_score vs  MM/3_Open  	Result: 13 to 7
  Match 4: ID_net_mobility_score vs MM/3_Improved 	Result: 9 to 11
  Match 5: ID_net_mobility_score vs  AB/5_Null  	Result: 14 to 6
  Match 6: ID_net_mobility_score vs  AB/5_Open  	Result: 14 to 6
  Match 7: ID_net_mobility_score vs AB/5_Improved 	Result: 13 to 7


Results:
----------
ID_net_mobility_score     67.86%
```


###### Distance-from-center Score (accessibility_score)

The scoring here is based on the distance of the player from the center. It favors the player that stays closer to the center. On its own, it is not a robust enough scoring mechanism, but it might find use alongside others during the initial part of the game, when being closer to the center might offer a longer-term advantage. Later in the game, this mechanism could be dropped, as long as its absence is compensated for by a new mechanism, or a pre-existing scoring mechanism that acquires a greater scoring weightage.

```
    score = winlose_score(game, player)
    if -INFINITY < score < INFINITY:
        center = (round(game.height / 2), round(game.width / 2))
        own_location = game.get_player_location(player)
        delta = gethopdistance(own_location, center)
        if delta < round(center[0]/2):
            score += 1
        elif delta > round(center[0]/2):
            score += -1
            
    return score
```

**Performance**

```
*************************
Evaluating: ID_accessibility_score
*************************

Playing Matches:
----------
  Match 1: ID_accessibility_score vs   Random    	Result: 18 to 2
  Match 2: ID_accessibility_score vs  MM/3_Null  	Result: 12 to 8
  Match 3: ID_accessibility_score vs  MM/3_Open  	Result: 12 to 8
  Match 4: ID_accessibility_score vs MM/3_Improved 	Result: 10 to 10
  Match 5: ID_accessibility_score vs  AB/5_Null  	Result: 14 to 6
  Match 6: ID_accessibility_score vs  AB/5_Open  	Result: 6 to 14
  Match 7: ID_accessibility_score vs AB/5_Improved 	Result: 6 to 14


Results:
----------
ID_accessibility_score     55.71%
```
As expected, this does not yield an impressive performance on its own. See composite score sections for improved performance.

###### Offensive-position Score (offensive_score())

This score is based on whether the current player is positioned to consume one of the opposing player's positions on the pending turn. However, if it is the other player's turn, the score outcome is the opposite. Therefore, this function favors an offensive tactic which I was hoping could be helpful in the later parts of the game. However, in general it might not be a robust heuristic on its own, and likely requires another heuristic to be utilized alongside it.

```
    score = winlose_score(game, player)
    if -INFINITY < score < INFINITY:
        own_moves = game.get_legal_moves(player)
        opp_moves = game.get_legal_moves(game.get_opponent(player))
        current_overlap = [x for x in own_moves if x in opp_moves]
        if player == game.active_player:
            if len(current_overlap) > 0:
                score += 1  # We are at an advantage
        else:
            if len(current_overlap) > 0:
                score -= 1  # We are at a disadvantage
    return score
```

**Performance**

```
*************************
Evaluating: ID_offensive_score
*************************

Playing Matches:
----------
  Match 1: ID_offensive_score vs   Random    	Result: 19 to 1
  Match 2: ID_offensive_score vs  MM/3_Null  	Result: 13 to 7
  Match 3: ID_offensive_score vs  MM/3_Open  	Result: 8 to 12
  Match 4: ID_offensive_score vs MM/3_Improved 	Result: 9 to 11
  Match 5: ID_offensive_score vs  AB/5_Null  	Result: 13 to 7
  Match 6: ID_offensive_score vs  AB/5_Open  	Result: 3 to 17
  Match 7: ID_offensive_score vs AB/5_Improved 	Result: 5 to 15


Results:
----------
ID_offensive_score     50.00%
```

As expected, the performance here is not impressive. See composite score sections for improved performance.


###### Proximity score (proximity_score())

This favors keeping the opponent close, and nothing else. As with some of hte other heuristics above, it does not yield good performance on its own and might achieve a better outcome when used in conjunction with other scoring heuristics.

```
    score = winlose_score(game, player)
    if -INFINITY < score < INFINITY:
        own_location = game.get_player_location(player)
        opp_location = game.get_player_location(game.get_opponent(player))
        delta = gethopdistance(own_location, opp_location)
        score -= delta   # Since the knight moves in an L shape (so can jump 2 spaces net)
    return score
```

**Performance**

```
*************************
Evaluating: ID_proximity_score
*************************

Playing Matches:
----------
  Match 1: ID_proximity_score vs   Random    	Result: 20 to 0
  Match 2: ID_proximity_score vs  MM/3_Null  	Result: 17 to 3
  Match 3: ID_proximity_score vs  MM/3_Open  	Result: 9 to 11
  Match 4: ID_proximity_score vs MM/3_Improved 	Result: 10 to 10
  Match 5: ID_proximity_score vs  AB/5_Null  	Result: 12 to 8
  Match 6: ID_proximity_score vs  AB/5_Open  	Result: 9 to 11
  Match 7: ID_proximity_score vs AB/5_Improved 	Result: 10 to 10


Results:
----------
ID_proximity_score     62.14%
```
Note that this competes well with the improved_score() heuristic (10 to 10), and very well against the random player and the winlose_score() heuristic. However, it falls short against the open_moves score heuristic. See composite sections below.


###### Distance-from-open-spaces score (horizon_score())

This scoring is based on the distance of the player from the open spaces. The goal is to encourage the player to always stay closer, or move towards, parts of the board with more open spaces, so as to avoid getting trapped in a region of limited mobility, particularly near the middle or end of the game. It helps potentially mitigate (though not entirely eliminate) the *horizon problem*.

It is still a WIP and has therefore not been listed here. One challenge is the time comlexity required to process islands of open spaces. Taking the center of mass of the open slots might not achieve the desired result, since the objective is to move directly towards larger clusters. The center of mass might not be located at the right spot.

Unless a very efficient approach to this problem emerges, to identify the reachable open-space islands on the board, I have doubts as to whether this heuristic could be useful. Nevertheless, I felt it might be interesting mentioning it here.

**Performance**

TBD

##### Composite Scoring

A scoring function that is a combination of other scoring mechanisms is possible too, but should be implemented carefully to avoid overweighting or underweighting different sub-scores.

The following composite function was attempted

###### Offensive + Keeping-opponent-close + Net-mobility (combo_offensive_nearopponent_netmobility_score())

**Performance**

```
*************************
Evaluating: ID_combo_offensive_nearopponent_netmobility_score
*************************

Playing Matches:
----------
  Match 1: ID_combo_offensive_nearopponent_netmobility_score vs   Random    	Result: 18 to 2
  Match 2: ID_combo_offensive_nearopponent_netmobility_score vs  MM/3_Null  	Result: 16 to 4
  Match 3: ID_combo_offensive_nearopponent_netmobility_score vs  MM/3_Open  	Result: 12 to 8
  Match 4: ID_combo_offensive_nearopponent_netmobility_score vs MM/3_Improved 	Result: 14 to 6
  Match 5: ID_combo_offensive_nearopponent_netmobility_score vs  AB/5_Null  	Result: 16 to 4
  Match 6: ID_combo_offensive_nearopponent_netmobility_score vs  AB/5_Open  	Result: 7 to 13
  Match 7: ID_combo_offensive_nearopponent_netmobility_score vs AB/5_Improved 	Result: 10 to 10


Results:
----------
ID_combo_offensive_nearopponent_netmobility_score     66.43%
```

Not particularly bad, though not very impressive either.

###### Net-advantage + Keeping-opponent-close (combo_netadvantage_nearopponent_score())

**Performance**

```
*************************
Evaluating: ID_combo_netadvantage_nearopponent_score
*************************

Playing Matches:
----------
  Match 1: ID_combo_netadvantage_nearopponent_score vs   Random    	Result: 18 to 2
  Match 2: ID_combo_netadvantage_nearopponent_score vs  MM/3_Null  	Result: 19 to 1
  Match 3: ID_combo_netadvantage_nearopponent_score vs  MM/3_Open  	Result: 12 to 8
  Match 4: ID_combo_netadvantage_nearopponent_score vs MM/3_Improved 	Result: 12 to 8
  Match 5: ID_combo_netadvantage_nearopponent_score vs  AB/5_Null  	Result: 17 to 3
  Match 6: ID_combo_netadvantage_nearopponent_score vs  AB/5_Open  	Result: 12 to 8
  Match 7: ID_combo_netadvantage_nearopponent_score vs AB/5_Improved 	Result: 12 to 8


Results:
----------
ID_combo_netadvantage_nearopponent_score     72.86%
```
This achieves reasonable performance, but needs some score tweaking. WIP.

###### Phased Scoring

Another approach that is worth attempting at a later date is one wherein the scoring heuristic changes as the game progresses. It starts out less offensive, and increases offensiveness as the game progresses.

**Performance**

```
Work-In-Progress

```

