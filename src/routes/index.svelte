<script lang="ts">
	import { onMount } from 'svelte';

	import '@tensorflow/tfjs-backend-webgl';
	import '@mediapipe/pose';
	import * as poseDetection from '@tensorflow-models/pose-detection';

	let browserSupported = true;
	let video: HTMLVideoElement;
	let detector: any;
	let ready = false;

    interface Keypoint {
		name: string;
		score: number;
		x: number;
		y: number;
	}
	interface BodyPartStats {
		partOneVisible: boolean;
		partTwoVisible: boolean;
		isTouching: boolean;
	}

	let rightHandTouchingElbow: BodyPartStats = {
		partOneVisible: false,
		partTwoVisible: false,
		isTouching: false
	};
    let leftHandTouchingRightElbow: BodyPartStats = {
		partOneVisible: false,
		partTwoVisible: false,
		isTouching: false
	};

	onMount(() => {
		if (canUseCamera()) {
			getVideoStream();
		} else {
			browserSupported = false;
		}
	});

	function canUseCamera() {
		const getUserMediaCheck = !!(navigator.mediaDevices && navigator.mediaDevices.getUserMedia);
		return getUserMediaCheck;
	}
	async function getVideoStream() {
		video = document.querySelector('#video');
		await navigator.mediaDevices.getUserMedia({ video: { aspectRatio: 1920/1080 } }).then((stream) => {
			video.srcObject = stream;
			video.onloadeddata = () => {
				setupTensorFlow();
			};
		});
	}
	async function setupTensorFlow() {
		// Create a detector.
		const model = poseDetection.SupportedModels.MoveNet;
		detector = await poseDetection.createDetector(model);
		ready = true;
		getPoses();
	}

	async function getPoses() {
		const poses = await detector.estimatePoses(video);
		const { keypoints } = poses[0];

		rightHandTouchingElbow = isBodyPartCloseToEachOther(keypoints, 'right_wrist', 'left_elbow');
        leftHandTouchingRightElbow = isBodyPartCloseToEachOther(keypoints, 'left_wrist', 'right_elbow');
		requestAnimationFrame(getPoses);
	}

	function isBodyPartCloseToEachOther(
		keypoints: Keypoint[],
		partOne: string,
		partTwo: string
	): BodyPartStats {
		const partOneKeypoint = keypoints.find((pose) => pose.name === partOne);
		const partTwoKeypoint = keypoints.find((pose) => pose.name === partTwo);

		const xDiff = Math.abs(partOneKeypoint.x - partTwoKeypoint.x);
		const yDiff = Math.abs(partOneKeypoint.y - partTwoKeypoint.y);
        
		return {
			partOneVisible: partOneKeypoint.score > 0.1,
			partTwoVisible: partTwoKeypoint.score > 0.1,
			isTouching: xDiff < 100 && yDiff < 100
		};
	}
    
</script>

{#if browserSupported}
    <main>
        <h1>Touch your left elbow</h1>
        <p>
            <a href="https://github.com/czaas/touch-your-elbow" target="_blank">Source code on github</a>
        </p>

        <video autoplay id="video" />

        {#if ready}
            {#if leftHandTouchingRightElbow.isTouching && !rightHandTouchingElbow.isTouching}
                <p>Wrong elbow!</p>
            {:else}
                {#if !rightHandTouchingElbow.partOneVisible && !rightHandTouchingElbow.partTwoVisible}
                    <p>We cannot see your left elbow or your right hand</p>
                {:else if rightHandTouchingElbow.partOneVisible && !rightHandTouchingElbow.partTwoVisible}
                    <p>We can see your right hand but not your left elbow</p>
                {:else if !rightHandTouchingElbow.partOneVisible && rightHandTouchingElbow.partTwoVisible}
                    <p>We can see your left elbow but not your right hand</p>
                {:else if !rightHandTouchingElbow.isTouching}
                    <p>Now touch</p>
                {:else}
					{#if leftHandTouchingRightElbow.isTouching && rightHandTouchingElbow.isTouching}
						<p>You're touching your both elbows, but that's okay!</p>
					{:else}
                    	<p>Yay! You're touching your left elbow</p>
					{/if}
                {/if}
            {/if}
        {:else}
            <p>Loading...</p>
        {/if}
    </main>
{:else}
	<h1>Your Browser is not supported</h1>
{/if}

<style lang="scss">
	:global(body) {
		text-align: center;
		font-size: 20px;
	}
	video {
		// turn video around because it's like a mirror
		transform: rotateY(180deg);
        width: 760px;
        max-height: 80vh;
        max-width: 100%;
	}
</style>
