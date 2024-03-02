---
title: "Hellbound: short Overview"
date: 2023-10-29 04:43:00 +0800
tags: []     # TAG names should always be lowercase
image: "https://img.itch.zone/aW1nLzEyNTQ3NDU2LnBuZw==/original/MBt6e9.png"
---

[Hellbound](https://buas.itch.io/hellbound) is a Hades-inspired game built on a custom engine that I have been working on since the start.
We got half a year to create an engine targeting top-down-action games with a team of 6 programmers.
After that half year we got teamed up with designers and artists. They came up with a game pitch that would be feasible with our custom engine.
We had a few weeks of pre-production and then 8 weeks to produce the final game!

I learned a lot from this experience, and my role in the team was of great importance to the project.<Br>
I was not directly involved with gameplay instead, My focus was on workflows to improve the productivity of the team.

## Levels
We [loaded levels from Houdini](https://www.artstation.com/artwork/w0K8yL).
One of our team members generates the levels and assigns special tags that we can read in the engine.
This allows Houdini to be used as a level editor (Or any GLTF file at the end of the day).
The engine gains information about everything it loads into the level from this.
Some of the things artists have full control over are: `Player/Enemy spawn point`, `Objects with collision`, `SpikeTraps`, `Checkpoints`, and `Lights`.

### performance improvements
Once we got artists on the team and they started to integrate their assets into the engine we saw our loading times becoming 10+ seconds one of the things I worked on was Improving this (to 1.5 seconds).<br>By profiling I discovered duplicate loading of textures and instances of the Gltf object being passed by value.

## Cross Platform
As we built this engine to work on pc and ps5 I came up with a way to deal with cross-platform code. A lot of functions can be defined in a cross-platform header and then linked with platform-specific .cpp files.<br> 
Every platform is its own Sub-project in the visual studio solution, giving us an easy way to write and maintain code in a platform-independent way.

## Editor
I created a [Dear Imgui](https://github.com/ocornut/imgui) Window that shows all [entt](https://github.com/skypjack/entt) entities and components with the use of code reflection.
This system has been open-sourced and can be found [here](https://github.com/TheDimin/EnttEditor)


<br><br>
Time to code,
- Damian