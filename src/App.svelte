<script>
	import { onMount } from 'svelte';
	import * as Tone from 'tone';

	const rootNotes = ['C', 'C#', 'D', 'D#', 'E', 'F', 'F#', 'G', 'G#', 'A', 'A#', 'B'];
	const scaleOptions = ['Major', 'Minor'];

	let selectedRoot = 'C';
	let selectedScale = 'Major';

	// -------------------------------------------
	// 2) Sound options
	// -------------------------------------------
	// Let's define several oscillator types
	const soundOptions = ['sine', 'triangle', 'square', 'sawtooth'];
	let selectedSound = 'triangle';    // default

	// Volume in decibels. 0 dB is "no change," negative is quieter. 
	// Tone.js volume ranges from -60 dB up to +6 dB.
	let volume = -6;

	// Some chord "offset templates" in semitones:
	const chordTemplates = {
		major: [
			[0, 4, 7], // I (major triad)
			[2, 5, 9], // ii (minor triad) in major scale
			[4, 7, 11], // iii
			[5, 9, 12], // IV
			[7, 11, 14], // V
			[9, 12, 16], // vi
			[11, 14, 17] // vii°
		],
		minor: [
			[0, 3, 7], // i
			[2, 5, 9], // ii°
			[3, 7, 10], // III
			[5, 9, 12], // iv
			[7, 10, 14], // v/V
			[8, 12, 15], // VI
			[10, 14, 17] // VII
		]
	};

	let progression = [null, null, null, null];
	let locked = [false, false, false, false];

	// Piano roll range
	const pianoRollStart = 60; // C4
	const pianoRollEnd = 83; // B5
	const totalNotes = pianoRollEnd - pianoRollStart + 1;

	// Convert MIDI number to note name + octave
	function midiToNoteName(midi) {
		const noteNames = ['C', 'C#', 'D', 'D#', 'E', 'F', 'F#', 'G', 'G#', 'A', 'A#', 'B'];
		const octave = Math.floor(midi / 12) - 1;    
		const noteIndex = midi % 12;
		return noteNames[noteIndex] + octave;
	}

	function isBlackKey(midi) {
		const blackNotes = new Set(['C#', 'D#', 'F#', 'G#', 'A#']);
		const nameOnly = midiToNoteName(midi).replace(/[0-9]/g, ''); // e.g. "C#" 
		return blackNotes.has(nameOnly);
	}

	// Generate chord progression
	function rollProgression() {
		// Root note offset from C4 (which is MIDI 60)
		const rootIndex = rootNotes.indexOf(selectedRoot);
		const rootMidi = 60 + rootIndex;
		const scaleType = selectedScale.toLowerCase();

		const templateArray = chordTemplates[scaleType];

		progression = progression.map((chord, i) => {
		if (locked[i]) return chord;
		const randomTemplate = templateArray[Math.floor(Math.random() * templateArray.length)];
		return randomTemplate.map(offset => rootMidi + offset);
		});
	}

	// -------------------------------------------
	// Play chords in succession
	// -------------------------------------------
	async function playProgression() {
		await Tone.start();
	
		// Create a Volume node to control the overall volume in dB
		const volumeNode = new Tone.Volume(volume).toDestination();

		const synth = new Tone.PolySynth(Tone.Synth, {
		oscillator: { type: selectedSound },
		envelope: { attack: 0.02, release: 1 }
		}).connect(volumeNode);
	
		let now = Tone.now();
		for (let i = 0; i < progression.length; i++) {
		if (progression[i] !== null) {
			const chordNotes = progression[i].map(m => Tone.Frequency(m, 'midi').toNote());
			synth.triggerAttackRelease(chordNotes, 1.8, now);
		}
		now += 2; // Play for 2 seconds
		}
	}

	// Svelte lifecycle + initial roll
	onMount(() => {
		rollProgression();
	});
</script>
	
