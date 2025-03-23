
Connect-4 AI Board Representation
=================================

In which we discuss the board representation of Connect-4 in an artificial intelligence algorithm (AI).

# The "Easy" Way
The first time I made this AI, I created a two dimensional (7x6) array to represent the board (`char[7][6] board`). This is the most intuitive representation, as you can get the piece value by calling `board[x][y]`.

While this design seems simple, it becomes a headache when it comes to checking for terminal states (e.g. if a Connect-4 has been made) or evaluation (determining how "good" a position is for a player).

For example, checking for a Connect-4 consists of looping through each piece of the board for all 4 directions! 7x6x4 = 168 processes, give-or-take, for one game state out of the millions you'll be calculating!

Clearly this board representation is not satisfactory for an optimal AI.

# Compressing a 2D board
If we know the dimensions of the board, we can easily compress a 2D board into a one-dimensional array. All we have to do is correspond the index of the 1D array to a value on a 2D model. Here's an example for a Connect-4 board using an array with 42 elements:

```
35 36 37 38 39 40 41
28 29 30 31 32 33 34
21 22 23 24 25 26 27
14 15 16 17 18 19 20
 7  8  9 10 11 12 13
 0  1  2  3  4  5  6
```
*Note: For example only! We will modify this representation later!*

We can still get the space value from an x,y coordinate pair by multiplying `y` by the width and adding `x`, e.g:

`(2, 1) = x + 7y = 9` (coordinates from bottom left)

This helps us limit memory and data complexity, but, if using arrays, it doesn't do us much good for speed.

# The Solution: Bitboards
How abut we do away with arrays completely, and examine a new model: bitboards. The concept is simple. Instead of using a single array with a value for each space type *(e.g. `0`: Blank | `1`: Player 1 Stone | `2`: Player 2 Stone)* in a traditional array as before, we will store everything in two arrays of true/false (boolean) values, one for each player. Perhaps an illustration will help:

The game: (with X representing Player 1 and O, Player 2)
```
- - - - - - -
- - - - - - -
- - - O - - - 
- - - X - - - 
- - - O X - -
- - - X O X -
```

Will be represented as:

Player 1 bitboard (represents Player 1's pieces):
```
0 0 0 0 0 0 0
0 0 0 0 0 0 0
0 0 0 0 0 0 0
0 0 0 1 0 0 0
0 0 0 0 1 0 0
0 0 0 1 0 1 0
```
`1`: True | `0`: False

Player 2 bitboard (represents Player 2's pieces):
```
0 0 0 0 0 0 0
0 0 0 0 0 0 0
0 0 0 1 0 0 0
0 0 0 0 0 0 0
0 0 0 1 0 0 0
0 0 0 0 1 0 0
```

Here's the catch: instead of actual arrays, these bitboards will be integers. We know that computers represent numbers in binary, and luckily our programming languages provide a way to interact with them in binary: [bitwise operations](https://en.wikipedia.org/wiki/Bitwise_operations_in_C).

# Win Checking
Let's start by designing the algorithm for checking if a Connect-4 has been made, signaling a win. We will go into more detail later, but the concept will help us design our board representation.

## The Idea
What makes a Connect-4? We can conjecture that **a Connect-4 has been made if any piece has 3 consecutively adjacent neighbors in the same direction.**

Let's look at a row of the board: 

`0 0 1 1 1 1 0`

If we perform a right bitshift of 1 (`>> 1`), it becomes 

`0 1 1 1 1 0 0`.

 *Notice how the right bitshift actually has the effect of shifting the row left. This is because numbers are read with the first digit on the right, wheras in our board the first digit is on the left.* If we compare the two integers with an AND operation:
```
b = player's bitboard

b & (b >> 1)
  0 0 1 1 1 1 0
& 0 1 1 1 1 0 0
= 0 0 1 1 1 0 0
```

Our new bitboard represents all the bits with a neighbor to the right. It's asking, "If I shift the board over, will there still be a 1 in my place?" You can also think of it as finding all the "Connect-2s".

A Connect-4 is made when two "Connect-2s" are together. In other words, if I have a neighbor and the piece two spaces to the right also has a neighbor, it is a Connect-4.
```
n = last output (all pieces with neighbors to the right)

n & (n >> 2)
  0 0 1 1 1 0 0
& 1 1 1 0 0 0 0
= 0 0 1 0 0 0 0
```
And there we have it! `n` represents all pieces with three consecutively adjacent neighbors. Win checking is simple, if `n` is equal to 0 (all bits are 0, meaning no pieces meet the definition), then the game continues (assuming the board is not full, which we will get to later). If `n` is greater than 0, it means there is a 1 somewhere in our bitboard, and a win exists.

Of course, we have only checked the horizontal direction so far, but more on the other directions soon!

## The Benefits
The benefit this method presents over the others is huge. Shifting and comparing bits is a low-level, easy function for a CPU *(your microwave can probably do bitshifts!)*, much easier than a for loop.

## A Pretty Big Problem
The only thing is, we have a pretty big problem with our win-checking system now. Remember that our bitboard is a one-dimensional binary number that is made to represent a two-dimensional board. Now consider the first two rows of this bitboard:
```
1 1 0 0 0 0 0
0 0 0 0 0 1 1
```
Clearly no Connect-4s here. But our program sees it as:

`0 0 0 0 0 1 1 1 1 0 0 0 0 0`

Now that looks like a Connect-4, and our algorithm will think it is. We need a way to separate the rows.

# Better Board Representation
All we need to do is add a [sentinel](https://en.wikipedia.org/wiki/Sentinel_value) column to act as a sort of "air-gap" between rows. You can also think of it as adding an unused column that will always be empty, so that Connect-4s won't happen across rows. Here's how that looks:
```
C0   C1  C2  C3  C4  C5  C6 Sent

[47][48][49][50][51][52][53][54]
 40  41  42  43  44  45  46 [46]
 32  31  32  33  34  35  36 [39]
 24  25  26  27  28  29  30 [31]
 16  17  18  19  20  21  22 [23]
  8   9  10  11  12  13  14 [15]
  0   1   2   3   4   5   6 [ 7]
```
Sentinel values are denoted by the brackets(`[]`) around them. I have also added a row of sentenial values at the top, which doesn't change anything but will help us conceptualize our move-generation algorithm later.

Now, as long as we are careful to make sure that our sentinals are always 0, we won't have the issue of wrapping.

# C++ Application
Although I covered the concept of win-checking here, the next post will go into more depth, so I will limit this application to the actual board representation.

The best data type for your bitboards is going to be a `uint64_t`. This is faster than an int because it is smaller, only including 64 bits.

We will also have an array `uint64_t[7]` to store the locations of the next piece on the bitboard per column. More on this in the next post.

---
Title: Connect-4 AI Board Representation

Author: Tux-76

Date: March 12, 2025
___
