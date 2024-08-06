---
weight: 1
title: "HTB cyber-apocalypse-2024 - Path of Survival"
date: 2024-01-29T16:37:00+06:00
lastmod: 2024-01-29T16:37:00+06:00
draft: false
author: "HostileNinja72"
authorLink: "https://HostileNinja72.github.io"
description: "Writeup for Path of Survival Misc Challenge."

tags: ["misc", "HTB", "algorithms", "djikstra", "hard"]
categories: ["Writeups"]

lightgallery: true

math:
  enable: true

toc:
  enable: true
---

Writeup for Path of Survival challenge from HTB cyber apocalypse CTF 2024.



<!--more-->

## Initial Analysis

Upon starting the instance, we get an ip and a port. As we connect, we see as it appears to be a game.

<kbd> <img src="https://github.com/hostileninja72/hostileninja72.github.io/blob/main/content/posts/htb/initial_view.png?raw=true"
 style="border-radius:2%" align="center" width = "100%" /> </kbd>

We can see in the game, multiple squares, each one with a specific image, defining its nature in the game. There is also a player icon. To further understand, we have access to rules.

Reading the rules, it is indeed a game, where we need to travel between tiles to get to a weapon tile, before the time runs out.

- **Empty** tiles cannot be accessed.
- **Cliffs** can only be entered from top and the left.
- **Geysers** can only be entered from bottom and the right.

We also have been given the **time costs**:

- A movement to or from a **Cliff** or **Geyser** costs 1 time point.
- A travel from a tile of one terrain to another tile of the same terrain cost 1 time point.
- Plains to Mountain: 5
- Mountain to Plains: 2
- Plains to Sand: 2
- Sand to Plains: 2
- Plains to River: 5
- River to Plains: 5
- Mountain to Sand: 5
- Sand to Mountain: 7
- Mountain to River: 8
- River to Mountain: 10
- Sand to River: 8
- River to Sand: 6

Noting we must win 100 times in a row, and given we have an API page `api` describing how to comm with the app, the solution is to script the process

## Solution

This is a path finding challenge, so we must implement a convenable algorithm.

The choices for me were between `A*` algorithm and `djikstra's` algorithm.

I chose to use `A*` as it is specifically designed for finding the shortest path from a single source node to a single target node. Also to further explore this algorithm as i am used to **djikstra** in my network classes.


### Setting Up

I started by setting up a cost dictionary 

```python 
costs = {
    ('P', 'M'): 5,
    ('M', 'P'): 2,
    ('P', 'S'): 2,
    ('S', 'P'): 2,
    ('P', 'R'): 5,
    ('R', 'P'): 5,
    ('M', 'S'): 5,
    ('S', 'M'): 7,
    ('M', 'R'): 8,
    ('R', 'M'): 10,
    ('S', 'R'): 8,
    ('R', 'S'): 6
}``` 

next we should understand the format of the api response.
Upon sending a post request we get the following:

```python
{'height': 10, 'player': {'position': [10, 8], 'time': 21}, 'tiles': {'(0, 0)': {'has_weapon': False, 'terrain': 'P'}, '(0, 1)': {'has_weapon': False, 'terrain': 'P'} [...]}, 'width': 15}
```

So we are given `height` and `width`, a `player` key that holds a `position` and a `time`. And if the end `tiles` keys wich is a dictionary of `position: {has_weapon, terrain}` values.

Giving this, a knowing that our desired `A*` algorithm operates on graphs, we define a function `create_graph` that will create a graph from the receiving data.

```python
def create_graph(data):
    graph = {}
    for key, value in data.items():
        x, y = eval(key)
        graph[(x, y)] = value
    return graph
```




