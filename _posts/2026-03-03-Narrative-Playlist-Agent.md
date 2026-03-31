---
layout: post
title: "Building a Narrative Playlist Generator: From Concept to Reality"
---

<div align="center">
  <img src="https://github.com/user-attachments/assets/f971de46-8745-4026-839e-523c28f887b7" 
       style="border: 1px solid #d0d7de; border-radius: 8px; box-shadow: 0 4px 6px rgba(0,0,0,0.1); width: 500px;">
</div><br>

We've all used the AI-generated playlist features on Spotify or YouTube Music. You enter a prompt asking for a type of music that you want to listen to because you either don't want to spend the time looking for a specific playlist or you just have a vibe in mind (and a vibe is truly hard to search for). You enter your 'vibe' prompt and the app delivers as per expectation. For example: if my vibe is "sleepy", then YouTube Music gives me slow, soft songs. Almost like a lullaby to put me to sleep.
While using this feature, I began to imagine a different scenario. What if, instead of a static mood, you want a playlist that takes you on a journey, your own narrative arc. One that would give you the perfect cinema level soundtrack for your life! Prompting my app with the same "sleepy" vibe, generates a playlist with songs that take you from feeling drowsy to falling asleep to blissfully dreaming to the jolt of awakening and finally, the dawn of a new morning. (See generated playlist [here](https://www.youtube.com/playlist?list=PLCLdnNkwmzcqN-NAM9odrE4VfeDxUMDIv))

<div align="center">
  <img src="https://github.com/user-attachments/assets/da04256e-ec60-4d9a-bb02-2f69ec7c3d4a" 
       style="border: 1px solid #d0d7de; border-radius: 8px; box-shadow: 0 4px 6px rgba(0,0,0,0.1); max-width: 80%;">
  <p><i>The narrative flow for the generated playlist</i></p>
</div>

<div align="center">
  <img src="https://github.com/user-attachments/assets/ef1e16ea-1a78-4ad6-b5cd-0a7dbf416f44" 
       style="border: 1px solid #d0d7de; border-radius: 8px; box-shadow: 0 4px 6px rgba(0,0,0,0.1); max-width: 80%;">
  <p><i>The energy arc for the generated playlist</i></p>
</div>

### The Production Crew
To bring my vision to life, I made use of a Production Crew of 8 specialized agents, each with a distinct role in the creative pipeline -
1. The Music Supervisor - translates the user's vibe into 4-6 emotional phases (like acts in a movie plot)
2. The Colorist - assigns a color to each phase for visual consistency
3. The Conductor - crafts precise music search queries tailored to the specific mood of each phase
4. The Quality Control Engineer - ensures that the list of retrieved songs matches the theme
5. The Sequencer - assigns songs to the phase that best fits the narrative flow
6. The Producer - scores each song on `energy_level`, `emotional_weight` and `intimacy` (between 0 and 1)
7. The Mixing Engineer - ensures smooth transitions between tracks
8. The Narrator - writes a narrative that explains the flow of the playlist from phase to phase and within each phase from song to song

#### The Tools
These agents are integrated into a sequential pipeline that leverages two specialized tools - 
1. The Searcher Tool - queries the yt-dlp library to find videos that match the Conductor's queries ensuring that no duplicate videos are picked and that videos with certain "banned" phrases (designed to ignore playlists, albums, hour long loops) are not returned
2. The Playlist Generator Tool - uses Google OAuth2 authentication to create and populate the playlist in the user's YouTube and YouTube Music accounts.

### The Pivot
My initial idea relied on the [Spotify API](https://developer.spotify.com/documentation/web-api), where I could access use musical metadata like `energy` and `danceability` to drive my Producer and Mixing Engineer agents. However, Spotify restricts usage to premium members only. Since this was a personal project, I chose to pivot to using the [yt-dlp library](https://github.com/yt-dlp/yt-dlp). Yt-dlp lacks the deep metadata that Spotify provides but it offered a more open and diverse catalog which forced my agents to "work harder". The Producer now has to infer the musical metadata from narrative context and song titles. This resulted in a good working "alpha" implementation.

### The Architecture
Because of the scale of an eight-agent "Production Crew," I transitioned from manual orchestration to LangChain and LCEL (LangChain Expression Language) to move toward a composable, observable pipeline. The entire process is encapsulated as a unified object. This architecture allows me to treat the playlist generator as a structured runnable that can be consistently invoked, traced, and unit-tested at any individual node, such as the YouTube search or the semantic filtering steps. LangChain also provides the critical LLM plumbing needed for clean prompt templating and standardized calling patterns. The "state management" is handled through a shared Python dictionary that is enriched step-by-step as it flows through the LCEL chain. This ensures that the complex data (act structures, energy scores, song metadata) is passed smoothly through the pipeline. This modular approach makes the current system more robust as well as "future-ready," allowing me to easily swap models or transition to a more complex LangGraph implementation with retries and parallel loops without a total code refactor.

<div align="center">
  <img src="https://github.com/user-attachments/assets/a3f95bbe-5a2e-4246-883b-dce0f18060da" 
       style="border: 1px solid #d0d7de; border-radius: 8px; box-shadow: 0 4px 6px rgba(0,0,0,0.1); width: 300px;">
</div>

### Future Roadmap
The current "alpha" implementation is solid, but I've thought about some ways to potentially enhance this project for greater usability
- Audio Analysis: Using libraries like Librosa to actually "listen" to the YouTube audio and generate that missing energy and mood metadata
- Refined Filtering: Using an LLM to scan the YouTube comments and video descriptions for even better mood verification
- User Feedback Loop: Allowing the user to "thumbs up/down" a song to help the Auditor agent learn their personal taste