<style>
	.container {
		display: flex;
		flex-direction: column;
		gap: 1rem;
		max-width: 700px;
		margin: 1rem auto;
		font-family: sans-serif;
	}
	.controls {
		display: flex;
		flex-wrap: wrap;
		gap: 0.5rem;
		align-items: center;
	}
	select, button {
		font-size: 1rem;
		cursor: pointer;
	}
	.locked {
		opacity: 0.6;
	}
	
	/* The main "piano roll" container: We have 1 column for the piano keys, plus 4 columns for the chords => 5 columns total. */
	.piano-roll-container {
		display: grid;
		grid-template-columns: auto repeat(4, 1fr); /* first column = piano keys, then 4 chord columns */
		border: 1px solid #ddd;
	}
	
	/* The piano-key-column has the same number of rows as our note grid. */
	.piano-key-column {
		display: grid;
		grid-template-rows: repeat(var(--rows), 1em);
		border-right: 2px solid #aaa;
	}
	.piano-key {
		position: relative;
		display: flex;
		align-items: center;
		justify-content: flex-end;
		padding-right: 0.3rem;
		border-bottom: 1px solid #eee;
		font-size: 0.8rem;
	}
	.black-key {
		background: #444;
		color: white;
	}
	.white-key {
		background: #fafafa;
	}

	.piano-column {
		display: grid;
		grid-template-rows: repeat(var(--rows), 1em);
		border-right: 1px solid #ddd;
		position: relative;
	}
	.piano-note {
		border-bottom: 1px solid #eee;
		background-color: white;
	}
	.piano-note.active {
		background-color: rgb(255, 149, 0);
	}
	.volume-slider {
		margin-left: 1rem;
		width: 100px;
	}
</style>

<div class="container">
	<!-- Controls row -->
	<div class="controls">
		<!-- Dropdown: Root Note -->
		<label for="rootSelect">Root:</label>
		<select id="rootSelect" bind:value={selectedRoot}>
		{#each rootNotes as note}
			<option value={note}>{note}</option>
		{/each}
		</select>
	
		<!-- Dropdown: Scale Type -->
		<label for="scaleSelect">Scale:</label>
		<select id="scaleSelect" bind:value={selectedScale}>
		{#each scaleOptions as sc}
			<option value={sc}>{sc}</option>
		{/each}
		</select>

		<!-- Dropdown: Sound Options (oscillator types) -->
		<label for="soundSelect">Sound:</label>
		<select id="soundSelect" bind:value={selectedSound}>
		{#each soundOptions as s}
			<option value={s}>{s}</option>
		{/each}
		</select>
	
		<!-- Volume slider (range: -60 to 0 dB) -->
		<label for="volumeSlider">Volume:</label>
		<input
		id="volumeSlider"
		class="volume-slider"
		type="range"
		min="-60" max="0" step="1"
		bind:value={volume}
		/>

		<!-- Roll and Play -->
		<button on:click={rollProgression}>🎲 Roll</button>
		<button on:click={playProgression}>▶ Play</button>
	</div>

	<!-- Chord lock/unlock row -->
	<div class="controls">
		{#each progression as chord, i}
		<button
			class={locked[i] ? 'locked' : ''}
			on:click={() => locked[i] = !locked[i]}>
			{locked[i] ? '🔒' : '🔓'} 
			{chord ? chord.map(midiToNoteName).join('-') : 'N/A'}
		</button>
		{/each}
	</div>

	<!-- Piano roll visualization: First column: vertical piano keys. Next 4 columns: chord grid 
		 Define --rows in the style so each "row" is 1em high. 
		 Top row is the highest note (pianoRollEnd), and the bottom row is the lowest (pianoRollStart).
	-->
	<div class="piano-roll-container" style="--rows: {totalNotes};">
		<!-- Left piano keys column -->
		<div class="piano-key-column">
		{#each Array(totalNotes) as _, rowIndex}
		<div class="piano-key {isBlackKey(pianoRollEnd - rowIndex) ? 'black-key' : 'white-key'}">
			{midiToNoteName(pianoRollEnd - rowIndex)}
		</div>
		{/each}
	</div>

	<!-- Four chord columns -->
		{#each progression as chord, colIndex}
			<div class="piano-column">
				{#each Array(totalNotes) as _, rowIndex}
					<!-- The 'active' cell if chord includes this note's MIDI. -->
					<div
					class="piano-note {chord && chord.includes(pianoRollEnd - rowIndex) ? 'active' : ''}"
					/>
				{/each}
			</div>
		{/each}
	</div>
</div>
