---
layout: post
title: "Building a Narrative Playlist Generator: From Concept to Reality"
---

![Narrative Playlist Agent](https://github.com/user-attachments/assets/f971de46-8745-4026-839e-523c28f887b7)

## Introduction

Music playlists have evolved from simple collections of songs to curated experiences that tell stories. This **Narrative Playlist Agent** is an AI-powered system that transforms a user's emotional prompt into a carefully sequenced playlist with a coherent emotional arc, just like a movie soundtrack for your life.

This blog post chronicles the journey of building this system: the architecture, the challenges, what worked, what didn't, and how to use it yourself.

## The Core Idea

Traditional playlist generators focus on similarity, i.e., songs that sound alike or share metadata. But what if we could create playlists that follow an emotional narrative? Like a story with a beginning, middle, and end, where each song serves a purpose in the emotional journey.

The Narrative Playlist Agent does exactly that:
1. Takes a user's emotional prompt (e.g., "Late-night reflective indie energy, but hopeful by the end")
2. Breaks it into narrative phases (like story acts)
3. Finds songs that match each phase
4. Orders them to create smooth emotional transitions
5. Generates a story explaining the playlist's arc
6. Visualizes the energy progression

## System Architecture

The system follows a multi-agent architecture where specialized AI agents handle different aspects of playlist generation. Here's the high-level architecture:

<svg viewBox="0 0 1200 1000" xmlns="http://www.w3.org/2000/svg">
  <!-- Background -->
  <rect width="1200" height="1000" fill="#f8f9fa"/>
  
  <!-- Title -->
  <text x="600" y="40" font-family="Arial, sans-serif" font-size="24" font-weight="bold" text-anchor="middle" fill="#2c3e50">Narrative Playlist Agent - System Architecture</text>
  
  <!-- User Layer -->
  <rect x="50" y="60" width="1100" height="100" rx="10" fill="#e3f2fd" stroke="#1976d2" stroke-width="2"/>
  <text x="600" y="100" font-family="Arial, sans-serif" font-size="18" font-weight="bold" text-anchor="middle" fill="#1976d2">User Interface (Streamlit)</text>
  <text x="600" y="130" font-family="Arial, sans-serif" font-size="14" text-anchor="middle" fill="#424242">Prompt Input | Progress Bar | Results Display | Energy Arc Visualization</text>
  
  <!-- Pipeline Layer -->
  <rect x="50" y="200" width="1100" height="80" rx="10" fill="#fff3e0" stroke="#f57c00" stroke-width="2"/>
  <text x="600" y="235" font-family="Arial, sans-serif" font-size="18" font-weight="bold" text-anchor="middle" fill="#f57c00">Narrative Playlist Pipeline</text>
  <text x="600" y="260" font-family="Arial, sans-serif" font-size="14" text-anchor="middle" fill="#424242">Orchestrates all agents | Manages progress | Handles cancellation</text>
  
  <!-- Agent Layer -->
  <rect x="50" y="320" width="1100" height="400" rx="10" fill="#f1f8e9" stroke="#558b2f" stroke-width="2"/>
  <text x="600" y="355" font-family="Arial, sans-serif" font-size="18" font-weight="bold" text-anchor="middle" fill="#558b2f">AI Agents</text>
  
  <!-- Row 1: Narrative Agents -->
  <rect x="80" y="380" width="200" height="120" rx="5" fill="#ffffff" stroke="#4caf50" stroke-width="2"/>
  <text x="180" y="405" font-family="Arial, sans-serif" font-size="14" font-weight="bold" text-anchor="middle" fill="#2e7d32">Narrative Agent</text>
  <text x="180" y="430" font-family="Arial, sans-serif" font-size="11" text-anchor="middle" fill="#424242">Generates story</text>
  <text x="180" y="450" font-family="Arial, sans-serif" font-size="11" text-anchor="middle" fill="#424242">phases from</text>
  <text x="180" y="470" font-family="Arial, sans-serif" font-size="11" text-anchor="middle" fill="#424242">user prompt</text>
  <text x="180" y="490" font-family="Arial, sans-serif" font-size="10" text-anchor="middle" fill="#757575">GPT-4o-mini</text>
  
  <rect x="300" y="380" width="200" height="120" rx="5" fill="#ffffff" stroke="#4caf50" stroke-width="2"/>
  <text x="400" y="405" font-family="Arial, sans-serif" font-size="14" font-weight="bold" text-anchor="middle" fill="#2e7d32">Color Agent</text>
  <text x="400" y="430" font-family="Arial, sans-serif" font-size="11" text-anchor="middle" fill="#424242">Assigns colors</text>
  <text x="400" y="450" font-family="Arial, sans-serif" font-size="11" text-anchor="middle" fill="#424242">to phases based</text>
  <text x="400" y="470" font-family="Arial, sans-serif" font-size="11" text-anchor="middle" fill="#424242">on emotion</text>
  <text x="400" y="490" font-family="Arial, sans-serif" font-size="10" text-anchor="middle" fill="#757575">GPT-4o-mini</text>
  
  <rect x="520" y="380" width="200" height="120" rx="5" fill="#ffffff" stroke="#4caf50" stroke-width="2"/>
  <text x="620" y="405" font-family="Arial, sans-serif" font-size="14" font-weight="bold" text-anchor="middle" fill="#2e7d32">Query Agent</text>
  <text x="620" y="430" font-family="Arial, sans-serif" font-size="11" text-anchor="middle" fill="#424242">Generates diverse</text>
  <text x="620" y="450" font-family="Arial, sans-serif" font-size="11" text-anchor="middle" fill="#424242">YouTube search</text>
  <text x="620" y="470" font-family="Arial, sans-serif" font-size="11" text-anchor="middle" fill="#424242">queries</text>
  <text x="620" y="490" font-family="Arial, sans-serif" font-size="10" text-anchor="middle" fill="#757575">GPT-4o-mini</text>
  
  <rect x="740" y="380" width="200" height="120" rx="5" fill="#ffffff" stroke="#4caf50" stroke-width="2"/>
  <text x="840" y="405" font-family="Arial, sans-serif" font-size="14" font-weight="bold" text-anchor="middle" fill="#2e7d32">Narrator Agent</text>
  <text x="840" y="430" font-family="Arial, sans-serif" font-size="11" text-anchor="middle" fill="#424242">Generates story</text>
  <text x="840" y="450" font-family="Arial, sans-serif" font-size="11" text-anchor="middle" fill="#424242">text explaining</text>
  <text x="840" y="470" font-family="Arial, sans-serif" font-size="11" text-anchor="middle" fill="#424242">playlist arc</text>
  <text x="840" y="490" font-family="Arial, sans-serif" font-size="10" text-anchor="middle" fill="#757575">GPT-4o-mini</text>
  
  <!-- Row 2: YouTube & Filtering Agents -->
  <rect x="80" y="520" width="200" height="120" rx="5" fill="#ffffff" stroke="#2196f3" stroke-width="2"/>
  <text x="180" y="545" font-family="Arial, sans-serif" font-size="14" font-weight="bold" text-anchor="middle" fill="#1565c0">YouTube Agent</text>
  <text x="180" y="570" font-family="Arial, sans-serif" font-size="11" text-anchor="middle" fill="#424242">Searches YouTube</text>
  <text x="180" y="590" font-family="Arial, sans-serif" font-size="11" text-anchor="middle" fill="#424242">Filters by duration,</text>
  <text x="180" y="610" font-family="Arial, sans-serif" font-size="11" text-anchor="middle" fill="#424242">view count</text>
  <text x="180" y="630" font-family="Arial, sans-serif" font-size="10" text-anchor="middle" fill="#757575">yt-dlp</text>
  
  <rect x="300" y="520" width="200" height="120" rx="5" fill="#ffffff" stroke="#2196f3" stroke-width="2"/>
  <text x="400" y="545" font-family="Arial, sans-serif" font-size="14" font-weight="bold" text-anchor="middle" fill="#1565c0">Semantic Filter</text>
  <text x="400" y="570" font-family="Arial, sans-serif" font-size="11" text-anchor="middle" fill="#424242">Filters songs by</text>
  <text x="400" y="590" font-family="Arial, sans-serif" font-size="11" text-anchor="middle" fill="#424242">emotional fit</text>
  <text x="400" y="610" font-family="Arial, sans-serif" font-size="11" text-anchor="middle" fill="#424242">(batch processing)</text>
  <text x="400" y="630" font-family="Arial, sans-serif" font-size="10" text-anchor="middle" fill="#757575">GPT-4o-mini</text>
  
  <rect x="520" y="520" width="200" height="120" rx="5" fill="#ffffff" stroke="#2196f3" stroke-width="2"/>
  <text x="620" y="545" font-family="Arial, sans-serif" font-size="14" font-weight="bold" text-anchor="middle" fill="#1565c0">Sequence Agent</text>
  <text x="620" y="570" font-family="Arial, sans-serif" font-size="11" text-anchor="middle" fill="#424242">Assigns songs</text>
  <text x="620" y="590" font-family="Arial, sans-serif" font-size="11" text-anchor="middle" fill="#424242">to phases</text>
  <text x="620" y="610" font-family="Arial, sans-serif" font-size="11" text-anchor="middle" fill="#424242">(batch processing)</text>
  <text x="620" y="630" font-family="Arial, sans-serif" font-size="10" text-anchor="middle" fill="#757575">GPT-4o-mini</text>
  
  <rect x="740" y="520" width="200" height="120" rx="5" fill="#ffffff" stroke="#2196f3" stroke-width="2"/>
  <text x="840" y="545" font-family="Arial, sans-serif" font-size="14" font-weight="bold" text-anchor="middle" fill="#1565c0">Ordering Agent</text>
  <text x="840" y="570" font-family="Arial, sans-serif" font-size="11" text-anchor="middle" fill="#424242">Scores & orders</text>
  <text x="840" y="590" font-family="Arial, sans-serif" font-size="11" text-anchor="middle" fill="#424242">songs per phase</text>
  <text x="840" y="610" font-family="Arial, sans-serif" font-size="11" text-anchor="middle" fill="#424242">(batch processing)</text>
  <text x="840" y="630" font-family="Arial, sans-serif" font-size="10" text-anchor="middle" fill="#757575">GPT-4o-mini</text>
  
  <!-- Smoothing Agent -->
  <rect x="960" y="460" width="180" height="120" rx="5" fill="#ffffff" stroke="#9c27b0" stroke-width="2"/>
  <text x="1050" y="485" font-family="Arial, sans-serif" font-size="14" font-weight="bold" text-anchor="middle" fill="#6a1b9a">Smoothing Agent</text>
  <text x="1050" y="510" font-family="Arial, sans-serif" font-size="11" text-anchor="middle" fill="#424242">Smooths energy</text>
  <text x="1050" y="530" font-family="Arial, sans-serif" font-size="11" text-anchor="middle" fill="#424242">transitions</text>
  <text x="1050" y="550" font-family="Arial, sans-serif" font-size="11" text-anchor="middle" fill="#424242">between songs</text>
  <text x="1050" y="570" font-family="Arial, sans-serif" font-size="10" text-anchor="middle" fill="#757575">Algorithm</text>
  
  <!-- External Services -->
  <rect x="50" y="770" width="1100" height="110" rx="10" fill="#fce4ec" stroke="#c2185b" stroke-width="2"/>
  <text x="600" y="800" font-family="Arial, sans-serif" font-size="18" font-weight="bold" text-anchor="middle" fill="#c2185b">External Services</text>
  
  <rect x="150" y="820" width="150" height="40" rx="5" fill="#ffffff" stroke="#e91e63" stroke-width="1"/>
  <text x="225" y="845" font-family="Arial, sans-serif" font-size="12" text-anchor="middle" fill="#424242">OpenAI API</text>
  
  <rect x="400" y="820" width="150" height="40" rx="5" fill="#ffffff" stroke="#e91e63" stroke-width="1"/>
  <text x="475" y="845" font-family="Arial, sans-serif" font-size="12" text-anchor="middle" fill="#424242">YouTube (yt-dlp)</text>
  
  <rect x="650" y="820" width="150" height="40" rx="5" fill="#ffffff" stroke="#e91e63" stroke-width="1"/>
  <text x="725" y="845" font-family="Arial, sans-serif" font-size="12" text-anchor="middle" fill="#424242">YouTube Data API</text>
  
  <rect x="900" y="820" width="150" height="40" rx="5" fill="#ffffff" stroke="#e91e63" stroke-width="1"/>
  <text x="975" y="845" font-family="Arial, sans-serif" font-size="12" text-anchor="middle" fill="#424242">Matplotlib</text>
  
  <!-- Arrows -->
  <path d="M 600 160 L 600 200" stroke="#424242" stroke-width="2" fill="none" marker-end="url(#arrowhead)"/>
  <path d="M 600 280 L 600 320" stroke="#424242" stroke-width="2" fill="none" marker-end="url(#arrowhead)"/>
  <path d="M 600 720 L 600 770" stroke="#424242" stroke-width="2" fill="none" marker-end="url(#arrowhead)"/>
  
  <!-- Arrow marker definition -->
  <defs>
    <marker id="arrowhead" markerWidth="10" markerHeight="10" refX="9" refY="3" orient="auto">
      <polygon points="0 0, 10 3, 0 6" fill="#424242"/>
    </marker>
  </defs>
</svg>

### Key Components

#### 1. **Narrative Agent** (`narrative_agent.py`)
Converts a user's emotional prompt into structured story phases. Uses GPT-4o-mini to generate 4-6 phases, each with:
- Phase name (e.g., "Opening", "Rising Action", "Climax")
- Description
- Emotion tags (3-5 keywords)

**Example Output:**
```json
[
  {
    "phase": "Opening",
    "description": "Quiet introspection begins",
    "emotion_tags": ["melancholic", "reflective", "introspective"]
  },
  {
    "phase": "Rising Action",
    "description": "Energy gradually builds",
    "emotion_tags": ["building", "hopeful", "growing"]
  }
]
```

#### 2. **Color Assignment Agent** (`color_agent.py`)
Assigns colors to phases based on their emotional content using color psychology. This ensures visual consistency across the UI and energy arc visualization.

#### 3. **YouTube Query Agent** (`youtube_query_agent.py`)
Generates search queries from the user prompt. Instead of searching for songs directly, it creates multiple diverse queries to cast a wider net.

#### 4. **YouTube Music Agent** (`youtube_agent.py`)
Searches YouTube using `yt-dlp` and filters results:
- Excludes playlists, mixes, compilations
- Filters by duration (max 10 minutes)
- Filters by view count (optional)
- Removes duplicates

#### 5. **YouTube Semantic Filter** (`youtube_filter_agent.py`)
Uses GPT to determine if songs emotionally match the theme. Processes all songs in a single batch API call for efficiency.

#### 6. **Sequence Director Agent** (`sequence_agent.py`)
Assigns songs to narrative phases. Uses batch processing to assign all songs at once, then delegates ordering to the Phase Ordering Agent.

#### 7. **Phase Ordering Agent** (`phase_ordering_agent.py`)
Scores and orders songs within each phase based on:
- Energy level (0-1)
- Emotional weight (0-1)

Uses batch scoring to process all songs in a phase simultaneously.

#### 8. **Smoothing Agent** (`smoothing_agent.py`)
Smooths transitions between songs by minimizing energy jumps. Uses a sliding window algorithm to swap songs when adjacent energy differences exceed a threshold.

#### 9. **Playlist Narrator Agent** (`playlist_narrator_agent.py`)
Generates a narrative story explaining the playlist's emotional arc, segmented by phase.

## Pipeline Flow

The main pipeline orchestrates all agents in a specific sequence:

<svg viewBox="0 0 1400 1000" xmlns="http://www.w3.org/2000/svg">
  <!-- Background -->
  <rect width="1400" height="1000" fill="#f8f9fa"/>
  
  <!-- Title -->
  <text x="700" y="40" font-family="Arial, sans-serif" font-size="24" font-weight="bold" text-anchor="middle" fill="#2c3e50">Pipeline Flow Diagram</text>
  
  <!-- Start -->
  <circle cx="100" cy="100" r="30" fill="#4caf50" stroke="#2e7d32" stroke-width="2"/>
  <text x="100" y="107" font-family="Arial, sans-serif" font-size="12" font-weight="bold" text-anchor="middle" fill="white">START</text>
  
  <!-- Step 1: Generate Phases -->
  <rect x="200" y="70" width="180" height="70" rx="10" fill="#e3f2fd" stroke="#1976d2" stroke-width="2"/>
  <text x="290" y="95" font-family="Arial, sans-serif" font-size="14" font-weight="bold" text-anchor="middle" fill="#1976d2">1. Generate Phases</text>
  <text x="290" y="115" font-family="Arial, sans-serif" font-size="11" text-anchor="middle" fill="#424242">Narrative Agent</text>
  <text x="290" y="130" font-family="Arial, sans-serif" font-size="10" text-anchor="middle" fill="#757575">0-5%</text>
  
  <!-- Step 2: Assign Colors -->
  <rect x="420" y="70" width="180" height="70" rx="10" fill="#fff3e0" stroke="#f57c00" stroke-width="2"/>
  <text x="510" y="95" font-family="Arial, sans-serif" font-size="14" font-weight="bold" text-anchor="middle" fill="#f57c00">2. Assign Colors</text>
  <text x="510" y="115" font-family="Arial, sans-serif" font-size="11" text-anchor="middle" fill="#424242">Color Agent</text>
  <text x="510" y="130" font-family="Arial, sans-serif" font-size="10" text-anchor="middle" fill="#757575">5%</text>
  
  <!-- Step 3: Generate Queries -->
  <rect x="640" y="70" width="180" height="70" rx="10" fill="#f1f8e9" stroke="#558b2f" stroke-width="2"/>
  <text x="730" y="95" font-family="Arial, sans-serif" font-size="14" font-weight="bold" text-anchor="middle" fill="#558b2f">3. Generate Queries</text>
  <text x="730" y="115" font-family="Arial, sans-serif" font-size="11" text-anchor="middle" fill="#424242">Query Agent</text>
  <text x="730" y="130" font-family="Arial, sans-serif" font-size="10" text-anchor="middle" fill="#757575">5-12.5%</text>
  
  <!-- Step 4: Search YouTube -->
  <rect x="860" y="70" width="180" height="70" rx="10" fill="#fce4ec" stroke="#c2185b" stroke-width="2"/>
  <text x="950" y="95" font-family="Arial, sans-serif" font-size="14" font-weight="bold" text-anchor="middle" fill="#c2185b">4. Search YouTube</text>
  <text x="950" y="115" font-family="Arial, sans-serif" font-size="11" text-anchor="middle" fill="#424242">YouTube Agent</text>
  <text x="950" y="130" font-family="Arial, sans-serif" font-size="10" text-anchor="middle" fill="#757575">12.5-37.5%</text>
  
  <!-- Step 5: Filter -->
  <rect x="1080" y="70" width="180" height="70" rx="10" fill="#e8eaf6" stroke="#3f51b5" stroke-width="2"/>
  <text x="1170" y="95" font-family="Arial, sans-serif" font-size="14" font-weight="bold" text-anchor="middle" fill="#3f51b5">5. Filter</text>
  <text x="1170" y="115" font-family="Arial, sans-serif" font-size="11" text-anchor="middle" fill="#424242">Semantic Filter</text>
  <text x="1170" y="130" font-family="Arial, sans-serif" font-size="10" text-anchor="middle" fill="#757575">37.5-50%</text>
  
  <!-- Step 6: Assign to Phases -->
  <rect x="1080" y="200" width="180" height="70" rx="10" fill="#fff9c4" stroke="#f9a825" stroke-width="2"/>
  <text x="1170" y="225" font-family="Arial, sans-serif" font-size="14" font-weight="bold" text-anchor="middle" fill="#f9a825">6. Assign to Phases</text>
  <text x="1170" y="245" font-family="Arial, sans-serif" font-size="11" text-anchor="middle" fill="#424242">Sequence Agent</text>
  <text x="1170" y="260" font-family="Arial, sans-serif" font-size="10" text-anchor="middle" fill="#757575">50-62.5%</text>
  
  <!-- Step 7: Order Within Phases -->
  <rect x="640" y="200" width="180" height="70" rx="10" fill="#e0f2f1" stroke="#00695c" stroke-width="2"/>
  <text x="730" y="225" font-family="Arial, sans-serif" font-size="14" font-weight="bold" text-anchor="middle" fill="#00695c">7. Order Phases</text>
  <text x="730" y="245" font-family="Arial, sans-serif" font-size="11" text-anchor="middle" fill="#424242">Ordering Agent</text>
  <text x="730" y="260" font-family="Arial, sans-serif" font-size="10" text-anchor="middle" fill="#757575">62.5-75%</text>
  
  <!-- Step 8: Smooth -->
  <rect x="200" y="200" width="180" height="70" rx="10" fill="#f3e5f5" stroke="#7b1fa2" stroke-width="2"/>
  <text x="290" y="225" font-family="Arial, sans-serif" font-size="14" font-weight="bold" text-anchor="middle" fill="#7b1fa2">8. Smooth</text>
  <text x="290" y="245" font-family="Arial, sans-serif" font-size="11" text-anchor="middle" fill="#424242">Smoothing Agent</text>
  <text x="290" y="260" font-family="Arial, sans-serif" font-size="10" text-anchor="middle" fill="#757575">75-87.5%</text>
  
  <!-- Step 9: Narrate -->
  <rect x="200" y="330" width="180" height="70" rx="10" fill="#e1bee7" stroke="#8e24aa" stroke-width="2"/>
  <text x="290" y="355" font-family="Arial, sans-serif" font-size="14" font-weight="bold" text-anchor="middle" fill="#8e24aa">9. Narrate</text>
  <text x="290" y="375" font-family="Arial, sans-serif" font-size="11" text-anchor="middle" fill="#424242">Narrator Agent</text>
  <text x="290" y="390" font-family="Arial, sans-serif" font-size="10" text-anchor="middle" fill="#757575">87.5-100%</text>
  
  <!-- Output -->
  <rect x="1000" y="330" width="180" height="70" rx="10" fill="#c8e6c9" stroke="#2e7d32" stroke-width="2"/>
  <text x="1090" y="355" font-family="Arial, sans-serif" font-size="14" font-weight="bold" text-anchor="middle" fill="#2e7d32">Output</text>
  <text x="1090" y="375" font-family="Arial, sans-serif" font-size="11" text-anchor="middle" fill="#424242">Playlist + Story</text>
  <text x="1090" y="390" font-family="Arial, sans-serif" font-size="10" text-anchor="middle" fill="#757575">100%</text>
  
  <!-- End -->
  <circle cx="1300" cy="360" r="30" fill="#f44336" stroke="#c62828" stroke-width="2"/>
  <text x="1300" y="367" font-family="Arial, sans-serif" font-size="12" font-weight="bold" text-anchor="middle" fill="white">END</text>
  
  <!-- Data Flow Annotations -->
  <rect x="50" y="450" width="1300" height="500" rx="10" fill="#ffffff" stroke="#bdbdbd" stroke-width="1" stroke-dasharray="5,5"/>
  <text x="700" y="480" font-family="Arial, sans-serif" font-size="18" font-weight="bold" text-anchor="middle" fill="#424242">Data Flow</text>
  
  <!-- User Prompt -->
  <rect x="100" y="500" width="200" height="80" rx="5" fill="#e3f2fd" stroke="#1976d2" stroke-width="2"/>
  <text x="200" y="525" font-family="Arial, sans-serif" font-size="14" font-weight="bold" text-anchor="middle" fill="#1976d2">User Prompt</text>
  <text x="200" y="550" font-family="Arial, sans-serif" font-size="11" text-anchor="middle" fill="#424242">"Late-night</text>
  <text x="200" y="570" font-family="Arial, sans-serif" font-size="11" text-anchor="middle" fill="#424242">reflective indie</text>
  
  <!-- Phases -->
  <rect x="350" y="500" width="200" height="80" rx="5" fill="#fff3e0" stroke="#f57c00" stroke-width="2"/>
  <text x="450" y="525" font-family="Arial, sans-serif" font-size="14" font-weight="bold" text-anchor="middle" fill="#f57c00">Story Phases</text>
  <text x="450" y="550" font-family="Arial, sans-serif" font-size="11" text-anchor="middle" fill="#424242">Opening, Rising,</text>
  <text x="450" y="570" font-family="Arial, sans-serif" font-size="11" text-anchor="middle" fill="#424242">Climax, Resolution</text>
  
  <!-- Search Queries -->
  <rect x="600" y="500" width="200" height="80" rx="5" fill="#f1f8e9" stroke="#558b2f" stroke-width="2"/>
  <text x="700" y="525" font-family="Arial, sans-serif" font-size="14" font-weight="bold" text-anchor="middle" fill="#558b2f">Search Queries</text>
  <text x="700" y="550" font-family="Arial, sans-serif" font-size="11" text-anchor="middle" fill="#424242">Multiple diverse</text>
  <text x="700" y="570" font-family="Arial, sans-serif" font-size="11" text-anchor="middle" fill="#424242">queries</text>
  
  <!-- Raw Tracks -->
  <rect x="850" y="500" width="200" height="80" rx="5" fill="#fce4ec" stroke="#c2185b" stroke-width="2"/>
  <text x="950" y="525" font-family="Arial, sans-serif" font-size="14" font-weight="bold" text-anchor="middle" fill="#c2185b">Raw Tracks</text>
  <text x="950" y="550" font-family="Arial, sans-serif" font-size="11" text-anchor="middle" fill="#424242">YouTube search</text>
  <text x="950" y="570" font-family="Arial, sans-serif" font-size="11" text-anchor="middle" fill="#424242">results</text>
  
  <!-- Filtered Tracks -->
  <rect x="1100" y="500" width="200" height="80" rx="5" fill="#e8eaf6" stroke="#3f51b5" stroke-width="2"/>
  <text x="1200" y="525" font-family="Arial, sans-serif" font-size="14" font-weight="bold" text-anchor="middle" fill="#3f51b5">Filtered Tracks</text>
  <text x="1200" y="550" font-family="Arial, sans-serif" font-size="11" text-anchor="middle" fill="#424242">Emotionally</text>
  <text x="1200" y="570" font-family="Arial, sans-serif" font-size="11" text-anchor="middle" fill="#424242">matched</text>
  
  <!-- Phase Buckets -->
  <rect x="100" y="660" width="200" height="80" rx="5" fill="#fff9c4" stroke="#f9a825" stroke-width="2"/>
  <text x="200" y="685" font-family="Arial, sans-serif" font-size="14" font-weight="bold" text-anchor="middle" fill="#f9a825">Phase Buckets</text>
  <text x="200" y="710" font-family="Arial, sans-serif" font-size="11" text-anchor="middle" fill="#424242">Songs grouped</text>
  <text x="200" y="730" font-family="Arial, sans-serif" font-size="11" text-anchor="middle" fill="#424242">by phase</text>
  
  <!-- Scored Playlist -->
  <rect x="350" y="660" width="200" height="80" rx="5" fill="#e0f2f1" stroke="#00695c" stroke-width="2"/>
  <text x="450" y="685" font-family="Arial, sans-serif" font-size="14" font-weight="bold" text-anchor="middle" fill="#00695c">Scored Playlist</text>
  <text x="450" y="710" font-family="Arial, sans-serif" font-size="11" text-anchor="middle" fill="#424242">Songs with energy</text>
  <text x="450" y="730" font-family="Arial, sans-serif" font-size="11" text-anchor="middle" fill="#424242">scores</text>
  
  <!-- Smoothed Playlist -->
  <rect x="600" y="660" width="200" height="80" rx="5" fill="#f3e5f5" stroke="#7b1fa2" stroke-width="2"/>
  <text x="700" y="685" font-family="Arial, sans-serif" font-size="14" font-weight="bold" text-anchor="middle" fill="#7b1fa2">Smoothed Playlist</text>
  <text x="700" y="710" font-family="Arial, sans-serif" font-size="11" text-anchor="middle" fill="#424242">Transitions</text>
  <text x="700" y="730" font-family="Arial, sans-serif" font-size="11" text-anchor="middle" fill="#424242">optimized</text>
  
  <!-- Final Output -->
  <rect x="850" y="660" width="200" height="80" rx="5" fill="#c8e6c9" stroke="#2e7d32" stroke-width="2"/>
  <text x="950" y="685" font-family="Arial, sans-serif" font-size="14" font-weight="bold" text-anchor="middle" fill="#2e7d32">Final Playlist</text>
  <text x="950" y="710" font-family="Arial, sans-serif" font-size="11" text-anchor="middle" fill="#424242">Ordered songs +</text>
  <text x="950" y="730" font-family="Arial, sans-serif" font-size="11" text-anchor="middle" fill="#424242">narrative story</text>
  
  <!-- Story Segments -->
  <rect x="1100" y="660" width="200" height="80" rx="5" fill="#e1bee7" stroke="#8e24aa" stroke-width="2"/>
  <text x="1200" y="685" font-family="Arial, sans-serif" font-size="14" font-weight="bold" text-anchor="middle" fill="#8e24aa">Story Segments</text>
  <text x="1200" y="710" font-family="Arial, sans-serif" font-size="11" text-anchor="middle" fill="#424242">Narrative text</text>
  <text x="1200" y="730" font-family="Arial, sans-serif" font-size="11" text-anchor="middle" fill="#424242">per phase</text>
  
  <!-- Arrows for data flow -->
  <path d="M 380 100 L 420 100" stroke="#424242" stroke-width="2" fill="none" marker-end="url(#arrowhead)"/>
  <path d="M 600 100 L 640 100" stroke="#424242" stroke-width="2" fill="none" marker-end="url(#arrowhead)"/>
  <path d="M 820 100 L 860 100" stroke="#424242" stroke-width="2" fill="none" marker-end="url(#arrowhead)"/>
  <path d="M 1040 100 L 1080 100" stroke="#424242" stroke-width="2" fill="none" marker-end="url(#arrowhead)"/>
  <path d="M 1170 140 L 1170 200" stroke="#424242" stroke-width="2" fill="none" marker-end="url(#arrowhead)"/>
  <path d="M 1080 230 L 820 230" stroke="#424242" stroke-width="2" fill="none" marker-end="url(#arrowhead)"/>
  <path d="M 640 230 L 380 230" stroke="#424242" stroke-width="2" fill="none" marker-end="url(#arrowhead)"/>
  <path d="M 290 270 L 290 330" stroke="#424242" stroke-width="2" fill="none" marker-end="url(#arrowhead)"/>
  <path d="M 380 360 L 1000 360" stroke="#424242" stroke-width="2" fill="none" marker-end="url(#arrowhead)"/>
  <path d="M 300 540 L 350 540" stroke="#424242" stroke-width="2" fill="none" marker-end="url(#arrowhead)"/>
  <path d="M 550 540 L 600 540" stroke="#424242" stroke-width="2" fill="none" marker-end="url(#arrowhead)"/>
  <path d="M 800 540 L 850 540" stroke="#424242" stroke-width="2" fill="none" marker-end="url(#arrowhead)"/>
  <path d="M 1050 540 L 1100 540" stroke="#424242" stroke-width="2" fill="none" marker-end="url(#arrowhead)"/>
  <path d="M 1200 580 L 1200 620 L 200 620 L 200 660" stroke="#424242" stroke-width="2" fill="none" marker-end="url(#arrowhead)"/>
  <path d="M 300 700 L 350 700" stroke="#424242" stroke-width="2" fill="none" marker-end="url(#arrowhead)"/>
  <path d="M 550 700 L 600 700" stroke="#424242" stroke-width="2" fill="none" marker-end="url(#arrowhead)"/>
  <path d="M 800 700 L 850 700" stroke="#424242" stroke-width="2" fill="none" marker-end="url(#arrowhead)"/>
  <path d="M 1050 700 L 1100 700" stroke="#424242" stroke-width="2" fill="none" marker-end="url(#arrowhead)"/>
  
  <!-- Arrow marker definition -->
  <defs>
    <marker id="arrowhead" markerWidth="10" markerHeight="10" refX="9" refY="3" orient="auto">
      <polygon points="0 0, 10 3, 0 6" fill="#424242"/>
    </marker>
  </defs>
</svg>

### Detailed Pipeline Steps

1. **Generate Phases** (0-5%)
   - Narrative Agent creates story phases from user prompt
   - Color Agent assigns colors to phases

2. **Generate Search Queries** (5-12.5%)
   - YouTube Query Agent creates diverse search queries

3. **Search YouTube** (12.5-37.5%)
   - YouTube Music Agent searches for each query
   - Deduplicates across queries
   - Applies filters (duration, view count, banned words)

4. **Filter by Emotional Fit** (37.5-50%)
   - YouTube Semantic Filter evaluates all songs in one batch
   - Keeps only emotionally relevant songs

5. **Assign to Phases** (50-62.5%)
   - Sequence Director Agent assigns songs to phases
   - Uses batch processing for efficiency

6. **Order Within Phases** (62.5-75%)
   - Phase Ordering Agent scores and orders songs per phase
   - Batch scores all songs in each phase

7. **Smooth Transitions** (75-87.5%)
   - Smoothing Agent minimizes energy jumps between songs

8. **Generate Story** (87.5-100%)
   - Playlist Narrator Agent creates narrative text

## Technical Deep Dive

### Batch Processing Optimization

One of the key optimizations was implementing batch processing for API calls. Initially, the system made individual API calls for each song, which was slow and expensive.

**Before:**
```python
# Individual calls - slow!
for song in songs:
    result = filter_agent.filter_song(song, emotion_tags)
```

**After:**
```python
# Single batch call - fast!
results = filter_agent.filter_matches(all_songs, emotion_tags)
```

This reduced API calls from O(n) to O(1) for filtering and phase assignment, dramatically improving performance.

### Cooperative Cancellation

Since Streamlit runs synchronously, we implemented cooperative cancellation using callbacks:

```python
def run(self, user_prompt: str, cancel_cb: Optional[Callable[[], bool]] = None):
    # Check cancellation at key points
    self._check_cancel(cancel_cb)
    # ... do work ...
```

The pipeline checks the cancellation flag at strategic points, allowing users to stop generation without blocking the UI.

### Error Handling and Robustness

The system includes multiple layers of error handling:

1. **Early Exit on Empty Results**: If no songs are found at any stage, the pipeline stops immediately to avoid wasting API calls.

2. **Graceful Degradation**: If color assignment fails, the system falls back to a hardcoded color palette.

3. **JSON Parsing Resilience**: All agents use robust JSON extraction that handles markdown code blocks, verbose responses, and malformed JSON.

### YouTube Search Optimization

YouTube search was initially slow because we extracted full metadata for every candidate. We optimized it with a two-phase approach:

1. **Phase 1**: Fast search with `extract_flat` to get basic metadata
2. **Phase 2**: Full extraction only for promising candidates (after filtering)

This reduced search time significantly while maintaining quality.

## What Worked Well

### ✅ Multi-Agent Architecture
The separation of concerns into specialized agents made the system:
- **Maintainable**: Each agent has a single responsibility
- **Testable**: Agents can be tested independently
- **Extensible**: Easy to add new agents or modify existing ones

### ✅ Batch Processing
Reducing API calls from O(n) to O(1) for filtering and scoring:
- **10x faster** for large playlists
- **Significantly cheaper** (fewer API calls)
- **More reliable** (fewer points of failure)

### ✅ Energy-Based Smoothing
The smoothing algorithm creates natural transitions between songs, making playlists feel more cohesive than simple similarity-based ordering.

### ✅ Visual Feedback
The energy arc visualization helps users understand the playlist's emotional journey at a glance.

### ✅ Streamlit UI
Streamlit provided a quick way to build an interactive UI without frontend development. The progress bar and cancellation feature improved UX significantly.

## What Didn't Work (and What We Learned)

### ❌ Initial Single-Call Approach
**Problem**: Trying to do everything in one API call (generate phases, find songs, order them) resulted in:
- Inconsistent results
- Difficulty debugging
- Limited control over each step

**Solution**: Breaking it into specialized agents with clear responsibilities.

### ❌ Individual API Calls
**Problem**: Making separate API calls for each song was:
- Extremely slow (minutes for a playlist)
- Expensive (hundreds of API calls)
- Rate-limit prone

**Solution**: Batch processing wherever possible.

### ❌ YouTube Search Without Filtering
**Problem**: Initial searches returned many irrelevant results (playlists, mixes, long videos).

**Solution**: Multi-stage filtering:
1. Search query exclusions (`-playlist -mix`)
2. Title-based filtering (banned words)
3. Duration filtering (max 10 minutes)
4. View count filtering (optional)
5. Semantic filtering (emotional fit)

### ❌ No Deduplication Across Queries
**Problem**: Same songs appeared multiple times when different queries returned overlapping results.

**Solution**: Track `seen_video_ids` at the pipeline level, not just within each search.

### ❌ View Count Filtering Challenges
**Problem**: YouTube search doesn't support filtering by view count in the query. We can only filter after extraction, which means:
- Need to search many more results
- Slower for high view count ranges (10M-50M)
- May not find enough results

**Solution**: 
- Increased search multiplier for high view count ranges
- Enhanced search queries (add "official music video" for popular content)
- More search attempts

**Lesson**: Some limitations are inherent to the data source (YouTube API), not our implementation.

## Code Highlights

### Pipeline Orchestration

```python
class NarrativePlaylistPipeline:
    def __init__(self):
        self.narrative_agent = NarrativeAgent()
        self.query_agent = YouTubeQueryAgent()
        self.youtube_agent = YouTubeMusicAgent()
        self.filter_agent = YouTubeSemanticFilter()
        self.sequence_agent = SequenceDirectorAgent()
        self.narrator_agent = PlaylistNarratorAgent()
        self.color_agent = ColorAssignmentAgent()
    
    def run(self, user_prompt: str, cancel_cb=None, progress_cb=None):
        # Generate phases
        phases = self.narrative_agent.generate_phases(user_prompt)
        phase_colors = self.color_agent.assign_colors(phases)
        
        # Search and filter
        queries = self.query_agent.generate_queries(user_prompt)
        all_tracks = []
        for q in queries:
            all_tracks.extend(self.youtube_agent.search(q))
        
        # Filter and assign
        filtered = self.filter_agent.filter_matches(all_tracks, emotion_tags)
        buckets = self.sequence_agent.group_songs_by_phase(filtered, phases)
        
        # Order and smooth
        scored_playlist = self.sequence_agent.order_all_phases(buckets, phases)
        smoothed = self.smoothing_agent.smooth(scored_playlist)
        
        # Narrate
        story = self.narrator_agent.narrate(phases, smoothed)
        
        return {
            "phases": phases,
            "playlist": smoothed,
            "story_segments": story,
            "phase_colors": phase_colors
        }
```

### Batch Filtering Example

```python
class YouTubeSemanticFilter:
    def filter_matches(self, songs: List[Tuple[str, str]], emotion_tags: List[str]) -> List[bool]:
        # Format all songs for single prompt
        songs_list = "\n".join([
            f"{idx + 1}. Title: {title}\n   Description: {description[:200]}"
            for idx, (title, description) in enumerate(songs)
        ])
        
        prompt = f"""Determine which songs fit the emotional theme: {', '.join(emotion_tags)}
        
        Songs:
        {songs_list}
        
        Return ONLY a comma-separated list of YES/NO values."""
        
        resp = self.client.chat.completions.create(
            model=self.model,
            messages=[{"role": "user", "content": prompt}],
            temperature=0,
        )
        
        # Parse YES/NO list
        response_text = resp.choices[0].message.content.upper()
        # ... parsing logic ...
        
        return results  # List of booleans
```

### Smoothing Algorithm

```python
class SmoothingAgent:
    def smooth(self, scored_playlist: List[ScoredSong]) -> List[ScoredSong]:
        playlist = scored_playlist[:]
        i = 0
        
        while i < len(playlist) - 2:
            jump = abs(self._energy(playlist[i]) - self._energy(playlist[i + 1]))
            
            if jump <= self.jump_threshold:
                i += 1
                continue
            
            # Find best swap in window
            best_k = None
            for k in range(i + 2, min(len(playlist), i + 2 + self.max_window)):
                new_jump = abs(self._energy(playlist[i]) - self._energy(playlist[k]))
                if new_jump < jump:
                    best_k = k
            
            if best_k:
                # Swap songs
                chosen = playlist.pop(best_k)
                playlist.insert(i + 1, chosen)
            else:
                i += 1
        
        return playlist
```

## Tutorial 1: Setting Up the Project Locally

### Prerequisites

- Python 3.8 or higher
- OpenAI API key
- Google Cloud account (for YouTube playlist creation - optional)
- Git

### Step 1: Clone the Repository

```bash
git clone https://github.com/yourusername/narrative-playlist-agent.git
cd narrative-playlist-agent
```

### Step 2: Create a Virtual Environment

**Windows:**
```bash
python -m venv playlist_env
playlist_env\Scripts\activate
```

**macOS/Linux:**
```bash
python3 -m venv playlist_env
source playlist_env/bin/activate
```

### Step 3: Install Dependencies

```bash
pip install -r requirements.txt
```

This installs:
- `openai` - For GPT API calls
- `yt-dlp` - For YouTube search
- `streamlit` - For the web UI
- `matplotlib` - For visualizations
- `pydantic` - For data validation
- `google-api-python-client` - For YouTube playlist creation (optional)

### Step 4: Set Up Environment Variables

Create a `.env` file in the project root:

```bash
OPENAI_API_KEY=your_openai_api_key_here
```

Get your OpenAI API key from [platform.openai.com](https://platform.openai.com/api-keys).

### Step 5: (Optional) Set Up YouTube Playlist Creation

If you want to create YouTube playlists:

1. Go to [Google Cloud Console](https://console.cloud.google.com/)
2. Create a new project or select an existing one
3. Enable the YouTube Data API v3
4. Create OAuth 2.0 credentials (Desktop application)
5. Download `client_secret.json`
6. Place it in the `narrative_playlist/` directory

### Step 6: Run the Application

```bash
streamlit run narrative_playlist/app_streamlit.py
```

The app will open in your browser at `http://localhost:8501`.

### Troubleshooting

**Issue**: `ModuleNotFoundError: No module named 'narrative_playlist'`
- **Solution**: Make sure you're in the project root directory and the virtual environment is activated.

**Issue**: OpenAI API errors
- **Solution**: Verify your API key is set correctly in `.env` and you have sufficient credits.

**Issue**: YouTube search is slow
- **Solution**: This is normal. The system searches multiple queries and filters results. Be patient!

## Tutorial 2: Using the Tool

### Basic Usage

1. **Open the Application**
   - Run `streamlit run narrative_playlist/app_streamlit.py`
   - The app opens in your browser

2. **Enter Your Prompt**
   - In the text area, describe your emotional situation or vibe
   - Example: "Late-night reflective indie energy, but hopeful by the end"
   - Be specific about the emotional journey you want

3. **Generate Playlist**
   - Click "Generate playlist"
   - Watch the progress bar as the system:
     - Generates story phases
     - Searches YouTube
     - Filters and orders songs
     - Creates the narrative

4. **Explore Results**
   - **Playlist Story**: Read the narrative explaining the emotional arc
   - **Phases**: See the story phases with color indicators
   - **Playlist**: Browse the songs in order
   - **Energy Arc**: Visualize the emotional progression

### Advanced Features

#### Additional Search Terms

Add specific artists, genres, or keywords to guide the search:

- Example: "indie rock, acoustic, 2020s"
- These terms are added to all search queries
- Helps narrow down results to your preferences

#### View Count Filters

Use the "Advanced Search Filters" expander to filter by popularity:

- **Minimum views**: Filter out less popular songs
- **Maximum views**: Filter out very popular songs
- Useful for finding underground gems or mainstream hits

**Note**: High view count filters (10M+) may take longer and return fewer results, as YouTube search doesn't support view count filtering directly.

#### Creating YouTube Playlists

1. Generate a playlist
2. Click "Create YouTube Playlist"
3. Authorize the application (first time only)
4. The playlist is created on your YouTube account with:
   - Title: Your original prompt
   - Description: The generated story
   - Songs: All tracks in order

### Tips for Best Results

1. **Be Specific**: Instead of "sad songs," try "melancholic evening reflection after a long day"

2. **Describe the Journey**: Mention how you want the emotion to evolve:
   - "Start calm, build to energetic, end triumphant"
   - "Dark beginning, gradual hope, peaceful resolution"

3. **Use Search Terms**: Add genre/artist hints to guide the search:
   - "indie folk, Bon Iver, atmospheric"

4. **Experiment**: Try different prompts to see how the narrative changes

5. **Be Patient**: Generation takes 1-3 minutes depending on:
   - Number of search queries
   - YouTube search speed
   - Number of songs found

### Understanding the Output

#### Story Phases
Each phase represents a part of the emotional journey:
- **Opening**: Sets the initial mood
- **Rising Action**: Builds energy/emotion
- **Climax**: Peak emotional moment
- **Resolution**: Concludes the journey

#### Energy Arc
The visualization shows:
- X-axis: Track position in playlist
- Y-axis: Estimated energy level (0-1)
- Colors: Match the phase colors
- Shape: Shows the emotional progression

#### Playlist Story
A narrative text explaining how the playlist progresses, written like a music journalist describing the emotional arc.

## Performance Metrics

Based on testing, here are typical performance numbers:

- **Total Generation Time**: 1-3 minutes
- **Phase Generation**: ~2-5 seconds
- **YouTube Search**: ~30-60 seconds (depends on number of queries)
- **Filtering**: ~5-10 seconds
- **Phase Assignment**: ~3-8 seconds
- **Ordering**: ~5-15 seconds
- **Smoothing**: <1 second
- **Story Generation**: ~3-8 seconds

**API Costs** (approximate per playlist):
- GPT-4o-mini calls: ~$0.01-0.03
- Total: Very affordable for personal use

## Future Improvements

### Potential Enhancements

1. **Caching**: Cache search results and phase assignments for similar prompts
2. **Parallel YouTube Searches**: Search multiple queries concurrently
3. **User Feedback Loop**: Learn from user preferences to improve future playlists
4. **Multiple Music Sources**: Support Spotify, Apple Music, etc.
5. **Playlist Templates**: Pre-defined narrative templates (workout, study, sleep)
6. **Collaborative Playlists**: Multiple users contribute to phases
7. **Real-time Generation**: Stream results as they're generated

### Technical Debt

1. **Error Recovery**: Better handling of partial failures
2. **Rate Limiting**: More sophisticated handling of API rate limits
3. **Testing**: Comprehensive unit and integration tests
4. **Documentation**: API documentation for programmatic use
5. **Configuration**: Externalize hardcoded values (thresholds, multipliers)

## Conclusion

The Narrative Playlist Agent demonstrates how AI can create more than just recommendations—it can craft experiences. By combining multiple specialized agents, we've built a system that understands emotional narratives and translates them into music.

The journey wasn't always smooth. We learned that:
- Batch processing is crucial for performance
- Multi-agent architectures require careful orchestration
- User experience matters as much as technical excellence
- Some limitations are inherent to external APIs

But the result is a tool that creates playlists with meaning, stories, and emotional arcs—something that goes beyond algorithmic similarity.

Whether you're looking for a soundtrack to your late-night reflection or an energetic workout mix, the Narrative Playlist Agent can craft a playlist that tells your story.

---

## Resources

- **GitHub Repository**: [https://github.com/RisNag777/narrative-playlist-agent](https://github.com/RisNag777/narrative-playlist-agent)
- **OpenAI API Docs**: [https://platform.openai.com/docs](https://platform.openai.com/docs)
- **Streamlit Docs**: [https://docs.streamlit.io](https://docs.streamlit.io)
- **yt-dlp Docs**: [https://github.com/yt-dlp/yt-dlp](https://github.com/yt-dlp/yt-dlp)


---

*This project was built as a demonstration of AI-powered creative tools. Feel free to fork, modify, and build upon it!*
