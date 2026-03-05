---
layout: post
title: "What Building a Wordle Solver Taught Me About AI Agents"
---

![Wordle](https://github.com/user-attachments/assets/bcf01716-7c22-461b-91e8-6162716f42d4)

Like most of the human race, towards the end of COVID, I was obsessed with Wordle. Over time my own streaks faded, but the game never really left me. My wife still plays Wordle religiously every morning, and our daily ritual is to solve it together in bed before we kickstart our day. On the days we're completely stumped, my phone quietly becomes our unofficial "six extra lives".

Those moments made me wonder what it would actually mean for a machine to play Wordle well. The game seemed like a simple entry point for an exploration into agentic AI. You have 6 chances to guess a five-letter goal word, and every valid guess you provides you some colour-coded feedback:
- Green: The letter is present in the solution and is located in the right position
- Yellow: The letter is present in the solution and is located in the wrong position
- Grey: The letter is absent in the solution
I chose to model "Hard-mode" because, contrary to how it sounds, it seemed like a simpler computational approach. Hard-mode enforces all discovered constraints on every subsequent guess. If your first guess reveals that 'A' is the second letter and 'E' is present but in the wrong position, every following guess **must** respect those rules. This effectively narrows the field of valid words much faster than the standard mode.

### The LLM as the Brain

My first instinct was to build a prompt for `gpt-4o-mini`. The prompt defined:
- The LLM's role and objective
- Grounding of the Solution Space
- Heuristic Logic
- Constraint Enforcement

The prompt turned out to be a total dud.

The following example illustrates the biggest issue I ran into:
After guessing "naked" and receiving feedback [🟨, 🟩, ⬜, ⬜, ⬜], the model confidently proposed "nasty" and explained that:
- "a" remains in the correct position
- "n" is included but in a different position
- "k", "e", "d" are absent
However, "n" was in the exact same position in both words!
It looked like the agent was giving me an answer and then trying to justify that by reverse-engineering an explanation that it thought would make sense.

### Refining the Logic

I finetuned the prompt but still kept running into issues like:
1. The output would not be in the right format I was expecting
2. The guessed word would be correct but the explanation for choosing it would not (and vice-versa)
3. The number of remaining words would shrink far quicker than I was expecting
The third problem was a particularly interesting one and was a consequence of how the code was handling duplicate letters. To refresh your memory, here is a case where the guess is "ALLEY" and the solution is "BLACK". What happens in the game is:
- The first L (position 1) is assigned feedback 🟩
- The second L (position 2) is assigned feedback ⬜
This means that:
- L exists in the word and MUST be at position 1
- L cannot be at position 2
So, I made it so that all greens were processed first, followed by yellows, and finally greys. This ensured that duplicates were properly handled. However, while trimming the solution space, the code would see that the second L was assigned grey and thereby, it would assume that all valid words did not contain the letter "L".
I fixed this by implementing a two-pass constraint engine. The first pass handled the duplicates as described above, while the second pass ensured that letters that were assigned green or yellow were absolutely not eliminated from the solution space. This made sure that the constraints were being strictly followed when the solution space was trimmed.

### The Takeaway

By now, you must have noticed that the only thing that the LLM is being used for is to select a random word from a filtered solution space of up to 20 words. That job could be done by a random number generator. Not really the big, unequivocal success that would change the way the world played Wordle, but a useful thing for me to learn early in this journey. Not every problem can be solved by an LLM, but also, not every problem **needs** an LLM. An LLM cannot be used effectively when there is an extremely limiting set of rules that needs to be applied in order to get the expected results. An LLM works best for problems that require a fuzzy solution, i.e., one where there is room for interpretation.

My main takeaway from this project is that agentic design is about more than just the model. The functions that supply the inputs to and process the outputs from the model are still extremely important. They provide the structure to the inherently unstructured nature of the LLM, extracting useful information from not necessarily coherent rambling. My first foray into building an agent was an unsuccessful one, but despite that, it was an extremely fruitful one. 
