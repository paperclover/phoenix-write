<script lang="ts">
	import {
		playAudio,
		startMusic,
		startMusicInstant,
		stopMusic,
		stopMusicInstant
	} from '$lib/audio';
	import { isFocused } from '$lib/isFocused';
	import { setScreenshake, setScreenshake2 } from '$lib/screenshake';

	const PRESS_MARGIN_END = 0.18;
	const PRESS_MARGIN_START = 0.18;

	import { currentMapId, setNextMap } from '$lib/stores';

	import type { LoadedCutscene } from '$lib/types';
	import { parseTimmyTimestamp } from '$lib/types';
	import { onDestroy } from 'svelte';
import { now, time_ranges_to_array } from 'svelte/internal';
	import { fade, fly } from 'svelte/transition';
	import Stats from './STATS.svelte';

	export let cutscene: LoadedCutscene;


	let lastTimeInFrames = 0;
	let timeBetweenLoopsInFrames = 1;
	let cutsceneStart = performance.now()/1000;
	let lost = false;
	

	let didAutoPlayOnce = false;

	// Number of times player has died to heart.
	let heartLosses = 0;
	let threshold = 2;
	let timeAtWhichHeartReachedThreshold = 0;
	let currentGameTime = 0;
	let delayInMillisecondsUntilHelpMessageAppears = 2*1000;
	let finallyGotIt = false;

	const boysString = 'bbbbbbooooooyyyyyysssssss';
	const sounds = [
		'b0',
		'b0',
		'b0',
		'b1',
		'b1',
		'b1',
		'b1',
		'b1',
		'b2',
		'b2',
		'b2',
		'b2',
		'b3',
		'b3',
		'b3',
		'b3',
		'b4',
		'b4',
		'b5',
		'b5',
		'b6',
		'b6',
		'b7',
		'b7',
		'b8',
		'boyscorrect'
	];

	let boysIndex = 0;
	let boysWon = false;

	let slap = false;

	function dependOn(...args: unknown[]) {}
	function unassociate<T>(arg: T) {
		return arg;
	}

	let STATSEEE = false;

	let videoElem: HTMLVideoElement;

	let currentSectionI = 0;
	$: currentSection = cutscene.subsection[currentSectionI];
	$: pauseTime = cutscene.subsection[currentSectionI]
		? cutscene.subsection[currentSectionI].end
			? parseTimmyTimestamp(cutscene.subsection[currentSectionI].end)
			: currentSectionI === cutscene.subsection.length - 1
			? videoElem?.duration
			: parseTimmyTimestamp(cutscene.subsection[currentSectionI + 1].begin) + 1 / 120
		: 0;

	$: {
		if (videoElem && currentSection) {
			videoElem.pause();
			videoElem.currentTime =
				parseTimmyTimestamp(currentSection.begin) + 1 / 120 + 1 / cutscene.fps;
			videoElem.play();

			if (currentSectionI === 0) {
				if (cutscene.subsection[0].startMusic) {
					// This line erroneously makes music start on bussinB.
					// For now I just moved the music to start on the second caption, which works fine story-wise anyway.
					startMusicInstant(cutscene.subsection[0].startMusic);
				} else if (cutscene.subsection[0].startMusicFade) {
					startMusic(cutscene.subsection[0].startMusicFade);
				} else if (cutscene.bgm) {
					startMusic(cutscene.bgm);
				}
			}
		}
		if (!currentSection) {
			if ($currentMapId === '06-endingscene') {
				STATSEEE = true;
			} else {
				setNextMap(cutscene);
			}
		}
	}




	if ($currentMapId === '06-endingscene') {
	}
	$: done = (dependOn(videoElem, currentSectionI), false);
	$: hasPressedSpace = (dependOn(videoElem, currentSectionI), false);
	$: keyIndex = (dependOn(videoElem, currentSectionI), 0);

	function play() {
		let running = true;
		function stopHandler() {
			if ($currentMapId === '06-endingscene') {
				STATSEEE = true;
			}

			running = false;
			videoElem.removeEventListener('pause', stopHandler);

			if (currentSection.theBoys && !boysWon) {
				lose(videoElem.currentTime);
				done = false;
				return;
			}
		}

		videoElem.addEventListener('pause', stopHandler);
		videoElem.play();

		let hasFadedOutMusic = false;

		requestAnimationFrame(function loop() {
			if (lost) return;
			if (!running) return;
			requestAnimationFrame(loop);


				// Add the bugfix for ending cutscene
			if(currentSection?.motherfuckingbugfix) 
			{ 
				localStorage.setItem(
				'phoenixwrite_savefile',
				currentSection?.savefile
				);
				console.log("Save state set to" + currentSection?.savefile);
			}
			
			if (
				currentSection.shake &&
				!didshake &&
				videoElem.currentTime > parseTimmyTimestamp(currentSection.shake)
			) {
				didshake = true;
				setScreenshake2();
			}
			if (
				currentSection.playqtslap &&
				!slap &&
				videoElem.currentTime > parseTimmyTimestamp(currentSection.playqtslap)
			) {
				slap = true;
				setScreenshake();
				playAudio('seriousBusinessSlapNoise');
			}

			if (
				currentSection.keys &&
				currentSection.keys.length &&
				keyIndex < currentSection.keys.length
				
			) {
				if(!currentSection.heartFlag)
				{
					if (
					videoElem.currentTime >=
					parseTimmyTimestamp(currentSection.keys[keyIndex].time) + PRESS_MARGIN_END
					) 
					{
						lose(parseTimmyTimestamp(currentSection.keys[keyIndex].time));
					}

				}
				// Basically, heart flag is so close to the pause time that adding PRESS_MARGIN_END makes it wait for a time that never occurs.
				else
				{
					currentGameTime = performance.now();
					// Keep iterating timeAtWhichHeartReachedThreshold until it reaches it...
					if(heartLosses < threshold)
					{
						timeAtWhichHeartReachedThreshold = performance.now();
					}

					if (
					videoElem.currentTime >=
					parseTimmyTimestamp(currentSection.keys[keyIndex].time)
					) 
					{
						lose(parseTimmyTimestamp(currentSection.keys[keyIndex].time));
					}
				}
			}

			if (
				currentSectionI === cutscene.subsection.length - 1 &&
				videoElem.currentTime >= pauseTime - 1 &&
				!hasFadedOutMusic
			) {
				stopMusic();
				hasFadedOutMusic = true;
			}
		

			//TIMMY TIMESTAMP RETURNS TO SAVE THE DAY
			let pauseSecond = Math.floor(pauseTime);
			let pauseFrame = Math.floor((pauseTime - pauseSecond)*cutscene.fps);

			let pauseTimeInFrames = pauseSecond*cutscene.fps + pauseFrame;

			let currentSecond = Math.floor(videoElem.currentTime);
			let currentFrame = (videoElem.currentTime - currentSecond)*cutscene.fps;

			let currentTimeInFrames = currentSecond*cutscene.fps + currentFrame;

			// Low pass filter this because it acts so fucking weirdly.
			timeBetweenLoopsInFrames = timeBetweenLoopsInFrames*.9 +  (currentTimeInFrames - lastTimeInFrames)*.1;

			lastTimeInFrames = currentTimeInFrames;
			// Look, let's assume the NEXT loop is going to take JUST AS LONG as the previous loop took.
			// If you extrapolate ahead, WILL YOU END UP AT OR BEYOND THE PAUSE FRAME?
			// If so, PAUSE THE FUCKER.
			// IF not, Don't pause the fucker.

			let projectedNextLoopTimeInFrames = currentTimeInFrames + timeBetweenLoopsInFrames;
			//console.log(timeBetweenLoopsInFrames);
			// As long as the video isn't about to end, we can use this smart method of pausing.
			if(videoElem.currentTime < videoElem.duration-1)
			{
				// Okay. Now this is very, very easy.
				if(projectedNextLoopTimeInFrames >= pauseTimeInFrames)
				{
					running = doPauseShit();
				}

				// Please, please, PLEASE work.

				// Holy shit, the low pass filtering is what fixed it. Fuck yes.
			}
			else
			{
				// Weird-ass pausing method reserved for when the cutscene's about to end.
				// When you don't have this, next video never loads.
				let offs = 1.6/cutscene.fps;
				if (videoElem.currentTime >= pauseTime - offs) 
				{
					running = doPauseShit();
				}
			}

		});
	}

	function doPauseShit()
	{
		let running = false;
		done = true;
		videoElem.pause();
		unassociate(videoElem).currentTime = pauseTime;
		if (currentSection.autoplay) {
			if (
				currentSection[
					'hardcoded flag Autoplay Only once. but this flag only is supported on this subsection and it wont work properly on any other sections'
				]
			) {
				if (didAutoPlayOnce) {
					return;
				}
				didAutoPlayOnce = true;
			}
			currentSectionI++;
		}
		return running;
	}

	const video = URL.createObjectURL(cutscene.video);

	onDestroy(() => {
		URL.revokeObjectURL(video);
	});

	let heartBuffer = '';
	let heartWon = false;
	let didshake = false;

	const validHeartPhrases = [
		'heart',
		'sparkling_heart',
		'sparklingheart',
		'<3',
		',3',
		'<#',
		',#',
		'💖',
		'❤️',
		'💗',
		'💓',
		'💝',
		'💞',
		'wtf',
		'killyourself',
		'love',
		'wuv'
	];

	function onKeyDown(event: KeyboardEvent) {
		if (event.ctrlKey || event.altKey || event.metaKey) return;
		if (lost) return;

		if (event.key === ' ' && !hasPressedSpace) {
			if (done) {
				hasPressedSpace = true;
				playAudio(currentSection.overrideSFX ?? 'dialogue_advance_default');

				if (!currentSection.flagNextSectionDoesntClear) {
					unassociate(videoElem).currentTime = videoElem.currentTime + 1 / cutscene.fps;
				}

				let nextSection = cutscene.subsection[currentSectionI + 1];
				if (nextSection) {
					let startMusicFade = nextSection.startMusicFade;
					if (startMusicFade) {
						startMusic(startMusicFade);
					} else if (nextSection.startMusic) {
						startMusicInstant(nextSection.startMusic);
					} else if (nextSection.endMusicFade) {
						stopMusic();
					} else if (nextSection.endMusic) {
						stopMusicInstant();
					}
				}

				setTimeout(() => {
					currentSectionI++;
				}, 170);
			} else if (!currentSection.unskippable) {
				videoElem.pause();
				unassociate(videoElem).currentTime = pauseTime;
				done = true;
			}
		}
		if (event.key === ' ') return;

		if (currentSection.heartFlag && event.key.length === 1) {
			heartBuffer += event.key.toLowerCase().trim();

			const found = validHeartPhrases.find((phrase) => heartBuffer.includes(phrase));
			if (found) {
				heartWon = true;
			}
		}

		if (
			currentSection.keys &&
			currentSection.keys.length &&
			keyIndex < currentSection.keys.length &&
			event.key.length === 1 &&
			videoElem.currentTime >= 4
		) {
			if (
				currentSection.keys[keyIndex].key === '💖'
					? true
					: currentSection.keys[keyIndex].key === '%'
					? // hardcoded
					  ['%', '5'].includes(event.key)
					: // regular
					  event.key === currentSection.keys[keyIndex].key
			) {
				// check timing information
				let startAllowed =
					parseTimmyTimestamp(currentSection.keys[keyIndex].time) - PRESS_MARGIN_START;
				let endAllowed =
					parseTimmyTimestamp(currentSection.keys[keyIndex].time) + PRESS_MARGIN_START;

				// We need to push startAllowed earlier when the fuckin' heart is there.
				if(currentSection.heartFlag)
				startAllowed = 
				parseTimmyTimestamp(currentSection.keys[keyIndex].time) - PRESS_MARGIN_START*2;

				let currentTime = videoElem.currentTime;

				if (currentTime >= startAllowed && currentTime <= endAllowed) {
					keyIndex++;
					playAudio('correctpluck');
					// If heart, you gotta say they got it and un-show the help text
					if(currentSection.heartFlag)
						finallyGotIt = true;

				} else {
					lose();
				}
			} else {
				lose();
			}
		}

		if (currentSection.theBoys && !boysWon) {
			let requiredKey = boysString.charAt(boysIndex);
			if (event.key.toLowerCase() === requiredKey) {
				boysIndex++;
				playAudio(sounds[boysIndex]);
				if (boysIndex === boysString.length) {
					boysWon = true;
				}
			}
		}
	}

	$: {
		if (!$isFocused && videoElem) {
			videoElem.pause();
			function handleUnpause() {
				if (document.visibilityState === 'visible') {
					videoElem.play();
					document.removeEventListener('visibilitychange', handleUnpause);
				}
			}

			document.addEventListener('visibilitychange', handleUnpause);
		}
	}

	let isFadeToWhite = false;

	export function lose(time?: number) {


		if(currentSection.heartFlag)
		{
			heartLosses++;

			if(heartLosses > threshold)
			{

				currentGameTime = performance.now();
				timeAtWhichHeartReachedThreshold = performance.now();
			}
		}

		setScreenshake();

		lost = true;
		videoElem.pause();

		unassociate(videoElem).currentTime =
			time ?? parseTimmyTimestamp(currentSection.keys[keyIndex].time) ?? videoElem.currentTime;

		if (currentSection.isBussinB) {
			stopMusicInstant();
			playAudio('doomedfarewell');
			stopMusicInstant();
			setTimeout(() => {
				isFadeToWhite = true;
			}, 500);
			setTimeout(() => {
				currentMapId.set('01-reddit-recap');
			}, 1500);
		} else {
			playAudio('screwup');
			setTimeout(() => {
				isFadeToWhite = true;
			}, 500);
			setTimeout(() => {
				isFadeToWhite = false;
				slap = false;
				boysIndex = 0;
				boysWon = false;
				heartBuffer = '';
				heartWon = false;
				keyIndex = 0;
				lost = false;
				done = false;
				currentSectionI = Math.max(0, currentSectionI - 1);
				currentSection = cutscene.subsection[currentSectionI];
				// You gotta reset the times and shit on lose, or else they get negative and shit could get real weird.
				lastTimeInFrames = currentSection.begin[0]*24 + currentSection.begin[1];

			}, 1500);
		}
	}


	
	if ($currentMapId === '06-endingscene') {
		window.a1 = new Image();
		window.a1.src = `./scoreboard/bad.png`;
		window.a2 = new Image();
		window.a2.src = `./scoreboard/better.png`;
		window.a3 = new Image();
		window.a3.src = `./scoreboard/best.png`;
		window.a4 = new Image();
		window.a4.src = `./scoreboard/statsbg.png`;
		window.a5 = new Image();
		window.a5.src = `./scoreboard/statseval.png`;
	}
