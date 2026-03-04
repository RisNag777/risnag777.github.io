---
layout: post
title: "What Building a Wordle Solver Taught Me About AI Agents"
---

![Wordle](https://github.com/user-attachments/assets/bcf01716-7c22-461b-91e8-6162716f42d4)

## Introduction

Like most of the human race, towards the end of COVID, I was obsessed with Wordle. Over time my own streaks faded, but the game didn't really leave my life. My wife still plays Wordle religiously every morning, and our daily ritual is to solve it together in bed before we kickstart our day. On the days we're completely stumped, my phone quietly becomes our unofficial "six extra lives."

Those moments made me wonder what it would actually mean for a machine to play Wordle well. Not just memorizing answers, but behaving like a real **agent**: maintaining beliefs about possible solutions, updating those beliefs with evidence, and choosing the next action strategically. So I decided to build a Wordle-solving agent! Not by hard-coding heuristics, but by combining deterministic game logic with an LLM that proposes guesses under strict constraints.

This post walks through that journey: the core idea, the system architecture, the agent loop, what worked, what didn't, and how you can run it yourself. Along the way, I'll connect this small game to larger **operational AI** patterns I care about: building reliable, production-grade agents that don't hallucinate actions, even when they lean heavily on LLMs.

## The Core Idea

At its heart, Wordle is a **belief-state search problem**.

A good Wordle solver is, in reality, **updating its beliefs** about the hidden word. Each guess and its feedback carve away impossible words until only the true solution remains.

Reframed in those terms:

| Concept        | Wordle meaning                     |
| ------------- | ---------------------------------- |
| Hidden state  | The true solution word             |
| Belief state  | All words consistent with feedback |
| Action        | Choosing the next guess            |
| Observation   | Feedback (0,1,2 per letter)        |

The loop becomes:

1. Start with a list of all possible solutions (i.e., the 2,315 canonical Wordle answers)
2. Pick an initial 5-letter guess
3. Receive feedback:
   - `2` (Green / Bull): correct letter, correct position  
   - `1` (Yellow / Cow): correct letter, wrong position  
   - `0` (Gray / Absent): letter not in the word
4. Trim the solution space to only words consistent with all feedback so far
5. Choose the next guess from that filtered belief state
6. Repeat until the solution is found (≤ 6 guesses)

Instead of treating Wordle as a language modeling task (where the model predicts the next good word), the system treats it as **sequential inference**: maintaining a belief over possible words, observe feedback, update, and act.

## System Architecture

The architecture cleanly separates **deterministic game logic** from the **stochastic LLM policy**. The code is split into two main modules:

- `game_logic.py`: pure, testable, deterministic Wordle mechanics
- `wordle_agent.py`: the agent loop plus interaction with OpenAI's GPT-4o-mini

Here's the high-level architecture:

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
      .data_source { fill: #F0FDF4; stroke: #10B981; stroke-width: 2.5; rx: 12; ry: 12; }
      .fallback { fill: #FEF3C7; stroke: #F59E0B; stroke-width: 2.5; rx: 12; ry: 12; }
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
  <rect x="50" y="90" width="360" height="90" fill="#F9FAFB" stroke="#D1D5DB" stroke-width="1" rx="8"/>
  <text x="60" y="115" class="component-title">Legend</text>
  <rect x="60" y="125" width="60" height="20" class="deterministic"/>
  <text x="130" y="140" class="component-text">Deterministic</text>
  <rect x="60" y="150" width="60" height="20" class="stochastic"/>
  <text x="130" y="165" class="component-text">Stochastic/LLM</text>
  <rect x="260" y="125" width="60" height="20" class="data_source"/>
  <text x="330" y="140" class="component-text">Data Source</text>
  <rect x="260" y="150" width="60" height="20" class="fallback"/>
  <text x="330" y="165" class="component-text">Fallback</text>

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
  <rect x="570" y="200" width="200" height="180" class="deterministic"/>
  <text x="670" y="230" class="component-title" text-anchor="middle">Belief Update Module</text>
  <text x="670" y="255" class="function-name" text-anchor="middle">trim_list()</text>
  <text x="670" y="275" class="component-text" text-anchor="middle">Filters candidates by:</text>
  <text x="670" y="290" class="component-text" text-anchor="middle">• Absent letters</text>
  <text x="670" y="305" class="component-text" text-anchor="middle">• Bull positions</text>
  <text x="670" y="320" class="component-text" text-anchor="middle">• Cow constraints</text>
  <text x="670" y="335" class="component-text" text-anchor="middle">• Excluded positions</text>
  <text x="670" y="360" class="function-name" text-anchor="middle">filter_candidates()</text>
  <text x="670" y="375" class="component-text" text-anchor="middle">Generic filter helper</text>

  <!-- Internal State -->
  <rect x="300" y="420" width="450" height="80" class="data-box"/>
  <text x="525" y="440" class="component-title" text-anchor="middle">Internal State (Belief State)</text>
  <text x="525" y="465" class="component-text" text-anchor="middle">candidates: List[str] - Words consistent with all feedback</text>
  <text x="525" y="485" class="component-text" text-anchor="middle">history: Dict[str, List[int]] - Guess -> Feedback mapping</text>

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
  <rect x="570" y="520" width="200" height="140" class="stochastic"/>
  <text x="670" y="540" class="component-title" text-anchor="middle">LLM Policy</text>
  <text x="670" y="565" class="function-name" text-anchor="middle">OpenAI GPT-4o-mini</text>
  <text x="670" y="585" class="component-text" text-anchor="middle">Temperature: 0</text>
  <text x="670" y="605" class="component-text" text-anchor="middle">Role: Heuristic ranker</text>
  <text x="670" y="625" class="component-text" text-anchor="middle">Proposes next guess</text>
  <text x="670" y="645" class="component-text" text-anchor="middle">from valid candidates</text>

  <!-- Validation Layer -->
  <rect x="300" y="670" width="200" height="120" class="deterministic"/>
  <text x="400" y="700" class="component-title" text-anchor="middle">Validation Layer</text>
  <text x="400" y="725" class="function-name" text-anchor="middle">extract_guess()</text>
  <text x="400" y="745" class="component-text" text-anchor="middle">Parse LLM response</text>
  <text x="400" y="765" class="component-text" text-anchor="middle">(regex patterns)</text>
  <text x="400" y="780" class="function-name" text-anchor="middle">if guess in candidates</text>

  <!-- Retry Loop -->
  <rect x="570" y="700" width="200" height="120" class="loop-box"/>
  <text x="670" y="730" class="component-title" text-anchor="middle">Retry Loop</text>
  <text x="670" y="755" class="component-text" text-anchor="middle">Max 5 attempts</text>
  <text x="670" y="775" class="component-text" text-anchor="middle">Feed error back to LLM</text>
  <text x="670" y="795" class="component-text" text-anchor="middle">Fallback: random choice</text>
  <text x="670" y="810" class="component-text" text-anchor="middle">if all retries fail</text>

  <!-- Main Agent Loop -->
  <rect x="50" y="840" width="700" height="120" class="deterministic"/>
  <text x="400" y="900" class="component-title" text-anchor="middle">Agent guesses a word</text>

  <!-- Data Flow Arrows -->
  <!-- Data Source to Game Logic -->
  <path class="arrow-data" d="M 230 250 L 300 250"/>
  <text x="240" y="245" class="data-label">guess +</text>
  <text x="240" y="265" class="data-label">solution</text>

  <!-- Game Logic to Belief Update -->
  <path class="arrow-data" d="M 500 290 L 570 290"/>
  <text x="507" y="280" class="data-label">feedback</text>

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
  <path class="arrow" d="M 200 640 L 200 650 L 570 650"/>
  <text x="225" y="665" class="data-label">20 examples</text>

  <!-- Policy Formatting to LLM -->
  <path class="arrow" d="M 500 590 L 570 590"/>
  <text x="505" y="580" class="data-label">formatted</text>
  <text x="510" y="605" class="data-label">prompt</text>

  <!-- LLM to Validation -->
  <path class="arrow" d="M 650 660 L 650 680 L 500 680"/>
  <text x="540" y="675" class="data-label">LLM response</text>

  <!-- Validation to Retry Loop (invalid) -->
  <path class="arrow-error" d="M 500 750 L 570 750"/>
  <text x="510" y="740" class="data-label">invalid</text>
  <text x="510" y="760" class="data-label">guess</text>

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
  <path class="arrow-error" d="M 770 800 L 800 800 L 800 900 L 750 900"/>
  <text x="810" y="850" class="data-label">fallback</text>

</svg>

Conceptually:

- **Data source**: `retrieve_word_list()` loads all candidate solution words from `words.txt`
- **Game logic**: `get_feedback()` computes bulls/cows/absent; `cow_bull_absent()` categorizes letters and tracks excluded positions
- **Belief update**: `trim_list()` filters candidates based on feedback and constraints
- **Policy layer**:
  - Deterministic sampling: `random_word_select()` picks 20 candidates for context
  - LLM prompt formatting: `feedback_explanation()` describes state in natural language
  - LLM policy: GPT-4o-mini proposes the next guess
- **Validation layer**: `extract_guess()` parses the LLM's response; deterministic checks enforce that the guess is valid
- **Retry / fallback**: If the LLM repeatedly fails to produce a valid guess, the system falls back to a random valid candidate

### Key Components

#### 1. `retrieve_word_list()` (`game_logic.py`)

Loads the word list from `data/words.txt` (optionally controlled by a `DATA_FOLDER` env var). This defines the initial belief state: all words that could plausibly be the solution.

#### 2. `get_feedback()` (`game_logic.py`)

Implements Wordle's feedback rules using a **two-pass algorithm**:

1. First pass: mark all **bulls** (correct letter, correct position), updating a letter count for the solution so duplicates aren't double-counted
2. Second pass: mark **cows** (correct letter, wrong position) only where remaining letter counts allow it
3. Everything else is **absent**

This is the core *environment model*: given a hidden solution and a guess, produce the observable feedback.

#### 3. `cow_bull_absent()` (`game_logic.py`)

Takes a guess and its feedback and categorizes letters into:

- `bulls`: `{letter: [positions...]}`  
- `cows`: `{letter: [positions...]}`  
- `absent`: set of letters not in the solution  
- `excluded_positions`: list of `(letter, pos)` where a letter got `0` but is known to be in the word elsewhere (duplicate-letter edge cases)

Those `excluded_positions` are crucial for correctly handling tricky cases like `ALLEY` vs `BALKS`.

#### 4. `trim_list()` + `filter_candidates()` (`game_logic.py`)

`trim_list()` performs the belief-state update:

1. Remove words containing **truly absent letters**
2. Enforce **bull positions** (letters that must be in specific slots)
3. Enforce **cow constraints** (letters must be present but in different positions)
4. Enforce **excluded positions** for duplicate-letter edge cases

All of this is implemented using a generic `filter_candidates()` helper that applies predicates over candidate words.

#### 5. `random_word_select()` (`game_logic.py`)

Picks a small sample (default 20) from the remaining candidates to show the LLM:

- Keeps prompts manageable
- Gives the LLM a "shape" of the remaining space without dumping thousands of words

#### 6. `feedback_explanation()` (`wordle_agent.py`)

Takes the current guess and feedback and converts them into a human-readable explanation for the LLM:

- Which letters are absent
- Which letters are bulls (and at which indices)
- Which letters are cows (and which positions they must avoid)

This bridges the gap between the internal symbolic representation and the LLM's preferred language-based view.

#### 7. `extract_guess()` (`wordle_agent.py`)

LLMs don't always respond in pure JSON. This function:

- Searches the raw response (and any code blocks) with several regex patterns
- Extracts a 5-letter candidate word from structures like `guess: baker`, `"guess": "baker"`, etc.
- Normalizes everything to a lowercase string

It's the first line of defense against messy natural language output.

#### 8. `wordle_agent()` (`wordle_agent.py`)

The main agent loop:

- Initializes the candidate list and picks a random solution
- Repeats up to 6 turns:
  - Computes feedback for the current guess
  - Updates history and candidate list
  - Checks for victory
  - Formats a prompt with state and sample candidates
  - Calls GPT-4o-mini to propose the next guess
  - Validates the guess and uses a retry/fallback mechanism if needed

This function is where deterministic game logic and stochastic policy come together into agent behavior.

## Pipeline Flow

Put together, the system forms a clear, iterative decision loop:

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

At a high level, each turn does:

1. Compute feedback for the current guess
2. Check if the solution is found
3. Update the belief state (candidate list)
4. Sample example candidates and format the prompt
5. Ask the LLM to propose the next guess
6. Extract and validate the guess
7. Retry or fall back if needed
8. Loop to the next turn

### Detailed Pipeline Steps

1. **Initialization**
   - Load full word list via `retrieve_word_list()`
   - Choose a random solution
   - Choose an initial random guess different from the solution

2. **Step 1 – Get Feedback**
   - Use `get_feedback()` to compute `[0, 1, 2, 0, 2]`-style feedback
   - Update a `history` dictionary mapping guess -> feedback

3. **Step 2 – Check Win Condition**
   - If `guess == solution`:
     - Print a victory message and game history
     - Exit the loop

4. **Step 3 – Update Belief State**
   - Use `trim_list()` to enforce:
     - Absent-letter elimination
     - Bull positions
     - Cow presence and disallowed positions
     - Excluded positions from duplicate-letter logic
   - The remaining candidates are now the updated belief state

5. **Step 4 – Sample Candidates**
   - Call `random_word_select(candidates, num_words=20)`
   - Build a comma-separated list of example words for the LLM prompt

6. **Step 5 – Format the Prompt**
   - Use `feedback_explanation()` to narrate:
     - The current turn
     - The guess and feedback
     - Constraints implied by bulls, cows, and absent letters
   - Combine this with:
     - Rules of the game
     - Hard constraints (no absent letters, bulls locked, cows must move)
     - The sampled candidates

7. **Step 6 – LLM Generation**
   - Call GPT-4o-mini (`temperature=0`) with:
     - The previous guess and feedback explanation
     - The list of valid candidate examples
     - Explicit response format instructions

8. **Step 7 – Extract and Validate Guess**
   - Run `extract_guess()` on the LLM's reply
   - Check if the extracted word:
     - Is 5 letters
     - Exists in the `candidates` list
   - If valid: adopt it as the next `guess`
   - If invalid:
     - Append a system message explaining why it's invalid (including history)
     - Retry up to 5 times

9. **Step 8 – Fallback**
   - If all retries produce invalid guesses:
     - Log that the agent couldn't find a valid guess
     - Pick a random candidate as a safe fallback
   - Proceed to the next turn

This loop embodies a classic pattern: **propose -> validate -> retry -> fallback**.

## Technical Deep Dive

### Handling Duplicate Letters and Position Constraints

One of the hardest bugs to shake out was handling duplicate letters correctly.

Example: guess `ALLEY` vs solution `BALKS`

- The first `L` (position 1) might get feedback `0`  
- The second `L` (position 2) gets feedback `2` (bull)

This doesn't mean `L` is absent. It means:

- `L` exists in the word
- `L` cannot be at position 1
- `L` must be at position 2

`cow_bull_absent()` encodes this by:

- Adding `L` at position 2 to the `bulls` map
- Marking `(L, 1)` as an `excluded_position` instead of treating `L` as absent

If you treat every `0` as "this letter is not in the word," you'll incorrectly eliminate valid candidates like `BALKS` just because of the gray `L`.

Here's a simplified snippet of the `excluded_positions` logic:

```python
excluded_positions = set()

for i, (ch, fb) in enumerate(zip(guess, feedback)):
    # If this letter got 0 *but* we know it appears elsewhere,
    # treat it as position-excluded instead of truly absent.
    if fb == 0 and (ch in bulls or ch in cows):
        excluded_positions.add((ch, i))
```

### Two-Pass Feedback and Deterministic Constraint Enforcement

`get_feedback()` uses a two-pass algorithm for a reason:

- **First pass**: mark bulls and decrement a `Counter` of letters in the solution  
- **Second pass**: mark cows only if `solution_counts[letter] > 0`

Consider `EERIE` vs `CRANE`:

- Without a two-pass approach, you might:
  - Mark multiple Es as cows even though the solution has only one E
- With the correct two-pass approach:
  - Only one E gets credit
  - Others remain 0 or are treated as excluded positions

Once the feedback is correct, `trim_list()` can safely enforce **Hard Mode-style** constraints:

- Bulls stay fixed in future guesses
- Cows must move positions (until they become bulls)
- Absent letters can never reappear
- Duplicate letters with mixed feedback are handled via `excluded_positions`

Critically, **none of this logic lives in the LLM**. All constraint enforcement happens in deterministic Python code.

### The LLM's Role: Heuristic Policy, Not Judge

When I let the LLM "own" the logic early on, it behaved convincingly but incorrectly.

After guessing `"naked"` with feedback `[1, 2, 0, 0, 0]`, the model confidently proposed `"nasty"` and explained:

- `"a"` remains in the correct position  
- `"n"` is included but in a different position  
- `"k"`, `"e"`, `"d"` are absent  

Except:

- `"n"` is still in the *same* position as before (violating constraints)
- The explanation was **post-hoc**—a story invented to justify a guess

The lesson: the LLM is great at **plausible stories and candidate words**, but not at faithfully enforcing symbolic constraints. That must remain in deterministic code.

### Propose, Validate, Retry, Fallback

The retry loop is where the architecture earns its keep:

1. The LLM proposes a guess (possibly illegal)
2. The system:
   - Parses it with `extract_guess()`
   - Checks membership in `candidates`
3. If invalid:
   - Adds a system message explaining what was wrong, including full history
   - Asks the LLM to try again
4. After 5 failed attempts:
   - Falls back to a random valid candidate

This pattern turns the LLM into a **constrained policy proposal engine** instead of an untrusted oracle.

From an **operational AI** perspective, this is the same reliability pattern you see in production agents that call internal APIs or tools: the model proposes a structured action, a deterministic validator checks schema and business constraints, failures trigger targeted retries, and only then does the action actually execute. Wordle is a toy domain, but the architecture—**propose -> validate -> retry -> fallback**—is exactly the kind of pattern you want when you're shipping enterprise agents that must not hallucinate API calls or corrupt downstream systems.

## What Worked Well

- **Clear separation of concerns**:
  - `game_logic.py` is 100% deterministic and unit-tested
  - `wordle_agent.py` orchestrates the agent loop and LLM interaction
- **Belief-state representation**:
  - Maintaining an explicit candidate list makes behavior easy to reason about
- **Duplicate-letter handling**:
  - `excluded_positions` plus bulls/cows maps correctly model tricky Wordle edge cases
- **Propose/verify loop**:
  - Made the system robust to LLM hallucinations and constraint violations
- **Prompt design**:
  - Natural language explanations of feedback (rather than raw arrays) helped the LLM propose better guesses

## What Didn't Work (and What I Learned)

- **Letting the LLM "own" the rules**:
  - Produced confident but invalid moves
  - Explanations were narratives, not proofs
- **Trusting output format too much**:
  - Even with strict instructions, responses varied (prose, code blocks, quasi-JSON)
  - Robust parsing via `extract_guess()` was non-optional
- **Treating all `0`s as absent**:
  - Broke duplicate-letter cases
  - Forced a re-think around excluded positions vs truly absent letters
- **Trying to do everything in one prompt**:
  - Early prompts tried to combine explanation, constraint reasoning, and candidate enumeration
  - Splitting responsibilities (game logic vs policy vs validation) made debugging and extension much easier

The main meta-lesson: **agentic behavior is mostly architecture**—how you represent state, enforce rules, and control where the LLM is allowed to act.

## Tutorial 1: Setting Up the Project Locally

### Prerequisites

- Python 3.7 or higher
- An OpenAI API key
- A word list file (`data/words.txt`) with valid 5-letter words (one per line)
- `git` (optional but recommended)

### Step 1: Clone the Repository

```bash
git clone <your-repository-url>
cd auto-solver-daily-puzzle-1
```

### Step 2: Create and Activate a Virtual Environment

**Windows:**

```bash
python -m venv wordle_env
wordle_env\Scripts\activate
```

**macOS / Linux:**

```bash
python3 -m venv wordle_env
source wordle_env/bin/activate
```

### Step 3: Install Dependencies

```bash
pip install -r requirements.txt
```

This installs:

- `openai` – OpenAI API client
- `python-dotenv` – environment variable management
- `pytest` – testing
- `black`, `flake8` – formatting and linting (dev deps)

### Step 4: Set Up Environment Variables

Create a `.env` file in the project root:

```env
OPENAI_API_KEY=your_openai_api_key_here
DATA_FOLDER=data  # optional, defaults to "data"
```

Get your OpenAI API key from `https://platform.openai.com/api-keys`.

### Step 5: Verify the Word List

Ensure `data/words.txt` exists and contains one 5-letter word per line. If you want to use the canonical Wordle list, you can start from resources like:

- `https://gist.github.com/cfreshman/a03ef2cba789d8cf00c08f767e0fad7b`

## Tutorial 2: Running the Wordle Agent

### Option 1: From a Python Shell

```python
from src.wordle_agent import wordle_agent

wordle_agent()
```

You'll see output similar to:

```text
SOLUTION:  crane
GUESS:  stare
FEEDBACK:  [0, 1, 2, 1, 2]
REMAINING CANDIDATES:  42
[LLM suggests next guess]
...
You've done it!
Your guess crane was the solution after all!
You finished in 4 turns!
```

### Option 2: Via a Simple `main.py` Script

Create a `main.py` in the project root:

```python
from src.wordle_agent import wordle_agent

if __name__ == "__main__":
    wordle_agent()
```

Run:

```bash
python main.py
```

### Running Tests

To validate the deterministic core:

```bash
pytest tests/ -v
```

The tests cover:

- Word list retrieval
- Feedback generation (including duplicates)
- Candidate filtering and belief updates
- Letter categorization
- Random word selection behavior

## Performance & Behavior

### Results

This solver has achieved a **100% success rate** in my tests on the standard Wordle solution set, i.e., it reliably finds the answer within 6 guesses whenever a valid solution exists in the candidate list. That reliability comes less from any particular prompt and more from the way deterministic constraints box in what the LLM is allowed to do.

### Why GPT-4o-mini at temperature 0?

I intentionally chose **GPT-4o-mini** with **temperature = 0** to optimize for **unit economics** and **deterministic behavior** over raw creativity. The model is inexpensive enough to run many games or simulations without worrying about cost, and temperature 0 ensures that, given the same state, the policy behaves predictably. Exactly what you want when you're thinking like a production engineer, not just a tinkerer.

### Visualizing Belief-State Shrinkage

One of the most satisfying parts of running the agent is watching the **belief state** collapse from thousands of possibilities down to a single word. A typical trajectory might look like:

```text
Turn 0  — candidates: 2315   (initial solution list)
Turn 1  — candidates: 150    (after first guess + feedback)
Turn 2  — candidates: 42
Turn 3  — candidates: 8
Turn 4  — candidates: 1      (only the solution remains)
```

That shrinking candidate count is a concrete visualization of the underlying probabilistic pruning the agent is doing each turn.

I didn't benchmark this as a research system, but qualitatively:

- The deterministic core is trivial CPU-wise; it's just list filtering and simple counters
- The main latency comes from LLM calls:
  - 1–5 calls per turn, depending on whether retries are needed
  - 6 turns max -> typically well within interactive latency
- With `temperature=0`, behavior is **consistent**:
  - Given the same solution and initial conditions, the agent tends to follow similar trajectories
  - But because the initial guess and solution are random, each game still feels different

The solver reliably plays "hard mode" Wordle, obeying all constraints—even when the LLM tries to cheat.

## Future Improvements

Some directions I'd like to explore:

- **Support for multi-board variants**: Quordle, Octordle, etc.
- **Statistics and analytics**:
  - Average turns to solve
  - Distribution of win/loss outcomes
  - Comparison of different prompting strategies
- **Model variations**:
  - Swap in different LLMs or policies
  - Compare pure heuristic policies vs LLM-guided policies
- **Web UI**:
  - Simple frontend to visualize the belief state over time
  - Step-through mode showing how candidates narrow each turn
- **Custom word lists & difficulty levels**:
  - Allow users to plug in their own dictionaries
  - Tune for "harder" or "easier" solution sets

## Conclusion

This project wasn't really about "beating Wordle with an LLM." It was about building a small but honest **agent**:

- A deterministic world model (`get_feedback`)
- A belief state (candidate list)
- A policy that proposes actions (the LLM)
- A validation layer that enforces rules before actions take effect

The key lessons:

- **Architecture matters more than any single prompt**: once you have a clean separation between model, state, and policy, agent-like behavior emerges naturally.
- **LLMs are heuristic engines, not rule engines**: they're fantastic at proposing plausible options, but correctness must be enforced outside the model.
- **Propose -> validate -> retry -> fallback** is a robust pattern for building reliable systems on top of probabilistic models.
- Most of the "hard work" in an agent is in the "boring" parts: state representation, edge-case handling, and constraints.

Wordle turned out to be a perfect sandbox for these ideas: small enough to fully understand, but rich enough to expose the difference between generative text and agentic behavior.

## Resources

- **GitHub Repository**: `[https://github.com/RisNag777/auto-solver-daily-puzzle-1](https://github.com/RisNag777/auto-solver-daily-puzzle)`
- **OpenAI API Docs**: `[https://platform.openai.com/docs](https://platform.openai.com/docs)`

*Feel free to fork, modify, and build on this project—especially if you want to explore new agent architectures on top of simple, well-defined games like Wordle.*

