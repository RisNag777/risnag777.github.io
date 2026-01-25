---
layout: post
title: "Travel Blog Agent"
---

# The Inspiration
After returning from an Italy trip, my wife and I got into our usual habit of posting pictures from our trip to Instagram and we got the usual replies from friends and relatives asking us for our itinerary. Earlier, I would put the details of our flights, hotels and activities into a Google sheet and share the sheet with anyone who asked. Now though, I wanted to do something a little different. I wanted to go at it from an agentic point of view. So here's my project to create an agent that takes photos, videos, text and csv/xlsx files as inputs and gives me a ready-to-post travel blog!  

# The Plan
The agent should:  
1. Ingest multiple modalities
2. Convert everything into structured representations
3. Plan a blog narrative
4. Write a cohesive travel blog using those representations

## The Architecture
Multimodal ingestion -> Semantic memory -> Planning agent -> Writing agent  

## The Inputs
Everything is converted into one unified schema making the agent modality-agnostic.  

### 1. Text
1. The main text input is WhatsApp messages I share with my family back home to keep them up-to-date with our daily activities.
2. Another text input is reviews of hotels, activities and restaurants from Google, GetYourGuide, Booking.com, etc.

#### Processing
1. Chunk text
2. Extract entities (locations, date, emotions, activities)

### 2. CSV/XLSX Files
1. Flight details
2. Hotel details
3. Activity details
4. Itinerary details

#### Processing
1. Pandas to read the data
2. Normalize dates
3. Summarize patterns

### 3. Photos
1. Landmarks
2. Food
3. Street shots

#### Processing
The tool won't embed raw images but will embed their descriptions
1. Landmark recognition
2. Scene classification
3. Object detection
4. Mood inference

### 4. Videos
1. Panoramic videos
2. Activity Recordings

#### Processing
1. Extract frames (every N seconds)
2. Transcribe audio (Whisper-style)
3. Summarize scenes

## The Agent Stack
### Agent 1: Ingestion Agent
#### Role
1. Detect file types
2. Route to correct parser
3. Normalize outputs

#### Tools
1. pandas (csv/xlsx)
2. Vision model (images)
3. Whisper (video/audio)
4. Embeddings

### Agent 2: Memory Builder
Stores everything in:
1. Vector DB (FAISS / Chroma)
2. Metadata store (JSON)  

This allows queries like:
1. "Which was the best city?"
2. "Show me highlights from Rome"

### Agent 3: Blog Planner Agent
Creates the outline:
1. Arrival & First Impressions
2. Food Adventures
3. Hidden Corners
4. Budget & Logistics
5. Final Reflections

It explicitly decides:
1. What content goes where
2. What images/videos support each section

### Agent 4: Writing Agent
Prompted with:
1. Section goal
2. Retrieved memories
3. Desired tone (personal / poetic / practical)
   Example:
   "Write a reflective paragraph about Venice evenings using serene images and low-spend insights."

## Minimal tech stack
### Backend
1. Python
2. LangGraph or plain orchestration
3. FAISS
4. pandas
5. ffmpeg (video frames)
   
### Models
1. One strong LLM
2. One vision-capable model
3. One speech-to-text

# Project Folder Structure
travel-blog-agent/
│
├── agents/
│   ├── ingestion_agent.py
│   ├── memory_agent.py
│   ├── planner_agent.py
│   └── writer_agent.py
│
├── config/
│   ├── blog_stle.yml
│   ├── models.yml
│   └── paths.yml
│
├── data/
│   ├── canonical/
│   │   └── travel_memory.json
│   │
│   ├── processed/
│   │   ├── images.json
│   │   ├── tabular.json
│   │   ├── text.json
│   │   └── videos.json
│   │
│   ├── raw/
│   │   ├── images/
│   │   ├── tabular/
│   │   ├── text/
│   │   └── videos/
│   │
│   ├── processed/
│   │   ├── images.json *
│   │   ├── processed_memories.json
│   │   ├── tabular.json *
│   │   ├── text.json *
│       └── videos.json *
│
├── outputs/
│   ├── assets/ *
│   ├── drafts/ *
│   ├── final/ *
│   │   ├── blog.html *
│       └── blog.md *
│
├── pipelines/
│   ├── build_memory.py *
│   ├── generate_outline.py *
│   ├── ingest.py
│   └── write_blog.py *
│
├── schemas/
│   ├── canonical_schema.json
│   ├── image_schema.json *
│   ├── tabular_schema.json *
│   ├── text_schema.json *
│   └── video_schema.json *
│
├── tests/
│   ├── test_ingestion.py *
│   ├── test_memory.py *
│   └── test_writer.py *
│
├── utils/
│   ├── chunking.py *
│   ├── embeddings.py
│   ├── file_router.py
│   ├── image_parser.py
│   ├── planner_prompts.py
│   ├── prompts.py *
│   ├── retrieval.py
│   ├── tabular_parser.py
│   ├── text_parser.py
│   ├── video.py *
│   ├── video_parser.py
│   ├── vision.py *
│   └── writer_prompts.py
│
├── vectorstore/
│   └── faiss_index/ *
│
├── main.py *
├── requirements.txt
└── README.md *

