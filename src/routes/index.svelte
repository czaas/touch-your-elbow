<script lang="ts">
	import { onMount } from 'svelte';

	import '@tensorflow/tfjs-backend-webgl';
	import '@mediapipe/pose';
	import * as poseDetection from '@tensorflow-models/pose-detection';

	let browserSupported = true;
	let video: HTMLVideoElement;
	let detector: any;
	let isTouchingElbow = false;
	let leftElbowCoord = { x: null, y: null };
	let rightHandCoord = { x: null, y: null };
    
    let rightHandExists = false;
    let leftElbowExists = false;
	$: haveBoth = rightHandExists && leftElbowExists;

	onMount(() => {
		if (hasGetUserMedia()) {
			getVideoStream();
		} else {
			browserSupported = false;
		}
	});

	function hasGetUserMedia() {
		return !!(navigator.mediaDevices && navigator.mediaDevices.getUserMedia);
	}

	async function getVideoStream() {
		video = document.querySelector('#video');
		await navigator.mediaDevices.getUserMedia({ video: true }).then((stream) => {
			video.srcObject = stream;
		});
		setupTensorFlow();
	}
	async function setupTensorFlow() {
		// Create a detector.
		const model = poseDetection.SupportedModels.MoveNet;
		detector = await poseDetection.createDetector(model);

		getPoses();
	}

	async function getPoses() {
		const poses = await detector.estimatePoses(video);
		const { keypoints } = poses[0];
        
		const rightHand = keypoints.find((pose) => pose.name === 'right_wrist');
		if (rightHand.score > 0.1) {
			rightHandCoord = { x: rightHand.x, y: rightHand.y };
			rightHandExists = true;
		} else {
			rightHandCoord = { x: null, y: null };
			rightHandExists = false;
		}

		const leftElbow = keypoints.find((pose) => pose.name === 'left_elbow');
		if (leftElbow.score > 0.1) {
			leftElbowCoord = { x: leftElbow.x, y: leftElbow.y };
			leftElbowExists = true;
		} else {
			leftElbowCoord = { x: null, y: null };
            leftElbowExists = false;
		}
		requestAnimationFrame(getPoses);
	}

    $: xDiff = Math.abs(rightHandCoord.x - leftElbowCoord.x);
    $: yDiff = Math.abs(rightHandCoord.y - leftElbowCoord.y);
	$: prettyDamnClose = haveBoth && xDiff < 100 && yDiff < 100;
</script>

{#if browserSupported}
	<h1>Touch your left elbow</h1>

	<video autoplay id="video" />

    {#if rightHandExists && !leftElbowExists}
        <p>We can see your right hand but not your left elbow</p>
    {/if}
    {#if !rightHandExists && leftElbowExists}
        <p>We can see your left elbow but not your right hand</p>
    {/if}

	{#if prettyDamnClose}
		<p>Yay! You're touching your elbow (more or less)</p>
	{/if}
{:else}
	<h1>Your Browser is not supported</h1>
{/if}

<style lang="scss">
	:global(body) {
		text-align: center;
	}
	video {
		margin: 0 auto;
		background: #222;
		// turn video around because it's like a mirror
		transform: rotateY(180deg);
	}
</style>
