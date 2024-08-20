---
weight: 1
title: "LeetCode - 1870. Minimum Speed to Arrive on Time"
date: 2024-08-20T16:37:00+06:00
lastmod: 2024-08-20T16:37:00+06:00
draft: false
author: "HostileNinja72"
authorLink: "https://HostileNinja72.github.io"
description: "LeetCode 1870 solution explanation"

tags: ["LeetCode", "Medium", "binary search", "C++"]
categories: ["LeetCode"]

lightgallery: true

math:
  enable: true

toc:
  enable: true
---

LeetCode 1870 solution explanation

<!--more-->

## Challenge Description

You are given a floating-point number hour, representing the amount of time you have to reach the office. To commute to the office, you must take n trains in sequential order. You are also given an integer array dist of length n, where dist[i] describes the distance (in kilometers) of the $i^th$ train ride.

Each train can only depart at an integer hour, so you may need to wait in between each train ride.

    For example, if the 1st train ride takes $1.5$ hours, you must wait for an additional 0.5 hours before you can depart on the 2nd train ride at the 2 hour mark.

Return the minimum positive integer speed (in kilometers per hour) that all the trains must travel at for you to reach the office on time, or $-1$ if it is impossible to be on time.

Tests are generated such that the answer will not exceed $10^7$ and hour will have at most two digits after the decimal point.


**Example 1:**

Input: dist = [1,3,2], hour = 6
Output: 1
Explanation: At speed 1:
- The first train ride takes 1/1 = 1 hour.
- Since we are already at an integer hour, we depart immediately at the 1 hour mark. The second train takes 3/1 = 3 hours.
- Since we are already at an integer hour, we depart immediately at the 4 hour mark. The third train takes 2/1 = 2 hours.
- You will arrive at exactly the 6 hour mark.

## Reformulating the problem

We first try to reformulate the problem so we can easily write it as an solve algorithm.

From the explanation, if we are in an integer hour, we depart immediately, so the time will be $dist[i]/x$ such as $x$ is the speed value we are looking for. If we are not in an integer hour, we wait until the 1 hour mark, meaning we are waiting a time value of $dist[i]/x + \left\lceil dist[i]/x \right\rceil$. The last distance will be a normal $dist[i]/x$.

Our task is to find the minimum possible $x$ which is the speed such as the sum of the time $\leq$ hour, in other words:

$$
\begin{aligned}
\sum_{i=1}^{n-1} \left\lceil \frac{\text{dist}[i]}{x} \right\rceil + \frac{\text{dist}[N]}{x} \leq \text{hour}
\end{aligned}
$$

Now we are in front of a non-linear function that involves a ceiling function, the binary search is an efficient way to find the minimum `x`

## Binary search code
We implement a binary search algorithm in a code:

```cpp
class Solution {
public:
bool canReachInTime(const vector<int> &dist, double hour, int speed)
{
    double time = 0;
    for (size_t i = 0; i < dist.size() - 1; ++i)
        time += ceil(static_cast<double>(dist[i]) / speed);
    
    time += static_cast<double>(dist.back()) / speed;
    return time <= hour;
}

int minSpeedOnTime(vector<int> &dist, double hour)
{
    int low = 1, high = 1e7;
    while (low < high)
    {
        int mid = low + (high - low) / 2;
        if (canReachInTime(dist, hour, mid))
            high = mid;
        else
        low = mid + 1;
        
    }
        return canReachInTime(dist, hour, low) ?  low : -1;

}

};
```












