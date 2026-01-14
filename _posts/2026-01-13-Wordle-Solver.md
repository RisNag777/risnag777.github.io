---
layout: post
title: "Wordle Solver"
---

Like most of the human race, towards the end of COVID, I was obsessed with Wordle. However, over the last year, after multiple broken streaks, I eventually fell off the wagon. Despite that, Wordle continues to be a big part of my life. My wife plays Wordle religiously every morning and our daily ritual is to play it together in bed before we kickstart our day. Sometimes, we're completely stumped and that's when my phone comes into play to 'unlock' an additional 5 chances!  

Naturally, it felt like a good idea to build an Agentic Wordle solver!  
My goal was to have a system inspired by Wordle's own WordleBot, an algorithm that evaluates your guesses based on how effectively they shrink the remaining solution space. Rather than hard-coding heuristics, I wanted to explore whether an agent could reason its way into selecting the next best guess at each step.  

The basic logic was simple  
1. Make a guess (5-letter word)  
2. Receive feedback:  
    Green (2): correct letter, correct position (I refer to these as Bulls)  
    Yellow (1): correct letter, wrong position (I refer to these as Cows)  
    Gray (0): letter not in the word  
3. Trim the solution space  
4. Choose the next guess  
5. Repeat until solved (≤ 6 guesses)

The first step was to find a list of all the possible solutions for Wordle. I used the list provided here - [https://github.com/tabatkins/wordle-list](https://github.com/tabatkins/wordle-list). A solution was picked at random from the above list.