</script>

<svelte:window on:keydown={onKeyDown} />

<main>
	{#if !STATSEEE}
		<video src={video} bind:this={videoElem} on:play={play} disablePictureInPicture />
		{#if done && !hasPressedSpace}
			<div class="bottom" in:fly={{ duration: 200, opacity: 0, y: 2 }} out:fade={{ duration: 100 }}>
				{@html currentSection.continueText ?? '(Press space to continue)'}

			</div>
		{/if}

		{#if isFadeToWhite}
			<div class="fade-to-white" transition:fade={{ duration: 500 }} />
		{/if}

		{#if currentSection && currentSection.heartFlag && heartLosses > 1 && currentGameTime > timeAtWhichHeartReachedThreshold + delayInMillisecondsUntilHelpMessageAppears && !finallyGotIt}
			<div class="bottom" in:fly={{ duration: 500, opacity: 0, y: 2 }} out:fade={{ duration: 100 }}>
				{#if heartLosses == 2}
					{@html 'Bro, just press any key at the right time.'}
				{/if}
				{#if heartLosses == 3}
					{@html '...Okay, press immediately BEFORE the right time.'}
				{/if}
				{#if heartLosses == 4}
					{@html 'C\'mon, you got this! You\'ve come so far.'}
				{/if}
				{#if heartLosses == 5}
					{@html 'This project took over 100 hours of work.'}
				{/if}
				{#if heartLosses == 6}
					{@html 'Sure, there were frustrating moments.'}
				{/if}
				{#if heartLosses == 7}
					{@html 'But I persevered! Just like I know you will.'}
				{/if}
				{#if heartLosses > 7}
					{@html 'Never give up.'}
				{/if}
			</div>
		{/if}

		{#if currentSection && currentSection.theBoys}
			<div class="THEBOYS">
				{#each boysString as char, i}
					<span class:hit={boysIndex > i}>{char.toUpperCase()}</span>
				{/each}
				
			</div>
		{/if}

		{#if currentSection?.keys}
			{#each currentSection.keys as key, i}
				<div
					class:ki_correct={keyIndex > i}
					class:ki_fail={lost && keyIndex === i}
					class="keyinstance"
					style="--kx:{key.pos[0]};--ky:{key.pos[1]}"
				>
					<div class="tx">
						{key.key}
					</div>
				</div>
			{/each}
		{/if}
	{:else}
		<Stats />
	{/if}
</main>

<div class="click-overlay" />

<style>
	main {
		position: absolute;
		top: 0;
		left: 0;
		width: 100%;
		height: 100%;
		pointer-events: none;
	}
	video {
		position: absolute;
		top: 0;
		left: 0;
		width: 100%;
		height: 100%;
	}
	.click-overlay {
		position: absolute;
		top: 0;
		left: 0;
		width: 100%;
		height: 100%;
		cursor: none;
		z-index: 100;
	}
	.bottom {
		position: absolute;
		bottom: calc(var(--unit) * 1);
		left: 0;
		width: 100%;
		text-align: center;
		font-size: calc(var(--unit) * 2);
		color: white;
		opacity: 0.8;
		cursor: none;
	}
	.fade-to-white {
		position: absolute;
		top: 0;
		left: 0;
		width: 100%;
		height: 100%;
		background: white;
	}

	.THEBOYS {
		position: absolute;
		top: calc(var(--unit) * 43);
		left: calc(var(--unit) * 10.75);
		font-size: calc(var(--unit) * 3.5);
		letter-spacing: calc(var(--unit) * 0.23);
		color: rgba(255, 255, 255, 0.5);
		animation: appearboys 1s linear both;

	}
	.THEBOYS span {
		display: inline-block;
	}
	.hit {
		color: white;
		animation: BOUNCER 0.3s ease-out both;
	}

	@keyframes appearboys {
		0% {
			opacity: 0;
		}
		100% {
			opacity: 1;
		}
	}

	@keyframes BOUNCER {
		0% {
			transform: translateY(0);
		}
		50% {
			transform: translateY(-10px);
		}
		100% {
			transform: translateY(0);
		}
	}

	.keyinstance {
		position: absolute;
		left: calc(var(--unit) * (var(--kx) / 19.2 - 1.5));
		top: calc(var(--unit) * ((9 / 16) * (var(--ky) / 10.8) - 1.65));
		width: calc(var(--unit) * 2.75);
		height: calc(var(--unit) * 2.75);
		border: calc(var(--unit) * 0.3) solid red;
		color: red;
		border-radius: 100%;
		display: flex;
		justify-content: center;
		align-items: center;
		font-size: calc(var(--unit) * 3.5);
		opacity: 0;
	}
	.tx {
		position: absolute;
		top: 0;
		left: 0;
		width: 100%;
		height: 100%;
		display: flex;
		justify-content: center;
		align-items: center;
		transform: translateY(calc(var(--unit) * -0.4));
	}

	.ki_correct {
		color: transparent;
		border-color: #0f0;
		animation: correctscale 0.3s ease-out both;
	}
	@keyframes correctscale {
		0% {
			transform: scale(1);
			opacity: 1;
		}
		100% {
			transform: scale(3);
			opacity: 0;
		}
	}

	.ki_fail {
		color: #f00;
		border-color: #f00;
		background-color: rgba(0, 0, 0, 0.5);
		animation: wrongscale 2s cubic-bezier(0.19, 1, 0.22, 1) both;
		opacity: 1;
	}

	@keyframes wrongscale {
		0% {
			transform: scale(1);
			opacity: 1;
		}
		50% {
			opacity: 1;
			transform: scale(2.5);
		}
		100% {
			opacity: 0;
			transform: scale(1);
		}
	}
</style>
