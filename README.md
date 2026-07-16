# Projects
These are projects that I have planned out to work on at a later date. They are all primarily for the purpose of personal development. Any projects that I intend to sell witll have "(£)" in the title. All these plans are subject to copyright. Copyright © 2026 Toby Hunter, All rights reserved.

**This was last updated on the 19/07/2026**

## Blackbox
### Summary
This is an idea for an automated diagnostics system baked into all applications I produce. The idea is based on the communications planes make with servers on the ground with which they pass diagnostic and locational data.

### Flow
- Application opens
- Application calls API and receives a session id
- Upon encountering an error, application sends the error and parameters to the API
- A small LLM analyses the error and either raises a GitHub item or appends the example to the existing GitHub item
- A text notification is sent to inform me of the error

### Data Gathered
The only data gathered will be the error and any parameters involved in causing that error

### Notes
This will be my first step into self-hosting an LLM and in-depth LLM usage.

## Password Manager
### Summary
In a bid to take my data offline, I have created this idea for a self-hostable password manager. The idea is that the data is stored in a database and served to the site by an API. The site and API are only accessible from approved devices.

### Notes
This is my first step into coding data encryption through the use of existing libraries.

## Self-Hosted LLM
LLMs like Claude are a useful tool for a developer to have in their toolbox. Due to data and environmental impact concerns, I have created this idea to host an LLM at home to use as my personal coding agent. It will only be accessible on approved devices through device registration. The plan is to make this as cost-effective but usable as possible, not compete with the likes of Claude and ChatGPT.

## Virtual Assistant
### Summary
Growing up, I always thought about creating my own AI assistant like Cortana from Halo and Jarvis from Marvel. At the time, Cortana was the option on Windows, and I disliked the fact that it needed internet to do things that didn't require it, like opening an application. With how valuable personal data is, I have redesigned and picked up this project, which I originally started in 6th form around 2020.

### Goal
Create a self-hostable virtual assistant that can be used as an alternative to current corporate options using an LLM to handle responses and dispatching of requests.

### Types
At present, I have two main assistant types in mind: standalone and multi-device. The standalone operates on a single machine, whereas the multi-device uses a server run by the user which clients on the machines talk to.

### Notes
There are many aspects of this project that will have me learning new things, specifically in the LLM world. While there are some features, such as the evolution of the assistant over time and the group chat feature, that seem excessive, it is there for the purpose of personal development.

## Bones and Brew (£)
### Summary
This is an arcade-style game based on the dice game Farkle. It features ranked/online play as well as invite/group/AI play.

### Modes
- Ranked is a 1 vs 1 game where two players play against each other; a score is attributed, which then contributes to the leaderboard.
- AI is a 1 vs 1 game against the game itself over three difficulty levels. Players in the hard game mode have the chance to encounter special dice that, if won, can be used in non-ranked games.
- Invite games have between 2 and 6 players who face off in a custom game with a custom score target they must hit to win.

### Features
- Achievements
- Levels
- Special Dice
- Local Play
- Online Play
- No Chat

### Notes
While this is currently intended to be sold for between £2.5 and £5, that may change following discussions with a Solicitor/Laywer on the legalities.
