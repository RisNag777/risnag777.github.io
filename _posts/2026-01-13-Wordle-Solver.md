---
layout: post
title: "What Building a Wordle Solver Taught Me About AI Agents"
---

## Introduction

Like most of the human race, towards the end of COVID, I was obsessed with Wordle. Over time my own streaks faded, but the game didn't really leave my life. My wife still plays Wordle religiously every morning, and our daily ritual is to solve it together in bed before we kickstart our day. On the days we're completely stumped, my phone quietly becomes our unofficial "six extra lives."

Those moments made me wonder what it would actually mean for a machine to play Wordle well. Not just memorizing answers, but behaving like a real **agent**: maintaining beliefs about possible solutions, updating those beliefs with evidence, and choosing the next action strategically. So I decided to build a Wordle-solving agent! Not by hard-coding heuristics, but by combining deterministic game logic with an LLM that proposes guesses under strict constraints.

This post walks through that journey: the core idea, the system architecture, the agent loop, what worked, what didn't, and how you can run it yourself.

## The Core Idea

At its heart, Wordle isn't just a word game—it's a **belief-state search problem**.

A good Wordle solver isn't "making guesses" so much as **updating its beliefs** about the hidden word. Each guess and its feedback carve away impossible words until only the true solution remains.

Reframed in those terms:

| Concept        | Wordle meaning                     |
| ------------- | ---------------------------------- |
| Hidden state  | The true solution word             |
| Belief state  | All words consistent with feedback |
| Action        | Choosing the next guess            |
| Observation   | Feedback (0,1,2 per letter)        |

The loop becomes:

1. Start with a list of all possible solutions (e.g., the 2,315 canonical Wordle answers)
2. Pick an initial 5-letter guess
3. Receive feedback:
   - `2` (Green / Bull): correct letter, correct position  
   - `1` (Yellow / Cow): correct letter, wrong position  
   - `0` (Gray / Absent): letter not in the word
4. Trim the solution space to only words consistent with all feedback so far
5. Choose the next guess from that filtered belief state
6. Repeat until the solution is found (≤ 6 guesses)

Instead of treating Wordle as a language modeling task ("predict the next good word"), the system treats it as **sequential inference**: maintain a belief over possible worlds, observe feedback, update, and act.

## System Architecture

The architecture cleanly separates **deterministic game logic** from the **stochastic LLM policy**. The code is split into two main modules:

- `game_logic.py`: pure, testable, deterministic Wordle mechanics
- `wordle_agent.py`: the agent loop plus interaction with OpenAI's GPT-4o-mini

Here's the high-level architecture:

```svg
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

  <!-- (diagram content omitted here for brevity – keep your full SVG when publishing) -->
</svg>
```

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

Takes the current guess and feedback and converts them into a human-readable explanation for the LLM, e.g.:

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

```svg
<svg xmlns="http://www.w3.org/2000/svg" width="900" height="1200" viewBox="-100 0 1000 1000">
  <!-- (use your second SVG diagram from the original post here – Wordle Agent Loop) -->
</svg>
```

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
   - Update a `history` dictionary mapping guess → feedback

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

This loop embodies a classic pattern: **propose → validate → retry → fallback**.

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

I didn't benchmark this as a research system, but qualitatively:

- The deterministic core is trivial CPU-wise; it's just list filtering and simple counters
- The main latency comes from LLM calls:
  - 1–5 calls per turn, depending on whether retries are needed
  - 6 turns max → typically well within interactive latency
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
- **Propose → validate → retry → fallback** is a robust pattern for building reliable systems on top of probabilistic models.
- Most of the "hard work" in an agent is in the "boring" parts: state representation, edge-case handling, and constraints.

Wordle turned out to be a perfect sandbox for these ideas: small enough to fully understand, but rich enough to expose the difference between generative text and agentic behavior.

## Resources

- **GitHub Repository**: `https://github.com/RisNag777/auto-solver-daily-puzzle-1`
- **OpenAI API Docs**: `https://platform.openai.com/docs`

*Feel free to fork, modify, and build on this project—especially if you want to explore new agent architectures on top of simple, well-defined games like Wordle.*
