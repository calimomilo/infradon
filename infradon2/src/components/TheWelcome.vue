<script setup lang="ts">
import { ref } from 'vue';


const counter = ref(0);

const increment = () => {
  counter.value++;
}

const reset = () => {
  counter.value = 0;
}

</script>

<template>
  <h1>Hello World</h1>
  <p style="font-size: 50px;">{{  counter  }}</p>
  <button @click="increment">increment</button>
  <button @click="reset">reset</button>
</template>
