<script lang="ts">
	import ChangingLogo from '$lib/ChangingLogo.svelte';
	import { onMount } from 'svelte';

	let changingLogo: ChangingLogo;
	let randomNum = Math.floor(Math.random() * 360);

	onMount(() => window.requestAnimationFrame(animate));

	let prevTimeStampLogo: number, prevTimeStampColor: number;

		function animate(timeStamp: number) {
		if (prevTimeStampColor === undefined) {
			prevTimeStampColor = timeStamp;
		}
		if (timeStamp >= prevTimeStampColor + 1200) {
			randomNum = Math.floor(Math.random() * 360);
			prevTimeStampColor = timeStamp
		} 
		if (prevTimeStampLogo === undefined) {
			prevTimeStampLogo = timeStamp;
		}
		if (timeStamp >= prevTimeStampLogo + 500) {
			changingLogo.getRandomLogo();
			prevTimeStampLogo = timeStamp
		}
		window.requestAnimationFrame(animate);
	}
	
</script>


<div class="" style:background-color="white">
	<div class="sticky h-[100vh] top-0 flex justify-center p-[10%]">
    <img src="/fuzue-logo.png">
		<div class="absolute bottom-10 text-slate-950 font-sans text center flex flex-col items-center">
			<p class="text-3xl">contact@fuzue.tech</p>
		</div>
	</div>
</div>

