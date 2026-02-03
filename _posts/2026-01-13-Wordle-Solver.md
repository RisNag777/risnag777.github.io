---
layout: post
title: "What Building a Wordle Solver Taught Me About AI Agents"
---

Like most of the human race, towards the end of COVID, I was obsessed with Wordle. However, over the last year, after multiple broken streaks, I eventually fell off the wagon. Despite that, Wordle continues to be a big part of my life. My wife plays Wordle religiously every morning and our daily ritual is to play it together in bed before we kickstart our day. Sometimes, we're completely stumped and that's when my phone comes into play to 'unlock' an additional 5 chances!  

During those moments, I started thinking about what it would actually mean for a machine to play Wordle well. It wouldn’t memorize patterns (that’s not how the game works), but it would maintain beliefs about possible solutions, update those beliefs with evidence from feedback, and choose its next action strategically. In other words, it would behave like an AI agent.

Naturally, I decided to build an agentic Wordle solver! My goal was to have a system inspired by Wordle's own WordleBot, an algorithm that evaluates guesses based on how effectively they shrink the remaining solution space. Rather than hard-coding heuristics, I wanted to explore whether an agent could reason its way into selecting the next best guess at each step.  

## System Design
Wordle Is a Belief-State Search Problem. A good Wordle solver does not gues. It updates beliefs.  
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

