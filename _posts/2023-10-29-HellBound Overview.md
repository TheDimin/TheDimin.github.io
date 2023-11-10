---
title: "Hellbound: Overview"
date: 2023-10-29 04:43:00 +0800
tags: []     # TAG names should always be lowercase
image: "https://img.itch.zone/aW1nLzEyNTQ3NDU2LnBuZw==/original/MBt6e9.png"
---

[Hellbound](https://buas.itch.io/hellbound) is a hadies inspired game build on a custom cross platform engine made by us.
We got half a year to create an engine targeting top-down-action games with a team of 6 programmers.
After the intial half year we got teamed up with designers and artists, to come up with a game pitch for the engine & then build it in 8 weeks!

I learned a lot of this experience, and my role in the team was great importance to the project.
I was not directly involved with gameplay code, instead My focus was on workflows & performance for the team.

## Levels
We [loaded levels from houdini](https://www.artstation.com/artwork/w0K8yL).
One of our team members generates the levels and assigned special data tags that we can read in engine.
This allows Houdini to be used as level editor (Or any gltf file at the end of the day).
The engine gains information about everything it loads into the level from this.
Some of the things artists have full control over: `Player/Enemy spawn`, `Objects with collision`, `SpikeTraps`, `Checkpoints`,`Lights`.

### performance improvements
Once we got artists on the team and they started to intergrate their assets into the engine we saw our loading times becoming 10+ seconds one of the things I worked on was Improve this (to 1.5 second), by profiling I discovered duplicate loading of textues and instances of the Gltf object being passed by value.

## Cross Platform
As we build this engine to work on pc and ps5 I came up with a way to deal with cross platform code. A lot of funcitons can be defined in a cross platform header and then be linked with platform specific .cpp files. 
Every platform is its own SubProject in the visual studio solution, giving us an easy way to write and maintain code in a platform independed way.

## Editor
I created a imgui Window that shows all entt entities and components with the use of code reflection.
This system has been opensourced and can be found [here](https://github.com/TheDimin/EnttEditor)
