<template>
  <div class="container">
    <h1>Lissajous Curve Demo</h1>
    <div class="input-container">
      <label for="midiNotes">Enter MIDI notes (space-separated):</label>
      <input id="midiNotes" v-model="midiNotesInput" @input="updateMidiNotes" />
    </div>
    <div class="lissajous-container">
      <Lissajous :midiNotes="midiNotes" />
    </div>
  </div>
</template>

<script setup>
import { ref, computed } from "vue";
import Lissajous from "./components/Lissajous.vue";

const midiNotesInput = ref("60 64 67");
const midiNotes = computed(() =>
  midiNotesInput.value
    .split(" ")
    .map(Number)
    .filter((n) => !isNaN(n))
);

const updateMidiNotes = () => {
  // This function is called on input change, but we don't need to do anything
  // extra here as the computed property will update automatically
};
</script>

<style scoped>
.container {
  max-width: 800px;
  margin: 0 auto;
  padding: 20px;
}

.input-container {
  margin-bottom: 20px;
}

.lissajous-container {
  width: 100%;
  height: 600px;
}
</style>
