---
layout: post
title: "What Building a Wordle Solver Taught Me About AI Agents"
---

Like most of the human race, towards the end of COVID, I was obsessed with Wordle. However, over the last year, after multiple broken streaks, I eventually fell off the wagon. Despite that, Wordle continues to be a big part of my life. My wife plays Wordle religiously every morning and our daily ritual is to play it together in bed before we kickstart our day. Sometimes, we're completely stumped and that's when my phone comes into play to 'unlock' an additional 5 chances!  

During those moments, I started thinking about what it would actually mean for a machine to play Wordle well. It wouldn’t memorize patterns (that’s not how the game works), but it would maintain beliefs about possible solutions, update those beliefs with evidence from feedback, and choose its next action strategically. In other words, it would behave like an AI agent.

Naturally, I decided to build an agentic Wordle solver! My goal was to have a system inspired by Wordle's own WordleBot, an algorithm that evaluates guesses based on how effectively they shrink the remaining solution space. Rather than hard-coding heuristics, I wanted to explore whether an agent could reason its way into selecting the next best guess at each step.  

## System Design
Wordle Is a Belief-State Search Problem. A good Wordle solver is not guessing. It is updating beliefs.  
At a high level, the system follows this loop:  
1. Make a guess (5-letter word)  
2. Receive feedback:  
    Green (2): correct letter, correct position (I refer to these as Bulls)  
    Yellow (1): correct letter, wrong position (I refer to these as Cows)  
    Gray (0): letter not in the word  
3. Trim the solution space  
4. Choose the next guess  
5. Repeat until solved (≤ 6 guesses)

The game can be reimagined as:  

| Concept      | Wordle meaning                     |  
| ------------ | ---------------------------------- |  
| Hidden state | The true solution word             |  
| Belief state | All words consistent with feedback |  
| Action       | Choosing the next guess            |  
| Observation  | Feedback (0,1,2 per letter)        |  

This reframes Wordle as a sequential inference problem, and not a language task.  

