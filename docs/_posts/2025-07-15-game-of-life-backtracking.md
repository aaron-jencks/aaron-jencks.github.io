---
layout: post
title:  "Game of Life Backtracking"
date:   2025-07-15 16:45:00 -0400
categories: jekyll update
---

# Project Overview

Conway's Game of Life is a cellular automation simulation that has a really simple system. It consists of a grid, and cells, each cell in the grid can either be alive or dead. There is a series of simple rules to determine if cells are alive or dead:
1. If a cell is alive and has no neighbors, it dies due to starvation
2. If a cell is alive and has 2-3 neighbors it survives
3. If a cell is alive and has >4 neighbors it dies as if by overpopulation
4. If a cell is dead and has exactly 3 neighbors, it becomes alive as if by reproduction
   
I had an idea roughly 2 months ago for a simulation for my girlfriend. If I could encode some kind of message created using stable patterns in Conway's Game of Life and then backtrack, I could make a cool looking transition with a cute final reveal. This was prior to learning that going backwards in the Game of Life is an NP hard problem. But here's what I tried anyway.

# Methodology

Implementing Conway's Game of Life is trivial. Adding ANSI escape sequences to do some interesting terminal control is interesting, but not that difficult. This is how the forward simulation works. Using C, I didn't see the need to implement multithreading. I added a utility to loading the initial state from a file, the size of the terminal would be queried and then that initial pattern would be centered onto the terminal.

To implement the reverse traversal a friend of mine told me about the Z3 constraint solver. I tried a couple of iterations. My first thought was to solve one grid at a time, start at the end, find a previous grid, then find the next previous and repeat until the desired number of frames is reached. I learned after one step that something called The Garden of Eden exists, which is a grid state of The Game of Life to which no parent exists. Since each grid has multiple parents, though, this meant that the solver might just land on a single grid that didn't have parents. My solution to this would be instead to solve for multiple frames at once.

Setting up the constraints essentially consisted of using the Conjunctive Normal Form of the state of each cell given it's neighbors. Essentially, if this state needs to be alive in the next state, here's the constraints for all of its neighbors in the previous step, otherwise if needs to be dead, here's the constraints in that case. You can chain this and just set the constants in the final state to be what they need to be, and then set up constraints for each preceding grid.

I learned that the solving time scales exponentially, so I created a simple script to characterize this. The results I present below show my findings.

# Results

| Grid Size | 1 Frame | 2 Frames | 3 Frames | 4 Frames |
| --------- | ------- | -------- | -------- | -------- |
| 15        | 0.83    | 6.32     | 84.887   | 748.842  |
| 20        | 1.579   | 6.817    | 60.417   | 646.375  |
| 50        | 23.479  | 54.895   | 7299.086 | DNF      |
| 100       | 70.023  | DNF      | DNF      | DNF      |
| 200       | 195.215 | DNF      | DNF      | DNF      |

The target was to predict the solving time for a sample grid of size (250, 61) for 10 frames. To do this we solve using log-linear regression in 2 phases. We also consider the area of the grid being solved instead of the grid size, since testing was only done on square grids. The first phase of regression, we solve for solving time of the (250, 61) size grid for 1, 2, and 3 frames. We excluded 4 frames, because there isn't enough data to show an exponential curve. This results in:

| 1 Frame | 2 Frames  | 3 Frames   |
| ------- | --------- | ---------- |
| 22.0136 | 1.25837e7 | 2.81198e15 |

Predicted solving time for a grid of area 15,250 (250, 61). We then fit another exponential regression to solve for 10 frames, we find this to be `y = 7.203e-07 * e^(1.624e+01 * x)`. This results in a predicted solving time of 2.44975e64 seconds or 7.7681e56 years

![solving time](/assets/images/gol-solving-time.png)

# Conclusion

To solve the grid that I set out for, I've determined that it was infeasible to continue. :(
