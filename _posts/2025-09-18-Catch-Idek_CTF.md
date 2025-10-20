---
title: "Catch [ IDEK CTF ]"
date: 2025-08-02
categories: [Crypto]
tags: [cryptography]
author: dead_droid
---
Author: Giapppp Description: Through every cryptic stride, the cat weaves secrets in the fabric of numbers—seek the patterns, and the path shall reveal the prize.

## Understanding the challenge
Meet the Cat and Its Map. The cat starts at some secret spot with the x and y random coordinates. It also has a bag of 125 little “tiles” (8-byte pieces).these pieces are fragments of random 1000 bytes long data. it will select 30 times random pieces from that list of steps (8-byte pieces) and perfrom this walking function with each piece or 8 byte.

```python
def walking(x, y, part):
    # Each step is guided by a fragment of the cat's own secret mind.
    epart = [int.from_bytes(part[i:i+2], "big") for i in range(0, len(part), 2)]
    xx = epart[0] * x + epart[1] * y
    yy = epart[2] * x + epart[3] * y
    return xx, yy
```
it will return xx and yy, the new coordination and it will use these in next turn with different piece or part and will do it 30 times. and in the end it will reach to a final coordination. than it will give us its initial position co-ordinations and the final cordinations and than in last it will give us that 1000 bytes long data which was used to create the pieces list or steps list.

then it will ask us to give the all 30 pieces used to reach the final coordinates from initial coordinates using that walking function. cause we don't know which peices it selected we just can't directly reverse the steps and know the pieces.

solution - we can somehow reverse the walking function using the inverse matrix and indentity matrix varification which can help us to figure out two vairables used in two different equations with known co-officients. this is the reverse logic i used.
```python
def reverse_walking(xx, yy, part):
    # it will take one part from the steps list and try to do inverse matrix varification to find out the previous values of x and y.
    epart = [int.from_bytes(part[i:i+2], "big") for i in range(0, len(part), 2)]
    a, b, c, d = epart
    det = a * d - b * c
    if det == 0:
        return False
    num_x = d * xx - b * yy
    num_y = -c * xx + a * yy
    if num_x % det != 0 or num_y % det != 0:
        return False
    x = num_x // det
    y = num_y // det
    return x, y
```
using this we can get the value of x and y used in previous step. we can do it 30 times and we will reach the initial_x and initial_y. and we can brute force the 125 steps or pieces in this process. this function is exactly doing the thing.

```python
def reverse_it(initial_x,initial_y,xx,yy,step):
    # this function will find out the all the parts used in to move from initial values to xx and yy. 
    parts_list = []
    while True:
        for i in range(len(step)):
            part = step[i]
            if reverse_walking(xx, yy, part):
                xx,yy = reverse_walking(xx, yy, part)
                parts_list.append(part)
            else:
                continue 
            if xx == initial_x or yy == initial_y:
                return parts_list
            break
```
and this is how we can solve it. now we just need to get the parts_list, reverse it and convert it to bytes strings and than convert it to hex. now we can supply this hex to the challenge server and repeat the same process 20 times. i just automated everything and it fetches the data from the challenge server using sockets and we can get the flag in few seconds.

`idek{Catch_and_cat_sound_really_similar_haha}`