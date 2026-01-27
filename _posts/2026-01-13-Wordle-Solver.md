---
layout: post
title: "Wordle Solver"
---

Like most of the human race, towards the end of COVID, I was obsessed with Wordle. However, over the last year, after multiple broken streaks, I eventually fell off the wagon. Despite that, Wordle continues to be a big part of my life. My wife plays Wordle religiously every morning and our daily ritual is to play it together in bed before we kickstart our day. Sometimes, we're completely stumped and that's when my phone comes into play to 'unlock' an additional 5 chances!  

Naturally, I decided to build an agentic Wordle solver!  
My goal was to have a system inspired by Wordle's own WordleBot, an algorithm that evaluates your guesses based on how effectively they shrink the remaining solution space. Rather than hard-coding heuristics, I wanted to explore whether an agent could reason its way into selecting the next best guess at each step.  

## Wordle Is a Belief-State Search Problem  
A good Wordle solver is not guessing. It is updating beliefs.  
The game can be reimagined as:  
| Concept      | Wordle meaning                     |
| ------------ | ---------------------------------- |
| Hidden state | The true solution word             |
| Belief state | All words consistent with feedback |
| Action       | Choosing the next guess            |
| Observation  | Feedback (0,1,2 per letter)        |  
This makes Wordle a sequential inference problem, and not a language task.  

## System Design
At a high level, the system follows this loop:  
1. Make a guess (5-letter word)  
2. Receive feedback:  
    Green (2): correct letter, correct position (I refer to these as Bulls)  
    Yellow (1): correct letter, wrong position (I refer to these as Cows)  
    Gray (0): letter not in the word  
3. Trim the solution space  
4. Choose the next guess  
5. Repeat until solved (≤ 6 guesses)

### Architecture

