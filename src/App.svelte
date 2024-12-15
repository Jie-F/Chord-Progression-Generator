<script>
	import { onMount } from 'svelte';
	import * as Tone from 'tone';

	const rootNotes = ['C', 'C#', 'D', 'D#', 'E', 'F', 'F#', 'G', 'G#', 'A', 'A#', 'B'];
	const scaleOptions = ['Major', 'Minor'];

	let selectedRoot = 'C';
	let selectedScale = 'Major';

	// Oscillator types
	const soundOptions = ['sine', 'triangle', 'square', 'sawtooth'];
	let selectedSound = 'triangle';    // default

	// Volume in decibels. 0 dB is "no change," negative is quieter. 
	// Tone.js volume ranges from -60 dB up to +6 dB.
	let volume = -6;

	// Single, persistent Volume node:
	// Will attach it to the Tone.Destination (master output).
	let volumeNode = new Tone.Volume(volume).toDestination();

	// Build a single, persistent PolySynth, connected to that volume node:
	let synth = new Tone.PolySynth(Tone.Synth, {
		oscillator: { type: selectedSound },
		envelope: { attack: 0.02, release: 1 }
	}).connect(volumeNode);

	// Some chord "offset templates" in semitones:
	// const chordTemplates = {
	// 	major: [
	// 		[0, 4, 7], // I (major triad)
	// 		[2, 5, 9], // ii (minor triad) in major scale
	// 		[4, 7, 11], // iii
	// 		[5, 9, 12], // IV
	// 		[7, 11, 14], // V
	// 		[9, 12, 16], // vi
	// 		[11, 14, 17] // vii°
	// 	],
	// 	minor: [
	// 		[0, 3, 7], // i
	// 		[2, 5, 9], // ii°
	// 		[3, 7, 10], // III
	// 		[5, 9, 12], // iv
	// 		[7, 10, 14], // v/V
	// 		[8, 12, 15], // VI
	// 		[10, 14, 17] // VII
	// 	]
	// };

	const baseChordTemplates = {
		major: [
			[0, 4, 7],   // I
			[2, 5, 9],   // ii
			[4, 7, 11],  // iii
			[5, 9, 12],  // IV
			[7, 11, 14], // V
			[9, 12, 16], // vi
			[11, 14, 17] // vii°
		],
		minor: [
			[0, 3, 7],   // i
			[2, 5, 9],   // ii°
			[3, 7, 10],  // III
			[5, 9, 12],  // iv
			[7, 10, 14], // v
			[8, 12, 15], // VI
			[10, 14, 17] // VII
		]
	};

	function allInversions(triad) {
		const [r1, r2, r3] = triad;
		// Root position
		const rootPos = [r1, r2, r3];
		// 1st inversion (move bottom note +12)
		const firstInv = [r2, r3, r1 + 12];
		// 2nd inversion (move bottom two notes +12)
		const secondInv = [r3, r1 + 12, r2 + 12];

		return [rootPos, firstInv, secondInv];
	}

	export const chordTemplates = {
		major: baseChordTemplates.major.map(triad => allInversions(triad)).flat(),
		minor: baseChordTemplates.minor.map(triad => allInversions(triad)).flat()
	};

	let progression = [null, null, null, null];
	let locked = [false, false, false, false];

	// Piano roll range
	const pianoRollStart = 60; // C4
	const pianoRollEnd = 97; // C#7
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

	// Play/Stop logic + playhead animation
	let isPlaying = false;
	let playheadPosition = 0;  // 0-100
	let animationFrame = null;
	let chordTimeouts = [];

	function getTotalPlayTime() {
		return progression.length * 2; // 2s per chord
	}

	function togglePlay() {
		if (isPlaying) {
			stopPlayback();
		} else {
			startPlayback();
		}
	}

	async function startPlayback() {
		await Tone.start();
		isPlaying = true;
		playheadPosition = 0;

		synth.set({ oscillator: { type: selectedSound } });
		volumeNode.volume.value = volume;

		let startTime = performance.now();
		let totalTimeMs = getTotalPlayTime() * 1000;

		// schedule chords via setTimeout
		chordTimeouts = [];
		for (let i = 0; i < progression.length; i++) {
			if (progression[i] !== null) {
				const chordNotes = progression[i].map(m => Tone.Frequency(m, 'midi').toNote());
				let chordTime = i * 2000; // ms offset
				let tHandle = setTimeout(() => {
					synth.triggerAttackRelease(chordNotes, 1.8, Tone.now());
				}, chordTime);
				chordTimeouts.push(tHandle);
			}
		}

		// animate playhead
		function animatePlayhead() {
			if (!isPlaying) return;
			let elapsed = performance.now() - startTime;
			let ratio = elapsed / totalTimeMs;
			if (ratio >= 1) {
				playheadPosition = 100;  // end
				isPlaying = false;
				return;
			} else {
				playheadPosition = ratio * 100;
				animationFrame = requestAnimationFrame(animatePlayhead);
			}
		}
		animationFrame = requestAnimationFrame(animatePlayhead);
	}

	function stopPlayback() {
		for (let t of chordTimeouts) {
			clearTimeout(t);
		}
		chordTimeouts = [];
		if (animationFrame) {
			cancelAnimationFrame(animationFrame);
			animationFrame = null;
		}
		synth.releaseAll();

		isPlaying = false;
		playheadPosition = 0;
	}

	// Reactively update volume in real time
	// Whenever `volume` changes in Svelte, update the volumeNode
	$: volumeNode && (volumeNode.volume.value = volume);
	// Whenever `selectedSound` changes, update the synth oscillator type
	$: synth && synth.set({ oscillator: { type: selectedSound } });

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
	.controls, .chords {
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

	.volume-slider {
		margin-left: 1rem;
		width: 100px;
	}

	/* The main container: 2 columns. 
	   1) piano-key-column, 2) chord-columns. */
	.piano-roll-container {
		display: grid;
		grid-template-columns: auto 1fr; /* left auto, right takes the rest */
		border: 1px solid #ddd;
	}

	.piano-key-column {
		display: grid;
		grid-template-rows: repeat(var(--rows), 1em);
		border-right: 2px solid #aaa;
	}
	.piano-key {
		display: flex;
		align-items: center;
		justify-content: flex-end;
		padding-right: 0.3rem;
		border-bottom: 1px solid #eee;
		font-size: 0.8rem;
	}
	.black-key { background: #444; color: white; }
	.white-key { background: #fafafa; }

	.chord-columns {
		position: relative;
		display: grid;
		grid-template-columns: repeat(4, 1fr); /* 4 chords */
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
		background-color: rgb(173, 118, 255);
	}

	.playhead-line {
		position: absolute;
		top: 0;
		bottom: 0;
		width: 2px;
		background: rgb(93, 0, 127);
		left: 0;  /* animate left in a style attribute binding */
		pointer-events: none;
		z-index: 999;
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
	</div>

	<!-- Chord lock/unlock row -->
	<div class="controls">
		<button on:click={rollProgression}>🎲 Roll</button>
		<button on:click={togglePlay}>
			{#if isPlaying}
				■ Stop
			{:else}
				▶ Play
			{/if}
		</button>
	</div>
	<div class="chords">
		{#each progression as chord, i}
		<button
			class={locked[i] ? 'locked' : ''}
			on:click={() => locked[i] = !locked[i]}>
			{locked[i] ? '🔒' : '🔓'} 
			{chord ? chord.map(midiToNoteName).join('-') : 'N/A'}
		</button>
		{/each}
	</div>

	<!-- Piano roll visualization -->
	<div class="piano-roll-container" style="--rows: {totalNotes};">
		<!-- Left column: piano keys -->
		<div class="piano-key-column">
			{#each Array(totalNotes) as _, rowIndex}
				<div class="piano-key {isBlackKey(pianoRollEnd - rowIndex) ? 'black-key' : 'white-key'}">
					{midiToNoteName(pianoRollEnd - rowIndex)}
				</div>
			{/each}
		</div>

		<!-- Right columns: chord grid + playhead line -->
		<div class="chord-columns">
			<!-- The vertical red line, absolutely positioned, left: {playheadPosition}% -->
			<div
				class="playhead-line"
				style="left: {playheadPosition}%;"
			></div>

			{#each progression as chord, colIndex}
				<div class="piano-column">
					{#each Array(totalNotes) as _, rowIndex}
						<div
							class="piano-note {chord && chord.includes(pianoRollEnd - rowIndex) ? 'active' : ''}"
						/>
					{/each}
				</div>
			{/each}
		</div>
	</div>
</div>
