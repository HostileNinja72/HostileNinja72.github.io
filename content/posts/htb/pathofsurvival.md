---
weight: 1
title: "HTB Cyber Apocalypse 2024 - Path of Survival"
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

# Initial Analysis

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

# Solution

This is a path finding challenge, so we must implement a convenable algorithm.

The choices for me were between `A*` algorithm and `djikstra's` algorithm.

I chose to use `A*` as it is specifically designed for finding the shortest path from a single source node to a single target node. Also to further explore this algorithm as i am used to **djikstra** in my network classes.


## Setting Up

### Building the code
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
}
``` 

next we should understand the format of the api response.
Upon sending a post request we get the following:

```python
{'height': 10, 'player': {'position': [10, 8], 'time': 21}, 'tiles': {'(0, 0)': {'has_weapon': False, 'terrain': 'P'}, '(0, 1)': {'has_weapon': False, 'terrain': 'P'} [...]}, 'width': 15}
```

So we are given `height` and `width`, a `player` key that holds a `position` and a `time`. And if the end `tiles` keys wich is a dictionary of `position: {has_weapon, terrain}` values.

Giving this, and knowing that our desired `A*` algorithm operates on graphs, we define a function `create_graph` that will create a graph from the received data.

```python
def create_graph(data):
    graph = {}
    for key, value in data.items():
        x, y = eval(key)
        graph[(x, y)] = value
    return graph
```

### Pathfinding
Now we need to implement the [`A*`](https://en.wikipedia.org/wiki/A*_search_algorithm) 

The **A* (A-star) algorithm** is a popular and efficient pathfinding and graph traversal algorithm used to find the shortest path between two nodes. It is widely used in various applications such as game development, robotics, and geographical information systems.

It combines between `djikstra` and `Greedy Best-First Search`: Guaranteeing the shortest path with the efficiency of GBFS (heuristic function).

Cost Function:
`A*` uses  a cost function $f(n)$ for each node $n$, which is defined as:

$$
\begin{aligned}
f(n) &= g(n) + h(n) \\
\end{aligned}
$$

Now with the implementation:
We define first a function `get_cost`, that we will need in the `a_star` function.

```python 
def get_cost(current, neighbor, graph):
    current_terrain = graph[current]['terrain']
    neighbor_terrain = graph[neighbor]['terrain']

    if current_terrain == 'E' or neighbor_terrain == 'E':
        return float('inf')  

    if neighbor_terrain == 'C' and (neighbor[0] == current[0] - 1 or neighbor[1] == current[1] - 1):
        return float('inf') 

    if neighbor_terrain == 'G' and  (neighbor[0] == current[0] + 1 or neighbor[1] == current[1] + 1):
        return float('inf')  

    return costs.get((current_terrain, neighbor_terrain), 1)
```

In this function we define the constraints giving in rules.

A star code:

```python
def a_star(graph, start, goal, h):
    open_set = set([start])
    closed_set = set()
    came_from = {}
    g_score = {start: 0}
    f_score = {start: h(start, goal)}

    while open_set:
        current = min(open_set, key=lambda x: f_score.get(x, float('inf')))
        if current == goal:
            path = [current]
            while current in came_from:
                current = came_from[current]
                path.append(current)
            return path[::-1], g_score[goal]

        open_set.remove(current)
        closed_set.add(current)

        for direction in [(0, 1), (1, 0), (0, -1), (-1, 0)]:
            neighbor = (current[0] + direction[0], current[1] + direction[1])
            if neighbor in closed_set or neighbor not in graph:
                continue
            
            tentative_g_score = g_score[current] + get_cost(current, neighbor, graph)
            if neighbor not in open_set or tentative_g_score < g_score.get(neighbor, float('inf')):
                came_from[neighbor] = current
                g_score[neighbor] = tentative_g_score
                f_score[neighbor] = tentative_g_score + h(neighbor, goal)
                if neighbor not in open_set:
                    open_set.add(neighbor)

    return None, float('inf')
```
### Crafting the main Code

The main solution code:

```python 
import requests
from tqdm import tqdm
from functions import a_star, get_directions_sequence, manhattan

host = "83.136.249.159"
port = 36967


def send_directions_and_process_responses(url, direction_sequence):
    for direction in direction_sequence:
        payload = {"direction": direction}
        response = requests.post(url, json=payload)
        
        response_data = response.json()
        
        if "error" in response_data:
            print(response_data["error"])
        
        if response_data.get("solved", False):
            print("We getting there")
            
        if "flag" in response_data:
            print(response_data["flag"])
            exit(0)



def create_graph(data):
    graph = {}
    for key, value in data.items():
        x, y = eval(key)
        graph[(x, y)] = value
    return graph

for i in tqdm(range(100), desc="FOR THE FLAG WE GOOO"):
    url = "http://{}:{}/map".format(host, port)
    response = requests.post(url)
    map_info = response.json()

    data = map_info  
    graph = create_graph(data['tiles'])
    start_position = tuple(data['player']['position'])
    weapon_positions = [eval(key) for key, tile in data['tiles'].items() if tile['has_weapon']]


    shortest_path = None
    lowest_cost = float('inf')
    for weapon_position in weapon_positions:
        path, cost = a_star(graph, start_position, weapon_position, lambda x, y: manhattan(x, y))
        if cost < lowest_cost:
            lowest_cost = cost
            shortest_path = path

    player_time = data['player']['time']
    if lowest_cost <= player_time:
        direction_sequence = get_directions_sequence(shortest_path)

    else:
        direction_sequence = get_directions_sequence(shortest_path)


    url = "http://{}:{}/update".format(host, port)


    send_directions_and_process_responses(url, direction_sequence)
```

It is worth noting that we have also defined a function `get_directions_sequence` that will translate the change in directions and sequences of **L or R or D or U** so the server can understand our requests.

```python
def get_directions_sequence(coordinates):
    directions_sequence = []
    
    for i in range(len(coordinates) - 1):
        current_point = coordinates[i]
        next_point = coordinates[i + 1]
        
        x_change = next_point[0] - current_point[0]
        y_change = next_point[1] - current_point[1]
        
        
        if x_change > 0:
            directions_sequence.extend(['R'] * x_change)
        elif x_change < 0:
            directions_sequence.extend(['L'] * abs(x_change))
        if y_change > 0:
            directions_sequence.extend(['D'] * y_change)
        elif y_change < 0:
            directions_sequence.extend(['U'] * abs(y_change))

    return directions_sequence
```