### Architecture
<svg xmlns="http://www.w3.org/2000/svg" width="680" height="780" viewBox="0 0 680 780">
  <defs>
    <style>
      .box { fill: #ffffff; stroke: #111827; stroke-width: 2; rx: 14; ry: 14; }
      .accent { fill: #EEF2FF; stroke: #6366F1; stroke-width: 2; rx: 14; ry: 14; }
      .note { fill: #FFFBEB; stroke: #F59E0B; stroke-width: 2; rx: 14; ry: 14; }
      .title { font: 700 22px ui-sans-serif; fill: #111827; }
      .text { font: 600 16px ui-sans-serif; fill: #111827; }
      .muted { font: 500 14px ui-sans-serif; fill: #374151; }
      .arrow { stroke: #111827; stroke-width: 2.5; fill: none; marker-end: url(#arrowhead); }
    </style>
    <marker id="arrowhead" markerWidth="12" markerHeight="12" refX="10" refY="6" orient="auto">
      <path d="M 0 0 L 12 6 L 0 12 z" fill="#111827"/>
    </marker>
  </defs>

  <text x="40" y="50" class="title">Wordle Agent Architecture</text>
  <text x="40" y="80" class="muted">Belief-state reasoning + policy + LLM proposal + deterministic validation</text>

  <!-- Row 1 -->
  <rect x="40" y="120" width="260" height="110" class="box"/>
  <text x="60" y="155" class="text">Wordle Environment</text>
  <text x="60" y="180" class="muted">Hidden solution</text>
  <text x="60" y="200" class="muted">Returns feedback</text>

  <rect x="360" y="120" width="260" height="110" class="box"/>
  <text x="380" y="155" class="text">Game Logic</text>
  <text x="380" y="180" class="muted">get_feedback()</text>
  <text x="380" y="200" class="muted">Bulls, cows, duplicates</text>

  <path class="arrow" d="M 300 175 L 360 175"/>

  <!-- Row 2 -->
  <rect x="40" y="290" width="260" height="130" class="box"/>
  <text x="60" y="325" class="text">Belief-State Update</text>
  <text x="60" y="350" class="muted">trim_list()</text>
  <text x="60" y="370" class="muted">Remove impossible words</text>

  <rect x="360" y="290" width="260" height="130" class="accent"/>
  <text x="380" y="325" class="text">Internal State</text>
  <text x="380" y="350" class="muted">Candidate words</text>
  <text x="380" y="370" class="muted">Guess history</text>

  <!-- Diagonal: Game Logic → Belief Update -->
  <path class="arrow" d="M 490 230 L 170 290"/>

  <path class="arrow" d="M 300 355 L 360 355"/>

  <!-- Row 3 -->
  <rect x="40" y="490" width="260" height="120" class="box"/>
  <text x="60" y="525" class="text">Policy Layer</text>
  <text x="60" y="550" class="muted">Heuristic scoring +</text>
  <text x="60" y="570" class="muted">optional stochastic selection</text>

  <rect x="360" y="490" width="260" height="120" class="box"/>
  <text x="380" y="525" class="text">LLM (Optional)</text>
  <text x="380" y="550" class="muted">Ranks valid guesses</text>
  <text x="380" y="570" class="muted">Language priors</text>

  <!-- Diagonal: Internal State → Policy -->
  <path class="arrow" d="M 490 420 L 170 490"/>

  <path class="arrow" d="M 300 550 L 360 550"/>

  <!-- Validation -->
  <rect x="180" y="650" width="320" height="90" class="note"/>
  <text x="200" y="690" class="text">Validity Check</text>
  <text x="200" y="710" class="muted">Ensure guess ∈ candidates</text>

  <!-- Diagonal: LLM → Validation -->
  <path class="arrow" d="M 490 610 L 360 650"/>
</svg>
>


### Step 1
**Find a list of all the possible solutions for Wordle.**  
I used the list provided here - [https://github.com/tabatkins/wordle-list](https://github.com/tabatkins/wordle-list). A solution was picked at random from the above list of 14,855 words. These were taken from the game's source code.

### Step 2
**After each guess, track the Bulls, Cows and Absent letters.**  
Here's where I ran into my first problem. Absent letters don't necessarily imply that the letter is not in the word, but they are position-specific. Meaning that if I guess the word "ALLEY", the first "L" might show yellow while the second "L" shows gray. The tool needs to know to not ignore "L" because it would be gray. I had to keep my feedback order fixed. Ensure that a letter is considered absent **ONLY** if it is not a bull or a cow.  

### Step 3
**The solution space is then reduced.**  
Words containing letters that are truly absent (i.e., not bulls or cows anywhere) are removed. I enforced Hard Mode-style constraints to simplify the reasoning system. Also, the remaining solution space will not contain words where the Bulls are not in the correct position or the Cows that are in the same position as in the guess. I had to implement this because when I was giving the LLM access to the guessed word and the feedback, it would:
- Produce illegal words
- Violate constraints
- Give confident but false reasoning

For the false reasoning, it seemed like the LLM would generate a plausible guess (either right or wrong) and then try to give an explanation for why it picked that word (which would be wholly incorrect).  
For example, the guess was 'naled' (yeah, a real word accepted by Wordle) and the feedback was \[1,2,0,0,0\]  
The agent suggested the next guess to be 'nasty' and gave the reasoning as below:  
This guess satisfies the feedback rules as follows:  
- The letter 'a' is in the correct position (index 1), which is maintained in this guess.  
- The letter 'n' is included but in a different position (index 0), which is required since it received a feedback of 1 in the previous guess.  
- The letters 'l', 'e', and 'd' are not included in this guess, as they received a feedback of 0, indicating they are not in the solution.  
- The guess contains common letters like 's' and 't', which may help in identifying the solution.

The agent is confident about its answer but the letter 'n' is in the same position it was before.  
The model wasn’t reasoning, it was choosing a guess first and fabricating justification afterward.  
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

<svg xmlns="http://www.w3.org/2000/svg" width="680" height="820" viewBox="0 0 680 820">
  <defs>
    <style>
      .box { fill: #ffffff; stroke: #111827; stroke-width: 2; rx: 14; ry: 14; }
      .accent { fill: #EEF2FF; stroke: #6366F1; stroke-width: 2; rx: 14; ry: 14; }
      .warn { fill: #FFFBEB; stroke: #F59E0B; stroke-width: 2; rx: 14; ry: 14; }
      .title { font: 700 22px ui-sans-serif; fill: #111827; }
      .text { font: 600 16px ui-sans-serif; fill: #111827; }
      .muted { font: 500 14px ui-sans-serif; fill: #374151; }
      .arrow { stroke: #111827; stroke-width: 1.8; fill: none; marker-end: url(#arrowhead); }
    </style>
    <marker id="arrowhead" markerWidth="10" markerHeight="10" refX="8" refY="5" orient="auto">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#111827"/>
    </marker>
  </defs>

  <text x="40" y="50" class="title">Agent Turn Loop</text>
  <text x="40" y="80" class="muted">Guess → Feedback → Trim → LLM Suggestion → Validate → Next Guess</text>

  <!-- Guess -->
  <rect x="180" y="120" width="320" height="90" class="accent"/>
  <text x="205" y="160" class="text">Guess</text>

  <!-- Feedback -->
  <rect x="180" y="240" width="320" height="90" class="box"/>
  <text x="205" y="280" class="text">Feedback</text>

  <!-- Trim -->
  <rect x="180" y="360" width="320" height="90" class="box"/>
  <text x="205" y="400" class="text">Trim Candidates</text>

  <!-- LLM -->
  <rect x="180" y="480" width="320" height="90" class="box"/>
  <text x="205" y="520" class="text">LLM Suggestion</text>

  <!-- Validate -->
  <rect x="180" y="600" width="320" height="90" class="box"/>
  <text x="205" y="640" class="text">Validate Guess</text>

  <!-- Retry -->
  <rect x="180" y="710" width="320" height="80" class="warn"/>
  <text x="205" y="750" class="text">Retry / Fallback if invalid</text>

  <!-- Longer, lighter down arrows -->
  <path class="arrow" d="M 340 210 L 340 250"/>
  <path class="arrow" d="M 340 330 L 340 370"/>
  <path class="arrow" d="M 340 450 L 340 490"/>
  <path class="arrow" d="M 340 570 L 340 610"/>
  <path class="arrow" d="M 340 690 L 340 730"/>

  <!-- Loop arrow around right side -->
  <path class="arrow" d="M 500 750 L 620 750 L 620 165 L 500 165"/>
</svg>


The architectural roles of each module:  

| Layer          | What it does                                                                               | Why it exists                                                                                |
| -------------- | ------------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------- |
| `get_feedback` | Computes Wordle feedback (bulls, cows, absent letters) given a guess and the true solution | Acts as the **environment model**, defining the rules of the game and producing observations |
| `trim_list`    | Filters the candidate word list using feedback constraints                                 | Performs the **belief-state update**, removing impossible worlds based on new evidence       |
| Candidate list | Stores all words still consistent with observed feedback                                   | Represents the agent’s **internal state** — its current belief about the hidden solution     |
| LLM            | Suggests the next guess from the valid candidates                                          | Serves as a **policy proposal mechanism**, using language priors to guide action selection   |


This is the agent structure.  
At this point, the system has all the components of a classical AI agent:
- an internal state (candidate list)
- a perception-update loop (feedback -> trim)
- a policy for action selection  
The intelligence emerges from the architecture, not any single component.

## The Real Role of the LLM
LLMs operate on probability, not symbolic rules. They’re excellent at proposing linguistically plausible guesses, but they don’t reliably enforce hard constraints, which is why deterministic validation must remain in our control. So in this system, the LLM acts as a:  
- Heuristic ranker
- Proposal generator

## What This Project Really Shows  
This work is not primarily about using an LLM to solve Wordle. Rather, it illustrates an agent architecture in which language models provide heuristic guidance while symbolic reasoning enforces correctness. The type of reasoning contributed by each component:  

| Component     | Type            |
| ------------- | --------------- |
| Game logic    | Deterministic   |
| Belief update | Deterministic   |
| Policy        | Stochastic      |
| LLM           | Heuristic prior |

If the LLM is removed, the system still works but it simply becomes fully rule-based. However, if the reasoning engine is removed, the agent loses the ability to operate meaningfully at all.

## Lessons Learned Building Agents
The biggest lesson from this project is that agentic behavior isn’t magic, it’s architecture. Once you separate the system into:
1. A deterministic world model
2. A belief state
3. A policy that chooses actions  
you get something that behaves like an agent almost automatically.

The second lesson is less comfortable, LLMs are not rule engines. They're powerful heuristic guides, but they will confidently suggest illegal actions unless the system makes those actions impossible. The safest and most reliable pattern (used across mature AI systems) is simple. The model proposes and deterministic code verifies.

Finally, debugging Wordle reinforced a lesson that applies to building agents in general. Most of the real difficulty lies in the "boring" parts, i.e., state representation, edge cases, and constraint enforcement. Once those foundations are correct, you can layer in heuristic scoring, stochastic policies, or LLM-based ranking, and everything else becomes easier because the system has a solid base to stand on.

## Conclusion
Wordle turned out to be a small, controlled version of a much bigger story in AI. Language models are extraordinary at generating possibilities, but reliable systems are built on structure, i.e., world models, state, and rules that don't bend. The intelligence of an agent doesn't live in any single component, it emerges from how deterministic reasoning and probabilistic models are combined. Wordle just made that lesson impossible to ignore.
