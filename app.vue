<template lang="pug">
div
  <!-- You need to bring your own Tone.js for the player, and tfjs for the model -->
  Script(src="https://cdnjs.cloudflare.com/ajax/libs/tone/14.7.58/Tone.js") 
  Script(
    src="https://cdnjs.cloudflare.com/ajax/libs/tensorflow/1.2.8/tf.min.js"
  ) 
  <!-- Core library, since we're going to use a player -->
  Script(src="https://cdn.jsdelivr.net/npm/@magenta/music@^1.0.0/es6/core.js") 
  <!--Model we want to use -->
  Script(
    src="https://cdn.jsdelivr.net/npm/@magenta/music@^1.0.0/es6/music_vae.js"
  )
  Script(
    src="https://cdn.jsdelivr.net/npm/@magenta/music@^1.0.0/es6/music_rnn.js"
  ) 
  button(v-if="!running", @click="start") START
  button(v-else, @click="stop") STOP
</template>

<script setup>
const running = ref(false);
const stopped = ref(true);

let player;
let musicVae;
let musicRnn;
const playLoop = async (notes) => {
  running.value = true;
  notes = await notes;
  for (var i = 0; i < notes.notes.length; i++) {
    while (notes.notes[i].pitch < 48) notes.notes[i].pitch += 12;
    while (notes.notes[i].pitch > 83) notes.notes[i].pitch -= 12;
  }
  player.start(notes);

  const unwatch = watch(running, () => {
    const nextNotes = musicRnn.continueSequence(notes, 64, 1.5);
    if (!running.value && !stopped.value) {
      unwatch();
      playLoop(nextNotes);
    }
  });
};
const start = async () => {
  const Player = core.Player;
  const MusicVAE = music_vae.MusicVAE;
  const MusicRNN = music_rnn.MusicRNN;

  stopped.value = false;
  if (!player) {
    player = new Player(false, {
      run: (note) => {},
      stop: () => {
        running.value = false;
      },
    });
  }
  if (!musicVae) {
    musicVae = new MusicVAE(
      "https://storage.googleapis.com/magentadata/js/checkpoints/music_vae/mel_4bar_small_q2"
    );
    await musicVae.initialize();
  }
  if (!musicRnn) {
    musicRnn = new MusicRNN(
      "https://storage.googleapis.com/magentadata/js/checkpoints/music_rnn/basic_rnn"
    );
    await musicRnn.initialize();
  }
  if (player.isPlaying()) {
    player.stop();
  }
  console.log("initialized");
  let notes = await musicVae.sample(1, 1.5);
  playLoop(notes[0]);
};

const stop = () => {
  stopped.value = true;
  running.value = false;
  player.stop();
};
</script>
