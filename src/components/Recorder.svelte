<script lang="ts">
  import { onMount } from "svelte";

  let record: HTMLButtonElement;
  let stop: HTMLButtonElement;

  let handleRequestAudioClick = () => {};

  onMount(() => {
    const requestAudio = () => {
      // Main block for doing the audio recording

      if (navigator.mediaDevices.getUserMedia) {
        console.log("getUserMedia supported.");

        const constraints = { audio: true };
        let chunks = [];

        let onSuccess = function (stream) {
          const mediaRecorder = new MediaRecorder(stream);

          visualize(stream);

          record.onclick = function () {
            mediaRecorder.start();
            console.log(mediaRecorder.state);
            console.log("recorder started");
            record.style.background = "red";

            stop.disabled = false;
            record.disabled = true;
          };

          stop.onclick = function () {
            mediaRecorder.stop();
            console.log(mediaRecorder.state);
            console.log("recorder stopped");
            record.style.background = "";
            record.style.color = "";
            // mediaRecorder.requestData();

            stop.disabled = true;
            record.disabled = false;
          };

          mediaRecorder.onstop = function (e) {
            console.log("data available after MediaRecorder.stop() called.");

            // const clipName = prompt(
            //   "Enter a name for your sound clip?",
            //   "My unnamed clip"
            // );

            const clipName = "new-clip"

            const clipContainer = document.createElement("article");
            const clipLabel = document.createElement("p");
            const audio = document.createElement("audio");
            const deleteButton = document.createElement("button");

            clipContainer.classList.add("clip");
            audio.setAttribute("controls", "");
            deleteButton.textContent = "Delete";
            deleteButton.className = "delete";

            if (clipName === null) {
              clipLabel.textContent = "My unnamed clip";
            } else {
              clipLabel.textContent = clipName;
            }

            clipContainer.appendChild(audio);
            // clipContainer.appendChild(clipLabel);
            // clipContainer.appendChild(deleteButton);
            soundClips.appendChild(clipContainer);

            audio.controls = true;
            const blob = new Blob(chunks, { type: "audio/webm; codecs=opus" });
            chunks = [];
            const audioURL = window.URL.createObjectURL(blob);
            audio.src = audioURL;
            console.log("recorder stopped");

            // Send blob to server
            const formData = new FormData();

            formData.append("file", blob, "audio.webm");

            fetch("https://confessions.phocks.org/upload", {
              //  fetch("http://localhost:5000/upload", {
              method: "POST",
              body: formData,
              headers: {
                // NOTE: Browsers will automatically set the Content-Type header for you
                // So this is not needed
                // "Content-Type": "multipart/form-data",
              },
            })
              .then((res) => {
                console.log(res);
                // button.innerHTML = "Audio sent...";
              })
              .catch((err) => {
                // button.innerHTML = "Error :(";
                console.error("Error occured", err);
              });

            deleteButton.onclick = function (e: any) {
              e.target.closest(".clip").remove();
            };

            clipLabel.onclick = function () {
              const existingName = clipLabel.textContent;
              const newClipName = prompt(
                "Enter a new name for your sound clip?"
              );
              if (newClipName === null) {
                clipLabel.textContent = existingName;
              } else {
                clipLabel.textContent = newClipName;
              }
            };
          };

          mediaRecorder.ondataavailable = function (e) {
            chunks.push(e.data);
          };
        };

        let onError = function (err) {
          console.log("The following error occured: " + err);
        };

        navigator.mediaDevices
          .getUserMedia(constraints)
          .then(onSuccess, onError);
      } else {
        console.log("getUserMedia not supported on your browser!");
      }
    };

    handleRequestAudioClick = requestAudio;

    // set up basic variables for app

    // const record = recordButtonEl;
    // const stop: any = document.querySelector(".stop");
    const soundClips: any = document.querySelector(".sound-clips");
    const canvas: any = document.querySelector(".visualizer");
    const mainSection: any = document.querySelector(".main-controls");

    // disable stop button while not recording

    stop.disabled = true;

    // visualiser setup - create web audio api context and canvas

    let audioCtx;
    const canvasCtx = canvas.getContext("2d");

    // Request audio
    // requestAudio();

    function visualize(stream) {
      if (!audioCtx) {
        audioCtx = new AudioContext();
      }

      const source = audioCtx.createMediaStreamSource(stream);

      const analyser = audioCtx.createAnalyser();
      analyser.fftSize = 2048;
      const bufferLength = analyser.frequencyBinCount;
      const dataArray = new Uint8Array(bufferLength);

      source.connect(analyser);

      draw();

      function draw() {
        const WIDTH = canvas.width;
        const HEIGHT = canvas.height;

        requestAnimationFrame(draw);

        analyser.getByteTimeDomainData(dataArray);

        canvasCtx.fillStyle = "rgb(200, 200, 200)";
        canvasCtx.fillRect(0, 0, WIDTH, HEIGHT);

        canvasCtx.lineWidth = 2;
        canvasCtx.strokeStyle = "rgb(0, 0, 0)";

        canvasCtx.beginPath();

        let sliceWidth = (WIDTH * 1.0) / bufferLength;
        let x = 0;

        for (let i = 0; i < bufferLength; i++) {
          let v = dataArray[i] / 128.0;
          let y = (v * HEIGHT) / 2;

          if (i === 0) {
            canvasCtx.moveTo(x, y);
          } else {
            canvasCtx.lineTo(x, y);
          }

          x += sliceWidth;
        }

        canvasCtx.lineTo(canvas.width, canvas.height / 2);
        canvasCtx.stroke();
      }
    }

    window.onresize = function () {
      canvas.width = mainSection.offsetWidth;
    };

    window.onresize(null);
  });
</script>

<div class="wrapper">
  <div class="main">
    <section class="main-controls">
      <canvas class="visualizer" height="60px" />
      <div id="buttons">
        <button on:click={handleRequestAudioClick}>
          Request recording permissions
        </button>
        <button bind:this={record} class="record">Record</button>
        <button bind:this={stop} class="stop">Stop</button>
      </div>
    </section>

    <section class="sound-clips" />
  </div>
</div>

<style lang="scss">
  .wrapper {
    // display: flex;
    // flex-direction: column;
    // align-items: center;
    // justify-content: center;
    // height: 100dvh;
  }

  canvas {
    width: 100%;
    height: 60px;
    background-color: #333;
  }

  .main {
    width: 100%;
    margin: 0 auto;
  }

  .main-controls {
    width: 100%;
    margin-block-end: 1em;
  }

  .request-audio {
    padding: 1rem;
  }

  .visualizer {
    vertical-align: middle;
  }

  #buttons {
    margin-block-start: 1em;
  }

  button {
    display: inline-block;
    margin-block-end: 10px;
    min-width: 100px;
    background-color: white;
    border: 1px solid black;
    padding: 0.5rem 1rem;
  }
</style>
