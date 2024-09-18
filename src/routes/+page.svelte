<script lang="ts">
  let audio: HTMLAudioElement;
  let playing = $state(false);
  let gainNode: GainNode | undefined = $state();

  const audioContext = $derived.by(() => {
    const audioContext = new AudioContext();

    gainNode = audioContext.createGain();
    const track = audioContext.createMediaElementSource(audio);

    track.connect(gainNode).connect(audioContext.destination);

    return audioContext;
  });
</script>

<audio bind:this={audio} controls loop src="/amen-break.mp3"></audio>

{#if gainNode}
  <input
    type="range"
    bind:value={gainNode.gain.value}
    min="0"
    max="4"
    step="0.01"
  />
{/if}

<button
  onclick={() => {
    if (audioContext.state === "suspended") {
      audioContext.resume();
    }
    if (playing) {
      audio.pause();
      playing = false;
    } else {
      audio.play();
      playing = true;
    }
  }}
  role="switch"
  aria-checked="false"
>
  <span>Play/Pause</span>
</button>
