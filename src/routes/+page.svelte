<script lang="ts">
  import "@fontsource-variable/open-sans";
  import "../app.css";

  import JSZip from "jszip";
  import { onMount } from "svelte";

  let status = $state(
    "Upload an audio file. to prepare it for the sampleslicer"
  );
  let filename = $state<string>("Upload");
  let download: { url: string; name: string; filename: string } | null =
    $state(null);
  let audioContext: AudioContext;
  // Map keyboard keys to the corresponding slice
  const keyMap = "qawsedrftgyhujik";
  let slices: AudioBuffer[] = $state([]);
  let current: number | null = $state(null);

  // Set up audio context on mount
  onMount(() => {
    audioContext = new window.AudioContext();
  });

  function reset() {
    status = "Upload an audio file to prepare it for the sampleslicer";
    filename = "Upload";
    download = null;
    slices = [];
    current = null;
  }

  // Handle file upload and processing
  async function handleFileUpload(event: Event) {
    const file = (event.target as HTMLInputElement).files?.[0];
    if (!file) {
      status = "Please upload a valid audio file.";
      return;
    }

    filename = file.name;
    status = "Processing...";
    download = null;
    slices = [];

    try {
      // Read the file as an ArrayBuffer
      const arrayBuffer = await file.arrayBuffer();
      const audioBuffer = await audioContext.decodeAudioData(arrayBuffer);

      // Split the audio into 16 equal parts
      slices = await splitAudio(audioBuffer);
      const zip = new JSZip();

      // Export each slice and generate download links
      slices.forEach((slice, index) => {
        const [leftChannel, rightChannel] = splitAudioIntoChannels(slice);

        const leftBlob = exportAsWav(leftChannel);
        zip.file(`${index + 1}-L.wav`, leftBlob);

        const rightBlob = exportAsWav(rightChannel ?? leftChannel);
        zip.file(`${index + 1}-R.wav`, rightBlob);
      });

      zip.generateAsync({ type: "blob" }).then(function (content) {
        // see FileSaver.js
        console.log(content);
        download = {
          name: `Download slices of ${filename}`,
          url: URL.createObjectURL(content),
        };
      });

      status = "";
    } catch (error) {
      status = "Error processing audio file.";
      console.error(error);
    }
  }

  // Split the audio buffer into 16 equal parts
  function splitAudio(audioBuffer: AudioBuffer) {
    const duration = audioBuffer.duration;
    const pieceDuration = duration / 16; // Duration of each slice in seconds
    const slices = [];

    let sliceDuration = Math.floor(pieceDuration * audioBuffer.sampleRate);

    console.log("Splitting audio into 16 slices", {
      channels: audioBuffer.numberOfChannels,
      duration,
      pieceDuration,
      sliceDuration,
      samplerate: audioBuffer.sampleRate,
    });

    for (let i = 0; i < 16; i++) {
      // Create a new AudioBuffer for each slice
      const slice = audioContext.createBuffer(
        audioBuffer.numberOfChannels,
        sliceDuration, // Number of samples for this slice
        audioBuffer.sampleRate
      );

      // Copy data from the original buffer to the slice
      for (let channel = 0; channel < audioBuffer.numberOfChannels; channel++) {
        const originalChannelData = audioBuffer.getChannelData(channel);
        const sliceChannelData = slice.getChannelData(channel);

        const startSample = i * sliceDuration;
        const endSample = (i + 1) * sliceDuration;

        const source = originalChannelData.subarray(startSample, endSample);
        sliceChannelData.set(source);
      }

      slices.push(slice);
    }

    return slices;
  }
  function splitAudioIntoChannels(audioBuffer: AudioBuffer) {
    const numChannels = audioBuffer.numberOfChannels;
    const sampleRate = audioBuffer.sampleRate;
    const length = audioBuffer.length;

    // Create new AudioBuffers for each channel
    const channels = [];
    for (let channel = 0; channel < numChannels; channel++) {
      const singleChannelBuffer = audioContext.createBuffer(
        1, // Number of channels
        length,
        sampleRate
      );
      singleChannelBuffer.copyToChannel(audioBuffer.getChannelData(channel), 0);
      channels.push(singleChannelBuffer);
    }

    return channels;
  }

  // Export an AudioBuffer to WAV
  function exportAsWav(audioBuffer: AudioBuffer) {
    const wavBuffer = audioBufferToWav(audioBuffer);
    const blob = new Blob([wavBuffer], { type: "audio/wav" });
    return blob;
  }

  // Function to convert AudioBuffer to WAV format
  function audioBufferToWav(buffer: AudioBuffer) {
    const numOfChan = buffer.numberOfChannels;
    const length = buffer.length * numOfChan * 2 + 44;
    const bufferArray = new ArrayBuffer(length);
    const view = new DataView(bufferArray);
    const channels = [];
    let sample;
    let offset = 0;
    let pos = 0;

    // Write WAV header
    setUint32(0x46464952); // "RIFF"
    setUint32(length - 8); // file length - 8
    setUint32(0x45564157); // "WAVE"
    setUint32(0x20746d66); // "fmt " chunk
    setUint32(16); // length = 16
    setUint16(1); // PCM (uncompressed)
    setUint16(numOfChan);
    setUint32(buffer.sampleRate);
    setUint32(buffer.sampleRate * 2 * numOfChan); // avg. bytes/sec
    setUint16(numOfChan * 2); // block-align
    setUint16(16); // 16-bit (hardcoded in this example)

    setUint32(0x61746164); // "data" - chunk
    setUint32(length - pos - 4); // chunk length

    for (let i = 0; i < numOfChan; i++) {
      channels.push(buffer.getChannelData(i));
    }

    while (pos < length) {
      for (let i = 0; i < numOfChan; i++) {
        sample = Math.max(-1, Math.min(1, channels[i][offset]));
        sample = (0.5 + sample * 32767) | 0;
        view.setInt16(pos, sample, true);
        pos += 2;
      }
      offset++;
    }

    function setUint16(data) {
      view.setUint16(pos, data, true);
      pos += 2;
    }

    function setUint32(data) {
      view.setUint32(pos, data, true);
      pos += 4;
    }

    return bufferArray;
  }

  let source = $state<AudioBufferSourceNode | null>(null);
  // Play the next audio slice in the queue
  function playNext(next: number) {
    if (source !== null) {
      source.stop();
      source.disconnect();
    }

    const slice = slices.at(next)!; // Get the next slice from the queue
    const newSource = audioContext.createBufferSource();
    newSource.buffer = slice;
    newSource.connect(audioContext.destination);

    // When the slice finishes, play the next one in the queue
    newSource.onended = (e) => {
      newSource?.disconnect();
      if (source === newSource || !source) {
        current = null;
      }
      source = null;
    };

    current = next;
    source = newSource;
    newSource.start();
  }

  // Handle keyboard input and queue the corresponding slice
  function handleKeydown(event: KeyboardEvent) {
    const key = event.key.toLowerCase();
    const index = keyMap.indexOf(key);
    if (index !== -1 && slices[index]) {
      playNext(index);
    }
  }
</script>

<svelte:document onkeydown={handleKeydown} />

<!-- Template for the UI -->
<main>
  <h1>SAMPLESLICER</h1>

  {#if download}
    <section class="stack">
      <a href={download.url} download="{download.filename}.zip"
        >{download.name}</a
      >

      <button onclick={reset}>Reset</button>
    </section>
  {:else}
    <!-- Audio file input -->
    <input type="file" accept="audio/*" onchange={handleFileUpload} />
  {/if}

  <!-- Status message -->
  <p>{status}</p>

  {#if slices.length > 0}
    <!-- Key mapping display -->
    <div>
      <h3>Press the following keys to play slices:</h3>
      <h4>Or use they keyboard ({keyMap})</h4>
    </div>

    <section class="display">
      {#each keyMap as key, index}
        <button
          class:current={current === index}
          onclick={() => playNext(index)}
        >
          {index + 1}
        </button>
      {/each}
    </section>
  {/if}
</main>
