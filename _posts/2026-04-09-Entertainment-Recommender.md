---
layout: post
title: "Crafting an Entertainment Oracle"
---

<div align="center">
  <img src="https://github.com/user-attachments/assets/f971de46-8745-4026-839e-523c28f887b7" 
       style="border: 1px solid #d0d7de; border-radius: 8px; box-shadow: 0 4px 6px rgba(0,0,0,0.1); width: 500px;">
</div><br>

When we use Netflix, or YouTube, or Spotify, the algorithm that tells recommends our next watch or listen is platform dependent. It makes sense, they want to keep you on their app. However, that prevents these recommendations from being truly multimodal. I wanted to build a recommender that you could report to about a piece of media that you liked and it would give you a multimodal recommendation ie, a book, a tv show, a movie, a song, an album and a podcast. I also wanted to treat the user's "taste" as an evolving beast. It's too simplistic to say that I like rock music or comedy movies. Within those genres, there's a lot of nuance that gets missed and could lead to poor recommendations.

### The Inner Workings of the Oracle
There are 4 specialized agents that work in tandem to make this possible - 
1. The Entertainment Brain - When the user shares a piece of media and what they like about it, this agent analyzes the underlying traits and generates some normalized themes that could work to find recommendations across different forms of media. The agent then recommends the 6 pieces of media based on the input prompt from the user. The prompt given to the agent includes a schema which strictly defines the format of the output to make it simpler for the rest of the code to work with.
2. The Tool Dispatcher - When the Entertainment Brain makes its initial selections for media recommendations,  this agent acts as an MCP to retrieve metadata from different APIs: [TMDB (The Movie Database)](https://www.themoviedb.org/) for movies and TV shows, [LastFM](https://www.last.fm/home) for songs and albums, [Google Books](https://books.google.com/) for books and [iTunes](https://www.apple.com/itunes/) for podcasts. If the recommendation could not be found on the above APIs (if the agent looked for "Avengers 2" rather than "Avengers: Age of Ultron" or if the recommended song was too niche for LastFM), the Entertainment Brain is instructed to try again and to come up with a different recommendation while following the same constraints as last time.
3. The Memory Manager - This agent updates the memory with the new traits extracted from the user. This is to give the user an overall view of how their tastes have grown over time.
4. The Normalizer - This is really important because the traits - "dark atmosphere" and "moody setting" mean roughly the same thing and I wanted to ensure that my system treated it as such. So, before any trait is saved to memory, this agent maps the traits to a consistent set of themes.

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

<div align="center">
  <img src="https://github.com/user-attachments/assets/a3f95bbe-5a2e-4246-883b-dce0f18060da" 
       style="border: 1px solid #d0d7de; border-radius: 8px; box-shadow: 0 4px 6px rgba(0,0,0,0.1); width: 300px;">
</div>

### Future Roadmap
- User Feedback: Allow a user to "rate" the system's recommendations
- Expand to a more structured database: Currently I am just using a JSON file to track user tastes

