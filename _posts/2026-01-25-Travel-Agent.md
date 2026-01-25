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
Multimodal ingestion → Semantic memory → Planning agent → Writing agent  

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
