---
layout: post
title: "Building a Narrative Playlist Generator: From Concept to Reality"
---

![Narrative Playlist Agent](https://github.com/user-attachments/assets/f971de46-8745-4026-839e-523c28f887b7)

We've all used the AI-generated playlist features on Spotify or YouTube Music. You enter a prompt asking for a type of music that you want to listen to because you either don't want to spend the time looking for a specific playlist or you just have a vibe in mind (and a vibe is truly hard to search for). You enter your 'vibe' prompt and the app delivers as per expectation. For example: if my vibe is "sleepy", then YouTube Music gives me slow, soft songs. Almost like a lullaby to put me to sleep.
However, while using this feature, I began to imagine a different scenario. What if, instead of a static mood, you want a playlist that takes you on a journey, your own narrative arc. One that would give you the perfect cinema level soundtrack for your life! Prompting my app with the same "sleepy" vibe, generates a playlist with songs that take you from feeling drowsy to falling asleep to blissfully dreaming to the jolt of awakening and finally, the dawn of a new morning.

To bring my vision to life, I made use of a Production Crew of 8 specialized agents, each with a distinct role in the creative pipeline -
1. The Screenwriter - translates the user's vibe into 4-6 emotional phases like acts in a movie plot
2. The Visual Designer - assigns a color for each phase for visual consistency
3. The Conductor - crafts precise music search queries tailored to the specific mood of each act
4. The Auditor - ensures that the list of retrieved songs matches the theme
5. The Sequencer - assigns songs to the act that best fits them
6. The Producer - scores each song on `energy_level`, `emotional_weight` and `intimacy` (between 0 and 1)
7. The Mixing Engineer - ensures smooth transitions between tracks
8. The Narrator - writes a narrative that explains the flow of the playlist from phase to phase and within acts from song to song

In order to execute the above functionalities, the agents utilize the following tools - 
1. The Searcher Tool - queries the yt-dlp library to find videos that match the music search queries ensuring that no duplicate videos are picked and that videos with certain "banned" phrases (designed to ignore playlists, albums, hour long loops) are not returned
2. The Playlist Generator Tool - uses Google authentication to create the playlist in the user's YouTube and YouTube Music accounts.

My initial idea involved using the [Spotify API](https://developer.spotify.com/documentation/web-api) but they restrict usage of that to premium members. Since this was a personal project, I chose to make do with using the [yt-dlp api](https://github.com/yt-dlp/yt-dlp). Yt-dlp lacks a lot of the "music" metadata that I think would genuinely improve this project by leaps and bounds, but I do have a good working "alpha" implementation.

I wanted to give LangChain a try this time because of the sheer number of agents I was working with.
