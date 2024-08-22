---
weight: 1
title: "LeetCode - 1870, Minimum Speed to Arrive on Time"
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

You are given a floating-point number hour, representing the amount of time you have to reach the office. To commute to the office, you must take n trains in sequential order. You are also given an integer array dist of length n, where `dist[i]` describes the distance (in kilometers) of the $i^{\text{th}}$ train ride.

Each train can only depart at an integer hour, so you may need to wait in between each train ride.

- For example, if the 1st train ride takes $1.5$ hours, you must wait for an additional 0.5 hours before you can depart on the 2nd train ride at the 2 hour mark.

Return the minimum positive integer speed (in kilometers per hour) that all the trains must travel at for you to reach the office on time, or $-1$ if it is impossible to be on time.

Tests are generated such that the answer will not exceed $10^7$ and hour will have at most two digits after the decimal point.


**Example 1:**

Input: `dist = [1,3,2]`, hour = 6
Output: 1
Explanation: At speed 1:
- The first train ride takes 1/1 = 1 hour.
- Since we are already at an integer hour, we depart immediately at the 1 hour mark. The second train takes 3/1 = 3 hours.
- Since we are already at an integer hour, we depart immediately at the 4 hour mark. The third train takes 2/1 = 2 hours.
- You will arrive at exactly the 6 hour mark.

## Reformulating the problem

Let's begin by reformulating the problem to make it easier to develop a solution algorithm.

Given the problem, if we are at an exact integer hour, we can depart immediately. In this case, the time taken for the $i^{\text{th}}$ train ride is:

$$
\begin{aligned}
\frac{\text{dist}[i]}{x}
\end{aligned}
$$

where $x$ is the speed value we need to determine. If we arrive at a non-integer hour, we must wait until the next full hour before we can depart. Therefore, the effective time spent includes this waiting period, which can be represented as:
 
$$
\begin{aligned}
\frac{\text{dist}[i]}{x} + \left\lceil \frac{\text{dist}[i]}{x} \right\rceil - \frac{\text{dist}[i]}{x}
\end{aligned}
$$

For the last train ride, there is no need to wait, so the time taken is simply:

$$
\begin{aligned}
\frac{\text{dist}[i]}{x} 
\end{aligned}
$$

Our objective is to find the minimum speed $x$ such that the total time spent on all train rides is less than or equal to the given hour. This can be expressed mathematically as:

$$
\begin{aligned}
\sum_{i=1}^{n-1} \left\lceil \frac{\text{dist}[i]}{x} \right\rceil + \frac{\text{dist}[n]}{x} \leq \text{hour}
\end{aligned}
$$

Given that this is a non-linear function involving a ceiling operation, binary search is a suitable and efficient method to find the minimum value of x that satisfies this condition.

## Binary search code
We implement a binary search algorithm in a `C++` code:

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

int minSpeedOnTime(const vector<int> &dist, double hour)
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
And then our submission will get all the tests validated.











