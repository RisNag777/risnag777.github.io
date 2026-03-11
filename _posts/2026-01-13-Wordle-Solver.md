---
layout: post
title: "What Building a Wordle Solver Taught Me About AI Agents"
---

<div align="center">
  <img src="https://github.com/user-attachments/assets/bcf01716-7c22-461b-91e8-6162716f42d4" 
       alt="User Input Question Interface" 
       style="border: 1px solid #d0d7de; border-radius: 8px; box-shadow: 0 4px 6px rgba(0,0,0,0.1); max-width: 80%;">
</div><br>

Like most of the human race, towards the end of COVID, I was obsessed with Wordle. Over time my own streaks faded, but the game never really left me. My wife still plays Wordle religiously every morning, and our daily ritual is to solve it together in bed before we kickstart our day. On the days we're completely stumped, my phone quietly becomes our unofficial "six extra lives".

Those moments made me wonder what it would actually mean for a machine to play Wordle well. The game seemed like a simple entry point for an exploration into agentic AI. You have 6 chances to guess a five-letter goal word, and every valid guess provides you some colour-coded feedback:
- Green: The letter is present in the solution and is located in the right position
- Yellow: The letter is present in the solution and is located in the wrong position
- Grey: The letter is absent in the solution

I chose to model "Hard-mode" because, contrary to how it sounds, it seemed like a simpler computational approach. Hard-mode enforces all discovered constraints on every subsequent guess. If your first guess reveals that "A" is the second letter and "E" is present but not in the first position, every following guess **must** respect those rules. This effectively narrows the field of valid words much faster than the standard mode.

### The LLM as the Brain

My first instinct was to build a prompt for `gpt-4o-mini`. The prompt defined:
- The LLM's role and objective
- Heuristic Logic
- Constraint Enforcement

The prompt turned out to be a total dud.  
After guessing "BAKED" and receiving feedback [🟨, 🟩, ⬜, ⬜, ⬜], the model confidently proposed "BATTY". However, despite knowing that "B" was yellow, it was in the exact same position in both words!

### Refining the Logic

I then enhanced the prompt with the following:
1. Treating the guess as a list of characters rather than a string to enforce letter consistency
2. Adding specific markers to the prompt like a data definition (to explicitly define how to interpret the feedback) and a detailed example (that showed both a good and bad guess with reasoning)
3. Asking the LLM to provide an explanation for its guess so I could understand its reasoning

However, I still ran into some significant issues:
1. It looked like the agent was giving me an answer and then trying to justify that by reverse-engineering an explanation that it thought would make sense.
2. Despite all the constraints, the agent would still pick an incorrect guess which fails the hard-mode constraints
3. For repeated letters, the agent would trip up. For example, when the guess is "ALLEY" and the solution is "BLACK", the first "L" would be green while the second would be grey. All future guesses made by the LLM would not contain "L" as it assumed that "L" was only grey.
4. The agent's memory was poor and it would not honour old constraints during future guesses even though it had all the history in its context.

This was a deterministic problem that I was trying to solve with a non-deterministic agent. This pure LLM-based approach was not working.

### Adding Deterministic Tools

I found a list of all possible (2315) Wordle words and used that as my search space. I also added tools to reduce the search space based on all constraints. In order to deal with the issue of repeated letters, the tools processed the constraints in order from green, to yellow, to grey.
These tools worked really well but there was one lingering problem: I could not pass all 2315 words to the LLM. That would unnecessarily increase the number of tokens used by the model. Instead, I had a tool generate a random sample of 20 words which respected all constraints and would tell the model to pick its next guess from words similar to the ones in the random sample.  

### The Takeaway

Even with all this, there were cases where the agent still made bad guesses that didn't match the hard-mode constraints. To top this all of, I could've easily replaced the LLM with a random number generator and still solved Wordle within 6 guesses. This wasn't really the big, unequivocal success that would change the way the world played Wordle, but a useful thing for me to learn early in this journey. Not every problem can be solved by an LLM, but also, not every problem **needs** an LLM. An LLM cannot be used effectively when there is an extremely limiting set of hard constraints that needs to be applied in order to get deterministic results. An LLM works best for problems that require a fuzzy solution, i.e., one where there is room for interpretation.

My main takeaway from this project is that agentic design is about more than just the model. The tools that supply the inputs to and process the outputs from the model are extremely important. They are the difference between just asking ChatGPT vs building your own Agentic Workflow. My first foray into building an agent was an unsuccessful one, but despite that, it was an extremely fruitful one.
