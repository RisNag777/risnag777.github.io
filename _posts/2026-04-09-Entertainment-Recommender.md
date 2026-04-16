---
layout: post
title: "Crafting an Entertainment Oracle"
---

<div align="center">
  <img src="https://github.com/user-attachments/assets/513ebe51-66cc-4529-9d56-8fab4f75f7b5" 
       style="border: 1px solid #d0d7de; border-radius: 8px; box-shadow: 0 4px 6px rgba(0,0,0,0.1); width: 500px;">
</div><br>

When we use Netflix, or YouTube, or Spotify, the algorithm that tells recommends our next watch or listen is platform dependent. It makes sense, they want to keep you on their app. However, that prevents these recommendations from being truly multimodal. I wanted to build a recommender that you could report to about a piece of media that you liked and it would give you a multimodal recommendation ie, a book, a tv show, a movie, a song, an album and a podcast. I also wanted to treat the user's "taste" as an evolving beast. It's too simplistic to say that I like rock music or comedy movies. Within those genres, there's a lot of nuance that gets missed and could lead to poor recommendations.

<div align="center">
  <img src="https://github.com/user-attachments/assets/017bc528-9094-403b-ad3e-d652bd163311"
       style="border: 1px solid #d0d7de; border-radius: 8px; box-shadow: 0 4px 6px rgba(0,0,0,0.1); max-width: 80%;">
  <p><i>The "Like" entered by the user</i></p>
</div>

<div align="center">
  <div style="display: inline-block; border: 1px solid #d0d7de; border-radius: 8px; box-shadow: 0 4px 6px rgba(0,0,0,0.1); max-width: 80%; overflow: hidden;">
    <img src="https://github.com/user-attachments/assets/bba6d688-b3ac-4189-a738-9a1bca04f9af" 
         style="width: 100%; display: block;">
    <img src="https://github.com/user-attachments/assets/00955947-77a4-470f-8704-55d5eb5d079a" 
         style="width: 100%; display: block; border-top: 1px solid #f0f0f0;">
  </div>
  <p><i>The generated recommendation grid</i></p>
</div>

<div align="center">
  <img src="https://github.com/user-attachments/assets/955e2efb-f8a4-46d7-ae79-9860581d9ed4" 
       style="border: 1px solid #d0d7de; border-radius: 8px; box-shadow: 0 4px 6px rgba(0,0,0,0.1); max-width: 80%;">
  <p><i>The updated taste profile after the generated recommendations</i></p>
</div>

### The Inner Workings of the Oracle
There are 4 specialized agents that work in tandem to make this possible - 
1. **The Entertainment Brain** - When the user shares a piece of media and what they like about it, this agent analyzes the underlying traits and generates themes that the agent uses to find recommendations across different forms of media. The agent then recommends 6 distinct pieces of media based on the input prompt from the user. The prompt given to the agent includes a schema which strictly defines the format of the output to ensure that the rest of the system can actually use the generated output.
2. **The Normalizer** - This is really important because the traits - "dark atmosphere" and "moody setting" mean roughly the same thing and I wanted to ensure that my system treated it as such. So, before any trait is saved to memory, this agent maps the traits to a consistent set of themes.
3. **The Memory Manager** - Once the traits are normalized, this agent updates the user's memory vault with the new traits extracted from the user. Interests for the user's tastes are adjusted, creating a dynamic vector that shows how their tastes have evolved over time. 
4. **The Tool Dispatcher** - While the Entertainment Brain makes a plan, this agent executes that plan. It acts as an MCP (Model Context Protocol) to retrieve metadata from different APIs: [TMDB (The Movie Database)](https://www.themoviedb.org/) for movies and TV shows, [LastFM](https://www.last.fm/home) for songs and albums, [Google Books](https://books.google.com/) for books and [iTunes](https://www.apple.com/itunes/) for podcasts. If the recommendation could not be found on the above APIs (if the agent looked for "Avengers 2" rather than "Avengers: Age of Ultron" or if the recommended song was too niche for LastFM), the Dispatcher signals the Entertainment Brain to try again and to come up with a different recommendation while following the same constraints as last time. This feedback loop is crucial as it ensures that the recommendation grid is never broken, making the system significantly robust.

<div align="center">
  <img src="https://github.com/user-attachments/assets/58838203-a6f4-4642-b9c9-b77d88ed849d"
       style="border: 1px solid #d0d7de; border-radius: 8px; box-shadow: 0 4px 6px rgba(0,0,0,0.1); max-width: 80%;">
  <p><i>System Architecture</i></p>
</div>

### Future Roadmap
- User Feedback: Allow a user to "rate" the system's recommendations
- Expand to a more structured database: Currently I am just using a JSON file to track user tastes
- Collaborative Filtering: Along with more users, allow the Memory Manager to find "Taste Twins", to allow for "Users who liked x also liked y" recommendations

