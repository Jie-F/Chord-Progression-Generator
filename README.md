# Chord Progression Generator

A simple **Svelte** composing tool that randomly gives you 4-chord progressions, which you can play and modify.

## Features

- Select root note, and scale (major/minor)
- Random chord generation with inversions
- Lock chords to keep them on re-roll
- Play/Stop
- Oscillator options (sine, triangle, square, sawtooth)
- Volume slider

## Install & Run

1. Clone this repo:
   ```
   git clone https://github.com/Jie-F/Chord-Progression-Generator.git
   cd <repo>
   ```
2. Install dependencies:
   ```
   npm install
   ```
3. Run in development:
   ```
   npm run dev
   ```

## Usage

1. Choose **Root**, **Scale**, **Sound**, **Volume**.
2. Click **“Roll”** to randomize any unlocked chords.
3. Press **“Play”** to hear and see the **playhead**. Press **“Stop”** to end.
4. **Lock** chords to save them on re-roll.

## Tech Stack

- **Svelte** (UI)
- **Tone.js** (audio synthesis)
 
![image](https://github.com/user-attachments/assets/2df175b8-534a-4741-b15d-432536a38b05)
