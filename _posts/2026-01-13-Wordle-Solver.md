---
layout: post
title: "What Building a Wordle Solver Taught Me About AI Agents"
---

Like most of the human race, towards the end of COVID, I was obsessed with Wordle. However, over the last year, after multiple broken streaks, I eventually fell off the wagon. Despite that, Wordle continues to be a big part of my life. My wife plays Wordle religiously every morning and our daily ritual is to play it together in bed before we kickstart our day. Sometimes, we're completely stumped and that's when my phone comes into play to 'unlock' an additional 5 chances!  

During those moments, I started thinking about what it would actually mean for a machine to play Wordle well. It wouldn’t memorize patterns (that’s not how the game works), but it would maintain beliefs about possible solutions, update those beliefs with evidence from feedback, and choose its next action strategically. Naturally, I decided to build an agentic Wordle solver! Rather than hard-coding heuristics, I wanted to explore whether an agent could reason its way into selecting the next best guess at each step.  

## System Design
Wordle is a Belief-State Search Problem. A good Wordle solver is not making guesses. It is updating its beliefs.  
At a high level, the system follows this loop:  
1. Pick a word at random (5-letter word)  
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
<svg xmlns="http://www.w3.org/2000/svg" width="900" height="1000" viewBox="0 0 900 1000">
  <defs>
    <style>
      .title { font: 700 24px ui-sans-serif; fill: #111827; }
      .subtitle { font: 500 14px ui-sans-serif; fill: #6B7280; }
      .component-title { font: 600 16px ui-sans-serif; fill: #111827; }
      .component-text { font: 500 13px ui-sans-serif; fill: #374151; }
      .function-name { font: 600 14px ui-mono, monospace; fill: #1E40AF; }
      .data-label { font: 500 12px ui-sans-serif; fill: #059669; }
      .deterministic { fill: #EEF2FF; stroke: #4F46E5; stroke-width: 2.5; rx: 12; ry: 12; }
      .stochastic { fill: #FFF7ED; stroke: #F59E0B; stroke-width: 2.5; stroke-dasharray: 5,3; rx: 12; ry: 12; }
      .data-box { fill: #F0FDF4; stroke: #10B981; stroke-width: 2; rx: 8; ry: 8; }
      .arrow { stroke: #111827; stroke-width: 2; fill: none; marker-end: url(#arrowhead); }
      .arrow-data { stroke: #059669; stroke-width: 2; fill: none; marker-end: url(#arrowhead-green); }
      .arrow-error { stroke: #DC2626; stroke-width: 2; fill: none; stroke-dasharray: 3,3; marker-end: url(#arrowhead-red); }
      .loop-box { fill: #FEF3C7; stroke: #F59E0B; stroke-width: 2; rx: 10; ry: 10; }
    </style>
    <marker id="arrowhead" markerWidth="10" markerHeight="10" refX="9" refY="5" orient="auto">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#111827"/>
    </marker>
    <marker id="arrowhead-green" markerWidth="10" markerHeight="10" refX="9" refY="5" orient="auto">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#059669"/>
    </marker>
    <marker id="arrowhead-red" markerWidth="10" markerHeight="10" refX="9" refY="5" orient="auto">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#DC2626"/>
    </marker>
  </defs>

  <!-- Title -->
  <text x="450" y="40" class="title" text-anchor="middle">Wordle Agent System Architecture</text>
  <text x="450" y="65" class="subtitle" text-anchor="middle">Deterministic Core + Stochastic Policy Layer</text>

  <!-- Legend -->
  <rect x="50" y="90" width="200" height="80" fill="#F9FAFB" stroke="#D1D5DB" stroke-width="1" rx="8"/>
  <text x="60" y="115" class="component-title">Legend</text>
  <rect x="60" y="125" width="60" height="20" class="deterministic"/>
  <text x="130" y="140" class="component-text">Deterministic</text>
  <rect x="60" y="150" width="60" height="20" class="stochastic"/>
  <text x="130" y="165" class="component-text">Stochastic/LLM</text>

  <!-- Data Source -->
  <rect x="50" y="200" width="180" height="100" class="data-box"/>
  <text x="140" y="230" class="component-title" text-anchor="middle">Data Source</text>
  <text x="140" y="255" class="function-name" text-anchor="middle">retrieve_word_list()</text>
  <text x="140" y="275" class="component-text" text-anchor="middle">words.txt (2,315 words)</text>
  <text x="140" y="290" class="component-text" text-anchor="middle">Initial candidates</text>

  <!-- Game Logic Module -->
  <rect x="300" y="200" width="200" height="180" class="deterministic"/>
  <text x="400" y="230" class="component-title" text-anchor="middle">Game Logic Module</text>
  <text x="400" y="255" class="function-name" text-anchor="middle">get_feedback()</text>
  <text x="400" y="275" class="component-text" text-anchor="middle">Two-pass algorithm:</text>
  <text x="400" y="290" class="component-text" text-anchor="middle">1. Process bulls (2)</text>
  <text x="400" y="305" class="component-text" text-anchor="middle">2. Process cows (1)</text>
  <text x="400" y="320" class="component-text" text-anchor="middle">3. Mark absent (0)</text>
  <text x="400" y="345" class="function-name" text-anchor="middle">cow_bull_absent()</text>
  <text x="400" y="365" class="component-text" text-anchor="middle">Categorize feedback</text>

  <!-- Belief Update Module -->
  <rect x="550" y="200" width="200" height="180" class="deterministic"/>
  <text x="650" y="230" class="component-title" text-anchor="middle">Belief Update Module</text>
  <text x="650" y="255" class="function-name" text-anchor="middle">trim_list()</text>
  <text x="650" y="275" class="component-text" text-anchor="middle">Filters candidates by:</text>
  <text x="650" y="290" class="component-text" text-anchor="middle">• Absent letters</text>
  <text x="650" y="305" class="component-text" text-anchor="middle">• Bull positions</text>
  <text x="650" y="320" class="component-text" text-anchor="middle">• Cow constraints</text>
  <text x="650" y="335" class="component-text" text-anchor="middle">• Excluded positions</text>
  <text x="650" y="360" class="function-name" text-anchor="middle">filter_candidates()</text>
  <text x="650" y="375" class="component-text" text-anchor="middle">Generic filter helper</text>

  <!-- Internal State -->
  <rect x="300" y="420" width="450" height="80" class="data-box"/>
  <text x="525" y="450" class="component-title" text-anchor="middle">Internal State (Belief State)</text>
  <text x="525" y="475" class="component-text" text-anchor="middle">candidates: List[str] - Words consistent with all feedback</text>
  <text x="525" y="495" class="component-text" text-anchor="middle">history: Dict[str, List[int]] - Guess → Feedback mapping</text>

  <!-- Policy Layer - Sampling -->
  <rect x="50" y="540" width="200" height="100" class="deterministic"/>
  <text x="150" y="570" class="component-title" text-anchor="middle">Policy: Sampling</text>
  <text x="150" y="595" class="function-name" text-anchor="middle">random_word_select()</text>
  <text x="150" y="615" class="component-text" text-anchor="middle">Sample 20 candidates</text>
  <text x="150" y="630" class="component-text" text-anchor="middle">for LLM context</text>

  <!-- Policy Layer - Formatting -->
  <rect x="300" y="540" width="200" height="100" class="deterministic"/>
  <text x="400" y="570" class="component-title" text-anchor="middle">Policy: Formatting</text>
  <text x="400" y="595" class="function-name" text-anchor="middle">feedback_explanation()</text>
  <text x="400" y="615" class="component-text" text-anchor="middle">Structure game state</text>
  <text x="400" y="630" class="component-text" text-anchor="middle">for LLM consumption</text>

  <!-- LLM Layer -->
  <rect x="550" y="520" width="200" height="140" class="stochastic"/>
  <text x="650" y="550" class="component-title" text-anchor="middle">LLM Policy</text>
  <text x="650" y="575" class="function-name" text-anchor="middle">OpenAI GPT-4o-mini</text>
  <text x="650" y="595" class="component-text" text-anchor="middle">Temperature: 0</text>
  <text x="650" y="615" class="component-text" text-anchor="middle">Role: Heuristic ranker</text>
  <text x="650" y="635" class="component-text" text-anchor="middle">Proposes next guess</text>
  <text x="650" y="655" class="component-text" text-anchor="middle">from valid candidates</text>

  <!-- Validation Layer -->
  <rect x="300" y="670" width="200" height="120" class="deterministic"/>
  <text x="400" y="700" class="component-title" text-anchor="middle">Validation Layer</text>
  <text x="400" y="725" class="function-name" text-anchor="middle">extract_guess()</text>
  <text x="400" y="745" class="component-text" text-anchor="middle">Parse LLM response</text>
  <text x="400" y="765" class="component-text" text-anchor="middle">(regex patterns)</text>
  <text x="400" y="780" class="function-name" text-anchor="middle">if guess in candidates</text>

  <!-- Retry Loop -->
  <rect x="550" y="700" width="200" height="120" class="loop-box"/>
  <text x="650" y="730" class="component-title" text-anchor="middle">Retry Loop</text>
  <text x="650" y="755" class="component-text" text-anchor="middle">Max 5 attempts</text>
  <text x="650" y="775" class="component-text" text-anchor="middle">Feed error back to LLM</text>
  <text x="650" y="795" class="component-text" text-anchor="middle">Fallback: random choice</text>
  <text x="650" y="810" class="component-text" text-anchor="middle">if all retries fail</text>

  <!-- Main Agent Loop -->
  <rect x="50" y="840" width="700" height="120" class="deterministic"/>
  <text x="400" y="900" class="component-title" text-anchor="middle">Agent guesses a word</text>

  <!-- Data Flow Arrows -->
  <!-- Data Source to Game Logic -->
  <path class="arrow-data" d="M 230 250 L 300 250"/>
  <text x="240" y="245" class="data-label">guess +</text>
  <text x="240" y="265" class="data-label">solution</text>

  <!-- Game Logic to Belief Update -->
  <path class="arrow-data" d="M 500 290 L 550 290"/>
  <text x="502" y="280" class="data-label">feedback</text>

  <!-- Belief Update to Internal State -->
  <path class="arrow-data" d="M 650 380 L 650 420"/>
  <text x="665" y="400" class="data-label">filtered candidates</text>

  <!-- Internal State to Policy Sampling -->
  <path class="arrow-data" d="M 300 460 L 200 460 L 200 520 L 200 540"/>
  <text x="220" y="450" class="data-label">candidates</text>

  <!-- Internal State to Policy Formatting -->
  <path class="arrow-data" d="M 400 500 L 400 540"/>
  <text x="405" y="520" class="data-label">history, feedback</text>

  <!-- Policy Sampling to LLM -->
  <path class="arrow" d="M 200 640 L 200 650 L 550 650"/>
  <text x="225" y="665" class="data-label">20 examples</text>

  <!-- Policy Formatting to LLM -->
  <path class="arrow" d="M 500 590 L 550 590"/>
  <text x="501" y="580" class="data-label">formatted</text>
  <text x="510" y="605" class="data-label">prompt</text>

  <!-- LLM to Validation -->
  <path class="arrow" d="M 650 660 L 650 680 L 500 680"/>
  <text x="540" y="675" class="data-label">LLM response</text>

  <!-- Validation to Retry Loop (invalid) -->
  <path class="arrow-error" d="M 500 750 L 550 750"/>
  <text x="505" y="740" class="data-label">invalid</text>
  <text x="505" y="760" class="data-label">guess</text>

  <!-- Retry Loop back to LLM -->
  <path class="arrow-error" d="M 680 700 L 680 660"/>
  <text x="690" y="685" class="data-label">error feedback</text>

  <!-- Validation to Agent Loop (valid) -->
  <path class="arrow-data" d="M 400 790 L 400 840"/>
  <text x="405" y="810" class="data-label">valid guess</text>

  <!-- Agent Loop back to Game Logic -->
  <path class="arrow-data" d="M 50 900 L 20 900 L 20 350 L 300 350"/>
  <text x="30" y="700" class="data-label">guess</text>

  <!-- Retry Loop to Agent Loop (fallback) -->
  <path class="arrow-error" d="M 750 800 L 800 800 L 800 900 L 750 900"/>
  <text x="810" y="850" class="data-label">fallback</text>

</svg>



### Step 1
**Find a list of all the possible solutions for Wordle**  
I used the list provided here - [https://gist.github.com/cfreshman/a03ef2cba789d8cf00c08f767e0fad7b](https://gist.github.com/cfreshman/a03ef2cba789d8cf00c08f767e0fad7b). A solution was picked at random from the above list of 2,315 words. These were taken from the game's source code.

### Step 2
**Handling Duplicate Letters and Position Constraints**  
One of the most subtle bugs I encountered was handling duplicate letters correctly. Consider guessing "ALLEY" against solution "BALKS". The first 'L' at position 2 gets feedback 0 (gray), while the second 'L' at position 3 gets feedback 2 (green). This doesn't mean 'L' is absent, just that it cannot be at position 2, but at position 3. The system tracks these as `excluded_positions`: letters that are in the word but forbidden at specific positions. Without this, the belief update would incorrectly eliminate valid candidates like "BALKS" because it saw a gray 'L' and assumed the letter was completely absent.


### Step 3
**Two-Pass Feedback and Deterministic Constraint Enforcement**  
The `get_feedback()` function processes feedback in two critical passes. First, it identifies bulls (exact matches) and decrements the solution's letter count for each match. Only then does it process cows (correct letter, wrong position). This ordering is important because if cows are processed first, duplicate letters can be double-counted.  

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
guess = random.choice([w for w in candidates if w != solution])

for turn in range(6):
    feedback = get_feedback(guess, solution)
    history[guess] = feedback
    
    if guess == solution:
        print("Victory!")
        break
    
    candidates = trim_list(guess, feedback, candidates)
    random_candidates = random_word_select(candidates, num_words=20)
    
    # Format prompt and get LLM suggestion
    prompt = feedback_explanation(turn, guess, feedback)
    # ... LLM interaction ...
    tmp_guess = extract_guess(ai_response_content)
    
    # Validation and retry loop
    if tmp_guess in candidates:
        guess = tmp_guess
    else:
        # Retry with error feedback (max 5 attempts)
        # Fallback to random if all retries fail
```

<svg xmlns="http://www.w3.org/2000/svg" width="900" height="1200" viewBox="-100 0 1000 1000">
  <defs>
    <style>
      .title { font: 700 28px ui-sans-serif; fill: #111827; }
      .subtitle { font: 500 16px ui-sans-serif; fill: #6B7280; }
      .step-title { font: 600 18px ui-sans-serif; fill: #111827; }
      .step-text { font: 500 14px ui-sans-serif; fill: #374151; }
      .function-name { font: 600 14px ui-mono, monospace; fill: #1E40AF; }
      .data-label { font: 500 12px ui-sans-serif; fill: #059669; }
      .decision { font: 600 14px ui-sans-serif; fill: #DC2626; }
      .step-box { fill: #EEF2FF; stroke: #4F46E5; stroke-width: 2.5; rx: 12; ry: 12; }
      .decision-box { fill: #FEE2E2; stroke: #DC2626; stroke-width: 2.5; rx: 12; ry: 12; }
      .llm-box { fill: #FFF7ED; stroke: #F59E0B; stroke-width: 2.5; stroke-dasharray: 5,3; rx: 12; ry: 12; }
      .loop-box { fill: #FEF3C7; stroke: #F59E0B; stroke-width: 3; rx: 10; ry: 10; }
      .arrow { stroke: #111827; stroke-width: 2.5; fill: none; marker-end: url(#arrowhead); }
      .arrow-data { stroke: #059669; stroke-width: 2; fill: none; marker-end: url(#arrowhead-green); }
      .arrow-error { stroke: #DC2626; stroke-width: 2; fill: none; stroke-dasharray: 3,3; marker-end: url(#arrowhead-red); }
      .arrow-loop { stroke: #F59E0B; stroke-width: 3; fill: none; marker-end: url(#arrowhead-orange); }
    </style>
    <marker id="arrowhead" markerWidth="10" markerHeight="10" refX="9" refY="5" orient="auto">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#111827"/>
    </marker>
    <marker id="arrowhead-green" markerWidth="10" markerHeight="10" refX="9" refY="5" orient="auto">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#059669"/>
    </marker>
    <marker id="arrowhead-red" markerWidth="10" markerHeight="10" refX="9" refY="5" orient="auto">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#DC2626"/>
    </marker>
    <marker id="arrowhead-orange" markerWidth="10" markerHeight="10" refX="9" refY="5" orient="auto">
      <path d="M 0 0 L 10 5 L 0 10 z" fill="#F59E0B"/>
    </marker>
  </defs>

  <!-- Title -->
  <text x="500" y="40" class="title" text-anchor="middle">Wordle Agent Loop</text>
  <text x="500" y="70" class="subtitle" text-anchor="middle">Iterative Decision-Making Process</text>

  <!-- Row 0: Initialization (0,0), Main Loop (0,1) -->
  <!-- Initialization (0,0) -->
  <rect x="50" y="100" width="180" height="130" class="step-box"/>
  <text x="140" y="130" class="step-title" text-anchor="middle">Initialization</text>
  <text x="140" y="155" class="step-text" text-anchor="middle">1. Load word list</text>
  <text x="140" y="175" class="function-name" text-anchor="middle">retrieve_word_list()</text>
  <text x="140" y="195" class="step-text" text-anchor="middle">2. Pick random solution</text>
  <text x="140" y="215" class="step-text" text-anchor="middle">3. Pick initial guess</text>

  <!-- Main Loop (0,1) -->
  <rect x="300" y="100" width="180" height="120" class="loop-box"/>
  <text x="390" y="130" class="step-title" text-anchor="middle">Main Loop</text>
  <text x="390" y="155" class="step-text" text-anchor="middle">For turn in range(6)</text>
  <text x="390" y="175" class="step-text" text-anchor="middle">Up to 6 turns</text>
  <text x="390" y="195" class="step-text" text-anchor="middle">to solve</text>

  <!-- Row 1: Step 1 (1,1), Step 2 (1,2), Victory (1,3) -->
  <!-- Step 1: Get Feedback (1,1) -->
  <rect x="270" y="280" width="180" height="130" class="step-box"/>
  <text x="360" y="310" class="step-title" text-anchor="middle">Step 1: Get Feedback</text>
  <text x="360" y="335" class="function-name" text-anchor="middle">get_feedback()</text>
  <text x="360" y="355" class="step-text" text-anchor="middle">Compare guess vs solution</text>
  <text x="360" y="375" class="step-text" text-anchor="middle">Returns [0,1,2] per position</text>
  <text x="360" y="395" class="step-text" text-anchor="middle">per position</text>

  <!-- Step 2: Check Win (1,2) -->
  <rect x="510" y="280" width="180" height="120" class="decision-box"/>
  <text x="600" y="320" class="step-title" text-anchor="middle">Step 2: Check Win</text>
  <text x="600" y="340" class="decision" text-anchor="middle">guess == solution?</text>
  <text x="600" y="365" class="step-text" text-anchor="middle">If YES: Victory! Exit</text>
  <text x="600" y="385" class="step-text" text-anchor="middle">If NO: Continue</text>

  <!-- Victory (1,3) -->
  <rect x="540" y="100" width="180" height="120" class="decision-box"/>
  <text x="630" y="130" class="step-title" text-anchor="middle">Victory!</text>
  <text x="630" y="155" class="step-text" text-anchor="middle">Print success message</text>
  <text x="630" y="175" class="step-text" text-anchor="middle">Display game history</text>
  <text x="630" y="195" class="step-text" text-anchor="middle">Exit loop</text>

  <!-- Row 2: Step 5 (2,0), Step 4 (2,1), Step 3 (2,2) -->
  <!-- Step 5: Format (2,0) -->
  <rect x="50" y="460" width="180" height="120" class="step-box"/>
  <text x="140" y="490" class="step-title" text-anchor="middle">Step 5: Format</text>
  <text x="140" y="515" class="function-name" text-anchor="middle">feedback_explanation()</text>
  <text x="140" y="535" class="step-text" text-anchor="middle">Structure game state</text>
  <text x="140" y="555" class="step-text" text-anchor="middle">Build LLM prompt</text>

  <!-- Step 4: Sample (2,1) -->
  <rect x="290" y="460" width="180" height="120" class="step-box"/>
  <text x="380" y="490" class="step-title" text-anchor="middle">Step 4: Sample</text>
  <text x="380" y="515" class="function-name" text-anchor="middle">random_word_select()</text>
  <text x="380" y="535" class="step-text" text-anchor="middle">Pick 20 random candidates</text>
  <text x="380" y="555" class="step-text" text-anchor="middle">for LLM context</text>

  <!-- Step 3: Update Belief (2,2) -->
  <rect x="530" y="460" width="180" height="120" class="step-box"/>
  <text x="620" y="490" class="step-title" text-anchor="middle">Step 3: Update Belief</text>
  <text x="620" y="515" class="function-name" text-anchor="middle">trim_list()</text>
  <text x="620" y="535" class="step-text" text-anchor="middle">Filter candidates by feedback</text>
  <text x="620" y="555" class="step-text" text-anchor="middle">Update history dict</text>

  <!-- Row 3: Step 6 (3,0), Step 7 (3,1), Step 8 (3,2) -->
  <!-- Step 6: LLM Generation (3,0) -->
  <rect x="50" y="650" width="200" height="120" class="llm-box"/>
  <text x="150" y="680" class="step-title" text-anchor="middle">Step 6: LLM Generation</text>
  <text x="150" y="705" class="function-name" text-anchor="middle">OpenAI GPT-4o-mini</text>
  <text x="150" y="725" class="step-text" text-anchor="middle">Temperature: 0</text>
  <text x="150" y="745" class="step-text" text-anchor="middle">Proposes next guess</text>
  <text x="150" y="760" class="step-text" text-anchor="middle">from valid candidates</text>

  <!-- Step 7: Extract & Validate (3,1) -->
  <rect x="310" y="650" width="220" height="120" class="step-box"/>
  <text x="420" y="680" class="step-title" text-anchor="middle">Step 7: Extract &amp; Validate</text>
  <text x="420" y="705" class="function-name" text-anchor="middle">extract_guess()</text>
  <text x="420" y="725" class="step-text" text-anchor="middle">Parse LLM response</text>
  <text x="420" y="740" class="step-text" text-anchor="middle">Check: guess in candidates?</text>
  <text x="420" y="755" class="step-text" text-anchor="middle">If invalid: retry (max 5x)</text>

  <!-- Step 8: Next Turn (3,2) -->
  <rect x="600" y="650" width="180" height="120" class="step-box"/>
  <text x="690" y="680" class="step-title" text-anchor="middle">Step 8: Next Turn</text>
  <text x="690" y="705" class="step-text" text-anchor="middle">Valid guess obtained</text>
  <text x="690" y="725" class="step-text" text-anchor="middle">Update guess variable</text>
  <text x="690" y="745" class="step-text" text-anchor="middle">Continue to Step 1</text>

  <!-- Row 4: Retry (4,0) -->
  <!-- Retry Loop (4,0) -->
  <rect x="50" y="840" width="180" height="130" class="loop-box"/>
  <text x="140" y="870" class="step-title" text-anchor="middle">Retry Loop</text>
  <text x="140" y="895" class="step-text" text-anchor="middle">If guess invalid:</text>
  <text x="140" y="915" class="step-text" text-anchor="middle">1. Feed error to LLM</text>
  <text x="140" y="935" class="step-text" text-anchor="middle">2. Regenerate (max 5x)</text>
  <text x="140" y="955" class="step-text" text-anchor="middle">3. Fallback: random choice</text>

  <!-- Flow Arrows -->
  <!-- Initialization to Main Loop -->
  <path class="arrow" d="M 230 160 L 300 160"/>
  
  <!-- Main Loop to Step 1 -->
  <path class="arrow" d="M 400 220 L 400 280"/>
  
  <!-- Step 1 to Step 2 -->
  <path class="arrow-data" d="M 450 300 L 510 300"/>
  <text x="455" y="285" class="data-label">feedback</text>
  
  <!-- Step 2 to Step 3 -->
  <path class="arrow" d="M 600 400 L 600 460"/>
  
  <!-- Step 2 to Victory -->
  <path class="arrow-data" d="M 610 280 L 610 220"/>
  <text x="620" y="256" class="data-label">solved!</text>
  
  <!-- Step 3 to Step 4 -->
  <path class="arrow-data" d="M 530 530 L 470 530"/>
  <text x="475" y="515" class="data-label">candidates</text>
  
  <!-- Step 4 to Step 5 -->
  <path class="arrow" d="M 290 530 L 230 530"/>
  
  <!-- Step 5 to Step 6 -->
  <path class="arrow" d="M 140 580 L 140 650"/>
  
  <!-- Step 6 to Step 7 -->
  <path class="arrow" d="M 250 700 L 310 700"/>
  <text x="255" y="690" class="data-label">LLM</text>
  <text x="255" y="720" class="data-label">response</text>
  
  <!-- Step 7 to Retry -->
  <path class="arrow-error" d="M 400 770 L 400 900 L 230 900"/>
  <text x="300" y="890" class="data-label">invalid</text>
  
  <!-- Retry to Step 6 -->
  <path class="arrow-error" d="M 140 840 L 140 770"/>
  <text x="150" y="810" class="data-label">error feedback</text>
  
  <!-- Step 7 to Step 8 -->
  <path class="arrow-data" d="M 530 700 L 600 700"/>
  <text x="540" y="690" class="data-label">valid</text>
  <text x="540" y="715" class="data-label">guess</text>
  
  <!-- Retry to Step 8 -->
  <path class="arrow-error" d="M 230 920 L 650 920 L 650 770"/>
  <text x="515" y="910" class="data-label">fallback</text>
  
  <!-- Step 8 to Main Loop -->
  <path class="arrow-loop" d="M 50 940 L 10 940 L 10 300 L 300 220"/>
  <text x="15" y="350" class="data-label">next turn</text>

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

But there's another layer of reality to deal with. LLMs don't speak in APIs, they speak in language. Even with explicit instructions, the model doesn't always return clean structured data. A guess might appear as "guess: baker" or "'guess': 'baker'" either buried inside a paragraph or wrapped in a code block. The `extract_guess()` function uses multiple regex patterns to parse these variations and recover the intended word. This parsing step is itself a form of validation. It accepts that model outputs are probabilistic language artifacts, not guaranteed structured responses.  

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
Rather than sending the entire candidate list to the LLM (which could be thousands of words), the system samples 20 random candidates to provide context. This serves two purposes:
- it keeps the prompt manageable
- it gives the LLM examples of valid words without overwhelming it  
The LLM uses these examples to understand the style and constraints, then generates its own guess from the full candidate space.

**Feedback Explanation Formatting**  
The `feedback_explanation()` function structures the game state into a human-readable format for the LLM. It categorizes letters into bulls, cows, and absent, and explicitly states position constraints. This formatting is crucial. Raw feedback arrays like `[1, 2, 0, 0, 0]` are opaque, but structured explanations help the LLM reason about constraints. The system is essentially translating between two representations:
- the internal state (arrays, sets, dictionaries)
- the language representation (natural language explanations) that the LLM can process

**Hard Mode Enforcement**  
The system enforces Hard Mode-style constraints. Once a letter is identified as a bull or cow, all future guesses must respect those constraints. This simplifies the reasoning system by making the constraints cumulative and explicit. Without this, the LLM might suggest guesses that ignore previous feedback, requiring even more complex validation logic.

## Lessons Learned Building Agents
The biggest lesson from this project is that agentic behavior isn't magic, it's architecture. Once you separate the system into:
- A deterministic world model
- A belief state
- A policy that chooses actions  

you get something that behaves like an agent almost automatically.  

The second lesson is less comfortable - LLMs are not rule engines. They're powerful heuristic guides, but they will confidently suggest illegal actions unless the system makes those actions impossible. The safest and most reliable pattern, used across mature AI systems, is simple. The model proposes, and deterministic code verifies.

In this project, that principle shows up directly in the retry loop. When the LLM violates constraints, the system doesn't accept the guess or fail silently. Instead, it:
- Catches the violation deterministically (if `tmp_guess` is not in candidates)
- Provides feedback to the LLM about what went wrong
- Gives the model another chance to correct itself
- Falls back to a random valid guess if correction fails

This `propose -> validate -> retry -> fallback` pattern is what makes the system reliable. The LLM's role is to suggest good guesses, but the system's role is to ensure those guesses are valid. Without this separation, every LLM mistake becomes a system failure.  

Finally, debugging Wordle reinforced a lesson that applies to building agents in general. Most of the real difficulty lies in the "boring" parts, i.e., state representation, edge cases, and constraint enforcement. Once those foundations are correct, you can layer in heuristic scoring, stochastic policies, or LLM-based ranking, and everything else becomes easier because the system has a solid base to stand on.

## Conclusion
Wordle turned out to be a small, controlled version of a much bigger story in AI. Language models are extraordinary at generating possibilities, but reliable systems are built on structure i.e., world models, state, and rules that don't bend. The code itself reflects this architecture. The deterministic components (`get_feedback`, `trim_list`) contain no randomness and are fully testable. The LLM interaction (`wordle_agent`) sits on top of those functions inside a retry loop that handles uncertainty and constraint violations. The intelligence isn't in any single function but it's in how these components are composed. The system works because the foundation is solid, not because any individual piece is particularly clever.
