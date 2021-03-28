---
layout: post
title: Maze of Tirna Scithe Code Kata
---

I think code katas are a great way to learn a new language or to improve your existing coding skills. If you truly want to learn, you need to practice, and code katas are a good starting point for that. I enjoy them because there are small enough that you can understand the problem to solve quickly and get the exercise into a "good enough" state (aka "done") that you don't feel like it's just another abandoned project.

Common code katas are interesting because you can easily find implementations from others and learn from those. Reading other peoples code can be an extremely valuable learning source. But you can only implement Fizz-Buzz so many times before it becomes tiring. That's why I invent my own code katas, small problems that can easily be digested. Since I love playing games, I try to convert some easy game mechacics into katas. Usually they consist of some game logic to solve while providing many options to extend the katas in different directions, depending on what you want to learn.

In this blog post, I'll introduce you to one of the latest kata that I've enjoyed, the Maze of Tirna Scithe.

## The Rules

It is a very simple game mechanic that I took from a World of Warcraf dungeon. The dungeons requires a party of players to find a way through the Maze of Tirna Scithe, which is a a set of rooms connected to each other. Every room contains 4 doors that lead to another room of the Maze but there is only one correct door that you can use. If you use the wrong door, you have to start over from the beginning.

How do you find out which door to use then? There are 4 pillars, one before each door, each with a unique symbol. The symbol consists of 3 different attributes:
* It has a shape, that is either a leaf or a blossom.
* The shape can be filled with a color, or not.
* The shape can be surrounded by a circle, or not.

The symbols aren't just selected randomly. Every symbol has at least another symbol that shares one of the 3 attributes, except for one. One symbol will have one attribute that no other symbol in the same room shares (but it can have other attributes that are shared!). The challenge is to find the outlier and advance to the next room by going through the door behind that pillar.

A bit confusing? Here are some examples:

...TODO...


## An example implementation

I used this kata to write some Rust code, which I've been trying to learn out of curiosity. To every Rust developer: I'm sorry about the following code, these are about the first few lines of Rust I've written.

Katas work fairly well with TDD. Mostly because you will refactor your code all the time and you don't want to manually test every single time you've changed something. So I usually start with a test case:

..TODO..

Now you got the test case, you can start implementing the very basic logic. It's your decision whether you want to follow the "pure breed TDD" approach and just do the minimal thing to get the test green vs. jumping a few steps ahead and maybe writing a few tests at once before implementing it. After implementing some test cases, you probably have some refactoring ideas. Don't be afraid of writing "ugly" or "inefficient" code to get the code working, working code always wins. Once you have some working code, refactor as you like. Try different ideas and find out which one you like the most. After some iterations (and a few more test cases) which started with plenty of "if/else" statements, I've ended up with something like this:


..TODO

This is just one implementation that fulfils the defined constraints (and most likely not a good one). The full code can be found here if you want to read more bad Rust code.

## Adjust the exercise

I like game-based katas because you can extend the exercise in so many different ways, entirely depending on what you enjoy or want to learn. This kata, for example, could easily be extended by one of these next steps:
* Build a console based UI, or even a graphical UI to play the game.
* Write some code to generate valid symbol combinations for a room.
* Benchmark your your code and find a more memory or CPU efficient implementation.

Have you come up with your own code katas too? I'd love to hear about them or to see your approach to implement this kata. If you're a Rust dev, your feedback is of course also welcome.
