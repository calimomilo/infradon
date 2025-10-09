<script setup lang="ts">
import { ref, onMounted } from 'vue';
import PouchDB from 'pouchdb';

const storage = ref();

onMounted(() => {
  console.log('=> Composant initialisé');
  initDatabase();
});

const initDatabase = () => {
  console.log('=> Connexion à la base de données');
  const db = new PouchDB('http://calimo:c@lim=m0m!lo@localhost:5984/infradon2')
  if (db) {
    console.log("Connected to collection : " + db?.name)
    storage.value = db
  } else {
    console.warn('Something went wrong')
  }
}

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