<svg xmlns="http://www.w3.org/2000/svg" width="1200" height="720" viewBox="0 0 1200 720">
  <defs>
    <style>
      .box { fill: #ffffff; stroke: #111827; stroke-width: 2; rx: 16; ry: 16; }
      .accent { fill: #EEF2FF; stroke: #6366F1; stroke-width: 2; rx: 16; ry: 16; }
      .note { fill: #FFFBEB; stroke: #F59E0B; stroke-width: 2; rx: 16; ry: 16; }
      .title { font: 700 20px ui-sans-serif; fill: #111827; }
      .text { font: 600 16px ui-sans-serif; fill: #111827; }
      .muted { font: 500 14px ui-sans-serif; fill: #374151; }
      .arrow { stroke: #111827; stroke-width: 2.5; fill: none; marker-end: url(#arrowhead); }
    </style>
    <marker id="arrowhead" markerWidth="12" markerHeight="12" refX="10" refY="6" orient="auto">
      <path d="M 0 0 L 12 6 L 0 12 z" fill="#111827"/>
    </marker>
  </defs>

  <text x="60" y="60" class="title">Wordle Agent Architecture</text>
  <text x="60" y="90" class="muted">Deterministic logic enforces rules. The LLM only proposes guesses.</text>

  <!-- Environment -->
  <rect x="60" y="150" width="360" height="120" class="box"/>
  <text x="90" y="190" class="text">Wordle Environment</text>
  <text x="90" y="220" class="muted">Hidden solution word</text>
  <text x="90" y="245" class="muted">Returns feedback (0,1,2)</text>

  <!-- Feedback Logic -->
  <rect x="60" y="300" width="360" height="120" class="box"/>
  <text x="90" y="340" class="text">Game Logic</text>
  <text x="90" y="370" class="muted">get_feedback()</text>
  <text x="90" y="395" class="muted">Handles bulls, cows, duplicates</text>

  <!-- Belief Update -->
  <rect x="60" y="450" width="360" height="140" class="box"/>
  <text x="90" y="490" class="text">Belief-State Update</text>
  <text x="90" y="520" class="muted">trim_list()</text>
  <text x="90" y="545" class="muted">Filters impossible words</text>

  <!-- Internal State -->
  <rect x="470" y="300" width="340" height="200" class="accent"/>
  <text x="500" y="340" class="text">Internal State</text>
  <text x="500" y="370" class="muted">Candidate words</text>
  <text x="500" y="395" class="muted">Guess history</text>
  <text x="500" y="420" class="muted">Current turn</text>

  <!-- Policy Layer -->
  <rect x="880" y="220" width="260" height="140" class="box"/>
  <text x="910" y="260" class="text">Policy Layer</text>
  <text x="910" y="290" class="muted">Select next guess</text>
  <text x="910" y="315" class="muted">Heuristic or stochastic</text>

  <!-- LLM -->
  <rect x="880" y="400" width="260" height="140" class="box"/>
  <text x="910" y="440" class="text">LLM (Optional)</text>
  <text x="910" y="470" class="muted">Ranks valid guesses</text>
  <text x="910" y="495" class="muted">Uses language priors</text>

  <!-- Validity Gate -->
  <rect x="880" y="580" width="260" height="100" class="note"/>
  <text x="910" y="620" class="text">Validity Check</text>
  <text x="910" y="645" class="muted">Guess ∈ candidates?</text>

  <!-- Arrows -->
  <path class="arrow" d="M 240 270 L 240 300"/>
  <path class="arrow" d="M 240 420 L 240 450"/>
  <path class="arrow" d="M 420 520 L 470 400"/>
  <path class="arrow" d="M 810 360 L 880 290"/>
  <path class="arrow" d="M 810 440 L 880 470"/>
  <path class="arrow" d="M 1010 540 L 1010 580"/>
</svg>


### Step 1
Find a list of all the possible solutions for Wordle. I used the list provided here - [https://github.com/tabatkins/wordle-list](https://github.com/tabatkins/wordle-list). A solution was picked at random from the above list of 14,855 words. These were taken from the game's source code.

### Step 2
After each guess, track the Bulls, Cows and Absent letters.  
Here's where I ran into my first problem. Absent letters don't necessarily imply that the letter is not in the word, but they are position-specific. Meaning that if I guess the word "ALLEY", the first "L" might show yellow while the second "L" shows gray. The tool needs to know to not ignore "L" because it would be gray.  
I had to keep my feedback order fixed. Ensure that a letter is considered absent **ONLY** if it is not a bull or a cow.  

### Step 3
The solution space is then reduced.  
Words containing letters that are truly absent (i.e., not bulls or cows anywhere) are removed. (This is an actual condition of Wordle Hard Mode, for a first attempt I decided to stick with it because it made the coding a bit easier). Also, the remaining solution space will not contain words where the Bulls are not in the correct position or the Cows that are in the same position as in the guess.  
I had to implement this because when I was giving the LLM access to the guessed word and the feedback, it would:
- Produce illegal words
- Violate constraints
- Give confident but false reasoning
For the false reasoning, it seemed like the LLM would generate a plausible guess (either right or wrong) and then try to give an explanation for why it picked that word (which would be wholly incorrect).  
For example, the guess was 'naled' (yeah, a real word accepted by Wordle) and the feedback was \[1,2,0,0,0\]
The agent suggested the next guess to be 'nasty' and gave the reasoning as below:  
reason = This guess satisfies the feedback rules as follows:  
- The letter 'a' is in the correct position (index 1), which is maintained in this guess.  
- The letter 'n' is included but in a different position (index 0), which is required since it received a feedback of 1 in the previous guess.  
- The letters 'l', 'e', and 'd' are not included in this guess, as they received a feedback of 0, indicating they are not in the solution.  
- The guess contains common letters like 's' and 't', which may help in identifying the solution.
The agent is confident about its answer but the letter 'n' is in the same position it was before.
The model wasn’t reasoning. It was choosing a guess first and fabricating justification afterward.  
This showed me that the LLM must never own the game logic. Deterministic code must enforce rules, otherwise correctness becomes probabilistic.

## The Agent Loop
Here’s the core system loop:  
```
candidates = retrieve_word_list()
solution = random.choice(candidates)

for turn in range(6):
    feedback = get_feedback(guess, solution)
    candidates = trim_list(guess, feedback, candidates)
    ...
    guess = LLM_policy(candidates)
```

<svg xmlns="http://www.w3.org/2000/svg" width="1200" height="500" viewBox="0 0 1200 500">
  <defs>
    <style>
      .box { fill: #ffffff; stroke: #111827; stroke-width: 2; rx: 16; ry: 16; }
      .accent { fill: #EEF2FF; stroke: #6366F1; stroke-width: 2; rx: 16; ry: 16; }
      .warn { fill: #FFFBEB; stroke: #F59E0B; stroke-width: 2; rx: 16; ry: 16; }
      .title { font: 700 20px ui-sans-serif; fill: #111827; }
      .text { font: 600 16px ui-sans-serif; fill: #111827; }
      .muted { font: 500 14px ui-sans-serif; fill: #374151; }
      .arrow { stroke: #111827; stroke-width: 2.5; fill: none; marker-end: url(#arrowhead); }
    </style>
    <marker id="arrowhead" markerWidth="12" markerHeight="12" refX="10" refY="6" orient="auto">
      <path d="M 0 0 L 12 6 L 0 12 z" fill="#111827"/>
    </marker>
  </defs>

  <text x="60" y="60" class="title">Agent Turn Loop</text>
  <text x="60" y="90" class="muted">Guess → Feedback → Trim → Prompt LLM → Validate → Next Guess</text>

  <rect x="60" y="160" width="180" height="90" class="accent"/>
  <text x="85" y="200" class="text">Guess</text>

  <rect x="270" y="160" width="200" height="90" class="box"/>
  <text x="295" y="200" class="text">Feedback</text>

  <rect x="500" y="160" width="200" height="90" class="box"/>
  <text x="525" y="200" class="text">Trim Candidates</text>

  <rect x="730" y="160" width="200" height="90" class="box"/>
  <text x="755" y="200" class="text">LLM Suggestion</text>

  <rect x="960" y="160" width="200" height="90" class="box"/>
  <text x="985" y="200" class="text">Validate Guess</text>

  <rect x="960" y="290" width="200" height="90" class="warn"/>
  <text x="985" y="330" class="text">Retry or Fallback</text>

  <path class="arrow" d="M 240 205 L 270 205"/>
  <path class="arrow" d="M 470 205 L 500 205"/>
  <path class="arrow" d="M 700 205 L 730 205"/>
  <path class="arrow" d="M 930 205 L 960 205"/>
  <path class="arrow" d="M 1060 250 L 1060 290"/>
  <path class="arrow" d="M 1060 380 L 150 380 L 150 250"/>
</svg>


| Layer          | Responsibility    |
| -------------- | ----------------- |
| `get_feedback` | Environment model |
| `trim_list`    | Belief update     |
| Candidate list | Internal state    |
| LLM            | Policy proposal   |

This is the agent structure.  
At this point, the system had all the components of a classical AI agent: an internal state (candidate list), a perception-update loop (feedback → trim), and a policy for action selection. The architecture is what made the agent really intelligent.

## The Real Role of the LLM
LLMs are good at language priors and heuristic scoring but fundamentally bad at constraint enforcement.  
So in this system, the LLM acts as:  
- Heuristic ranker
- Proposal generator

## What This Project Really Shows  
This isn’t really about an LLM that solves Wordle  
It’s more like:  

> **How to build an AI agent where language models assist reasoning - but never replace it.**

The intelligence is distributed as follows:

| Component     | Type            |
| ------------- | --------------- |
| Game logic    | Deterministic   |
| Belief update | Deterministic   |
| Policy        | Stochastic      |
| LLM           | Heuristic prior |

If the LLM is removed -> the logic still works  
But if the reasoning engine is removed -> there is no way the agent can do anythin useful  

## Final Insight
The project started as a Wordle bot and ended as a lesson in:
- agent design
- belief-state search
- and why LLMs should be advisors, not authorities

## Lessons Learned Building Agents
The biggest lesson from this project is that agentic behavior isn’t magic — it’s architecture. Once you separate the system into:
1. A deterministic world model
2. A belief state
3. A policy that chooses actions
you get something that behaves like an agent almost automatically.

The second lesson is more uncomfortable: LLMs are not rule engines. They’re excellent heuristic priors, but they will confidently generate illegal actions unless your system makes illegal moves impossible. The safest pattern is the one used in mature AI systems everywhere: the model proposes, and deterministic code verifies.

Finally, debugging Wordle taught me what building agents usually teaches: most of the real difficulty lives in the “boring parts” — state representation, edge cases, and enforcement. Once those are correct, you can plug in entropy scoring, stochastic policies, or LLM ranking and everything gets easier because the foundation is solid.