## 1. data/raw/
1. Dumb storage only
2. No logic
3. No assumptions
4. Just files dropped in

## 2. data/processed/
Each modality gets normalized before mixing:
1. Images -> captions, tags
2. Videos -> transcripts, summaries
3. CSV -> insights
This prevents cross-modal leakage bugs.

## 3. data/canonical/
Everything ends up here in one json format  
If models are later changed? - Canonical format stays the same.  

## 4. agents/ vs pipelines/
Agents = intelligence  
Pipelines = orchestration  
Example:  
1. ingestion_agent.py knows how to parse images
2. ingest.py decides when and in what order

## 5. schemas/
All inter-agent communication is schema-validated

## 6. utils/
Reusable, boring, critical stuff:
1. chunking
2. embeddings
3. ffmpeg wrappers
4. prompt templates
The agents stay clean and readable.

## 7. outputs/
Split drafts vs final so you can:
1. regenerate sections
2. compare tone changes
3. iterate safely

## 8. tests/
Includes basic tests like:
1. "Image -> caption exists"
2. "Canonical schema validates"

## 9. main.py
ingest -> build memory -> plan -> write -> export

# Canonical Travel Memory Schema
## Design goals
1. Modality-agnostic
2. Time & location aware
3. Easy to retrieve + write from
4. Stable even if models change

## 1. days[] is the natural narrative spine
Travel blogs are chronological by default. This lets the planner agent do:
1. Day-by-day blogs
2. Thematic reshuffles

## 2. summary is helpful for retrievak
One-line summaries make vector search much better.  
Example:  
"A slow, rainy morning wandering through Kyoto temples"  
This is embedded  

## 3. memory_item is deliberately loose
This is key for flexibility.  
The key design choice - content is opaque to the canonical layer  
Each modality defines its own structure inside content.

## How agents use this
### Planner Agent
1. Reads summary, tags, importance_score
2. Chooses which memories anchor each section

### Writer Agent
1. Never sees raw files
2. Only sees retrieved memory items
3. Produces cohesive prose

## Intentional omissions
Schema excludes:
1. Model names
2. Prompt text
3. Embeddings
4. UI concepts
Because these can change but the schema shouldn't

# Build Ingestion Agent
This agent will:
1. Scan data/raw/
2. Detect modality
3. Parse text + CSV/XLSX + images + videos
4. Emit:
data/processed/*.json
data/canonical/travel_memory.json
Vision and video are kept as pluggable stubs to allow for future model swaps

# Build Memory Builder Agent
This agent will:
1. Read canonical/travel_memory.json
2. Creates semantic representations - \[DAY\] + \[LOCATION\] + \[SUMMARY\] + \[TAGS\] + \[MODALITY\]
3. Stores them in a vector store (FAISS)
4. Enables natural-language retrieval across all modalities

# Build Planner Agent
This agent decides what story to tell and why, using the semantic memory
Given:
1. A blog goal
2. Access to semantic memory
It produces:
1. A structured outline
2. With explicit memory references per section
3. And a tone plan
4. No vibes. No randomness.

## Planner ≠ Retriever
The Planner:
1. does not retrieve memories
2. does not write paragraphs
3. It only decides - "What should be talked about, and in what order?"
The Writer Agent will later:
1. Execute each section
2. Retrieve memories
3. Generate prose
This separation is what prevents:
1. repetitive blogs
2. info dumps
3. hallucinated structure

This system separates narrative planning from semantic retrieval and prose generation.

# Build Writer Agent
The agent will:
1. Take one planned section
2. Retrieve relevant memories semantically
3. Ground the prose in actual moments
4. Write cohesive, non-generic travel writing

The Writer Agent works section-by-section so it can:
1. Maintain focus
2. Avoid repetition
3. Adapt tone per section

This Writer Agent is good because it is:
1. Grounded- It writes only from retrieved memories
2. Modular -
   i. Rewrite one section
   ii. Change tone halfway through
   iii. Regenerate without touching others
3. Non-repetitive - Each section has different retrieval queries.

# Full blog generation loop
'''
blog = f"# {plan['title']}\n\n"

for section in plan["sections"]:
    blog += writer.write_section(
        section=section,
        tone=["introspective", "warm"],
        length="medium"
    )
    blog += "\n\n"
'''

This system ingests multimodal travel data, builds semantic memory, plans narrative structure, and generates grounded long-form content using retrieval-augmented agents.
