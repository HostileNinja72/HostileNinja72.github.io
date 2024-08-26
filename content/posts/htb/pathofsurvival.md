---
weight: 1  
title: "HTB Cyber Apocalypse 2024 - Path of Survival"  
date: 2024-01-29T16:37:00+06:00  
lastmod: 2024-01-29T16:37:00+06:00  
draft: false  
author: "HostileNinja72"  
authorLink: "https://HostileNinja72.github.io"  
description: "Writeup for Path of Survival Misc Challenge."

tags: ["misc", "HTB", "algorithms", "Dijkstra", "hard", "A*"]  
categories: ["Writeups"]

lightgallery: true

math:  
  enable: true

toc:  
  enable: true

---

Writeup for the "Path of Survival" challenge from HTB Cyber Apocalypse CTF 2024.

<!--more-->

## Initial Analysis

Upon starting the instance, we receive an IP and a port. Connecting to the service reveals what appears to be a game.

<kbd> <img src="https://github.com/hostileninja72/hostileninja72.github.io/blob/main/content/posts/htb/initial_view.png?raw=true" style="border-radius:2%" align="center" width = "100%" /> </kbd>

The game features multiple squares, each with a specific image representing its nature. There is also a player icon. Reading the rules, we learn that the objective is to travel between tiles to reach a weapon tile before time runs out.

### Rules Overview

- **Empty** tiles cannot be accessed.
- **Cliffs** can only be entered from the top and left.
- **Geysers** can only be entered from the bottom and right.

### Time Costs

- Movement to/from a **Cliff** or **Geyser**: 1 time point
- Travel from one tile of terrain to another of the same terrain: 1 time point
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

To win 100 times consecutively, we need to automate the process using the provided API.

## Solution

This is a pathfinding challenge, so we must implement an appropriate algorithm. I chose the A* algorithm over Dijkstra's because A* is designed for finding the shortest path from a single source to a single target node, combining efficiency and accuracy.

### Setting Up

#### Building the Code

First, I set up a cost dictionary:

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
Upon sending a POST request, we receive the following response:

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

$g(n)$: The actual cost from the start node to the current node $n \\$ .
$h(n)$: The heuristic estimated cost from the current node n to the goal node.

Now with the implementation:
First we define a function `get_cost`, that we will need in the `a_star` function.
In this function we define the constraints giving in rules.

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


Here is the `A*` algorithm:

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
## Main Code

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

It is worth noting that we have also defined a function `get_directions_sequence` that will translate the change in directionst into sequences of **L, R, D or U** so the server can understand our requests.

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

By running the code, We get the following flag: `HTB{i_h4v3_mY_w3ap0n_n0w_dIjKStr4!!!}`