This reframes Wordle as a sequential inference problem rather than a language task.  

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
I used the list provided here - [https://gist.github.com/cfreshman/a03ef2cba789d8cf00c08f767e0fad7b](https://gist.github.com/cfreshman/a03ef2cba789d8cf00c08f767e0fad7b). A solution was picked at random from the above list of 2,315 words. These were taken from the game's source code.

### Step 2
**After each guess, track the Bulls, Cows and Absent letters.**  
One of the most subtle bugs I encountered was handling duplicate letters correctly. Consider guessing "ALLEY" against solution "BALKS". The first 'L' at position 2 gets feedback 0 (gray), while the second 'L' at position 3 gets feedback 2 (green). This doesn't mean 'L' is absent, but means 'L' cannot be at position 2, but must be at position 3. The system tracks these as `excluded_positions`: letters that are in the word but forbidden at specific positions. Without this, the belief update would incorrectly eliminate valid candidates like "BALKS" because it saw a gray 'L' and assumed the letter was completely absent.


### Step 3
**The solution space is then reduced.**  
The get_feedback() function processes feedback in two critical passes. First, it identifies bulls (exact matches) and decrements the solution's letter count for each match. Only then does it process cows (correct letter, wrong position). This ordering is important because if cows are processed first, duplicate letters can be double-counted.  
For example, guessing "EERIE" against "CRANE" would incorrectly mark both E's as cows, even though only one E exists in the solution. The two-pass approach ensures each letter in the solution is matched at most once.

Once feedback is computed correctly, the solution space is reduced. Words containing letters that are truly absent (i.e., not bulls or cows anywhere) are removed. I enforced Hard Mode-style constraints to simplify the reasoning system. The remaining candidates must keep bulls in their exact positions and not place cows in the same positions as before. This logic was necessary because when the LLM was allowed to propose guesses directly from feedback, it would:  
- Produce illegal words
- Violate constraints
- Give confident but false reasoning

The model would generate a plausible guess first, then fabricate an explanation to justify it. For example, after guessing "naked" with feedback \[1,2,0,0,0\], the agent proposed "nasty" and claimed:
- "a" remains in the correct position
- "n" is included but in a different position
- "k", "e", "d" are absent

But "n" is still in the same position as before, violating the constraint it claimed to follow. The model wasn’t reasoning. It was choosing a guess first and inventing a justification afterward. This made one thing clear to me. The LLM must never own the game logic. Deterministic code must enforce rules, otherwise correctness becomes probabilistic.

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

The agent implements a retry mechanism that perfectly illustrates the "propose and verify" pattern. When the LLM suggests a guess, the system doesn't trust it blindly. Instead, it:  
1. Extracts the guess from the LLM's response (which might be embedded in prose, JSON, or code blocks)
2. Validates that the guess exists in the filtered candidate list
3. If invalid, feeds the error back to the LLM with historical constraints and retries (up to 5 attempts)
4. If all retries fail, falls back to a random valid candidate  

```
valid_check = 0
while valid_guess:
    valid_check += 1
    completion = client.chat.completions.create(
        model="gpt-4o-mini", messages=messages, temperature=0
    )
    tmp_guess = extract_guess(ai_response_content)
    if tmp_guess in candidates:
        guess = tmp_guess
        break
    else:
        if valid_check == 5:
            guess = random.choice(candidates)  # Fallback
            break
        else:
            # Feed error back to LLM and retry
            invalid_guess_prompt = f"""Your guess {tmp_guess} is not valid..."""
```
This loop is where the deterministic validation layer enforces correctness. The LLM might confidently suggest "NATSY" (not a real word) or "NASTY" with letters in forbidden positions. Even with temperature=0, constraint violations could still occur. The model is consistent, but not rule-aware. The system catches these violations and forces correction. If the model cannot produce a valid action after multiple attempts, control reverts to the deterministic layer via a guaranteed-valid fallback.

The architectural roles of each module:  

| Layer          | What it does                                                                               | Why it exists                                                                                |
| -------------- | ------------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------- |
| `get_feedback` | Computes Wordle feedback (bulls, cows, absent letters) given a guess and the true solution | Acts as the **environment model**, defining the rules of the game and producing observations |
| `trim_list`    | Filters the candidate word list using feedback constraints                                 | Performs the **belief-state update**, removing impossible worlds based on new evidence       |
| Candidate list | Stores all words still consistent with observed feedback                                   | Represents the agent’s **internal state** — its current belief about the hidden solution     |
| LLM            | Suggests the next guess from the valid candidates                                          | Serves as a **policy proposal mechanism**, using language priors to guide action selection   |


This is what makes the system a true agent rather than a generative script. It has:  
- an internal state (candidate list)
- a perception-update loop (feedback -> trim)
- a policy for proposing actions
- a verification layer that enforces rules before actions are executed
The intelligence emerges from the architecture, not from the LLM alone.

## The Real Role of the LLM
LLMs operate on probability, not symbolic rules. They’re excellent at proposing linguistically plausible guesses, but they don't reliably enforce hard constraints, which is why deterministic validation must remain in our control. In this system, the LLM acts as a:  
- Heuristic ranker
- Proposal generator

But there's another layer of reality to deal with. LLMs don't speak in APIs, they speak in language. Even with explicit instructions, the model doesn't always return clean structured data. A guess might appear as "guess: baker" or "'guess': 'baker'" either buried inside a paragraph or wrapped in a code block. The extract_guess() function uses multiple regex patterns to parse these variations and recover the intended word. This parsing step is itself a form of validation. It accepts that model outputs are probabilistic language artifacts, not guaranteed structured responses.  

So the system has to be robust in two ways:  
- Format robustness: correctly extracting the intended action from messy language
- Constraint robustness: ensuring the extracted guess actually obeys game rules

Together, these layers make the difference between a system that asks a model to play Wordle and a system that controls how a model is allowed to act.

## What This Project Really Shows  
This work is not primarily about using an LLM to solve Wordle. Rather, it illustrates an agent architecture in which language models provide heuristic guidance while symbolic reasoning enforces correctness. The type of reasoning contributed by each component:  

| Component     | Type            |
| ------------- | --------------- |
| Game logic    | Deterministic   |
| Belief update | Deterministic   |
| Policy        | Stochastic      |
| LLM           | Heuristic prior |

If the LLM is removed, the system still works but it simply becomes fully rule-based. However, if the reasoning engine is removed, the agent loses the ability to operate meaningfully at all.

## Implementation Details
**Candidate Sampling Strategy**  
Rather than sending the entire candidate list to the LLM (which could be thousands of words), the system samples 20 random candidates to provide context. This serves two purposes: it keeps the prompt manageable, and it gives the LLM examples of valid words without overwhelming it. The LLM uses these examples to understand the style and constraints, then generates its own guess from the full candidate space.

**Feedback Explanation Formatting**  
The `feedback_explanation()` function structures the game state into a human-readable format for the LLM. It categorizes letters into bulls, cows, and absent, and explicitly states position constraints. This formatting is crucial. Raw feedback arrays like `[1, 2, 0, 0, 0]` are opaque, but structured explanations help the LLM reason about constraints. The system is essentially translating between two representations: the internal state (arrays, sets, dictionaries) and the language representation (natural language explanations) that the LLM can process.

**Hard Mode Enforcement**  
The system enforces Hard Mode-style constraints: once a letter is identified as a bull or cow, all future guesses must respect those constraints. This simplifies the reasoning system by making the constraints cumulative and explicit. Without this, the LLM might suggest guesses that ignore previous feedback, requiring even more complex validation logic.

## Lessons Learned Building Agents
The biggest lesson from this project is that agentic behavior isn't magic, it's architecture. Once you separate the system into:
- A deterministic world model
- A belief state
- A policy that chooses actions  

you get something that behaves like an agent almost automatically.  
The second lesson is less comfortable: LLMs are not rule engines. They're powerful heuristic guides, but they will confidently suggest illegal actions unless the system makes those actions impossible. The safest and most reliable pattern, used across mature AI systems, is simple: the model proposes, and deterministic code verifies.

In this project, that principle shows up directly in the retry loop. When the LLM violates constraints, the system doesn't accept the guess or fail silently. Instead, it:
- Catches the violation deterministically (if tmp_guess in candidates)
- Provides feedback to the LLM about what went wrong
- Gives the model another chance to correct itself
- Falls back to a random valid guess if correction fails

This propose -> validate -> retry -> fallback pattern is what makes the system reliable. The LLM's role is to suggest good guesses, but the system's role is to ensure those guesses are valid. Without this separation, every LLM mistake becomes a system failure.  

Finally, debugging Wordle reinforced a lesson that applies to building agents in general: most of the real difficulty lies in the "boring" parts, i.e., state representation, edge cases, and constraint enforcement. Once those foundations are correct, you can layer in heuristic scoring, stochastic policies, or LLM-based ranking, and everything else becomes easier because the system has a solid base to stand on.

## Conclusion
Wordle turned out to be a small, controlled version of a much bigger story in AI. Language models are extraordinary at generating possibilities, but reliable systems are built on structure i.e., world models, state, and rules that don't bend. The code itself reflects this architecture. The deterministic components (`get_feedback`, `trim_list`) contain no randomness and are fully testable. The LLM interaction (`wordle_agent`) sits on top of those functions inside a retry loop that handles uncertainty and constraint violations. The intelligence isn't in any single function but it's in how these components are composed.  

The system works because the foundation is solid, not because any individual piece is particularly clever. Wordle just made that lesson impossible to ignore.
