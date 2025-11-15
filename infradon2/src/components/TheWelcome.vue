<script setup lang="ts">
import { ref, onMounted } from 'vue'
import PouchDB from 'pouchdb'

declare interface Post {
  _id: string
  _rev: string
  title: string
  content: string
  attributes: {
    creation_date: any
  }
}

const storage = ref()
const postsData = ref<Post[]>([])
const sync = ref()

onMounted(() => {
  console.log('=> Composant initialisé')
  initDatabase()
})

const initDatabase = () => {
  console.log('=> Connexion à la base de données')
  const localdb = new PouchDB('collection_infradon2')
  if (localdb) {
    console.log('Connected to collection : ' + localdb?.name)
    storage.value = localdb
    localdb.replicate.from('http://calimo:admin@localhost:5984/infradon2')
    .on('complete', syncData)
    .then((_result) => {
      fetchData()
    })
  } else {
    console.warn('Something went wrong')
  }
}

const syncData = () => {
  sync.value = storage.value
  .sync('http://calimo:admin@localhost:5984/infradon2', {live: true, retry: true})
  .on('change', fetchData)
};

const toggle = () => {
  if (sync.value) {
    sync.value.cancel()
    sync.value = null
  } else {
    syncData()
  }
}

const fetchData = (): any => {
  storage.value
    .allDocs({
      include_docs: true,
    })
    .then((result: any) => {
      console.log('=> Données récupérées :', result.rows)
      postsData.value = result.rows.map((row: any) => row.doc)
    })
    .catch((error: any) => {
      console.error('Erreur lors de la récupération des données :', error)
    })
}

const createDoc = (): any => {
  counter++
  storage.value
    .post({
      title: 'Document ' + counter,
      content: 'Contenu du document : ' + words[Math.floor(Math.random() * words.length)],
    })
    .then(function (response: any) {
      fetchData()
      console.log(response)
    })
    .catch(function (err: any) {
      console.log(err)
    })
}

const deleteDoc = (post: Post): any => {
  storage.value
    .remove(post)
    .then(function (response: any) {
      fetchData()
      console.log(response)
    })
    .catch(function (err: any) {
      console.log(err)
    })
}

const updateDoc = (post: Post): any => {
  post.content = 'Contenu du document : ' + words[Math.floor(Math.random() * words.length)]
  storage.value
    .put(post)
    .then(function (response: any) {
      fetchData()
      console.log(response)
    })
    .catch(function (err: any) {
      console.log(err)
    })
}

let counter = 0
const words = [
  'léa',
  'inoé',
  'camilo',
  'teicir',
  'yannis',
  'elia',
  'loann',
  'sasita',
  'sarah',
  'enya',
  'gabriel',
  'nuno',
  'tanguy',
  'loic',
  'dylan',
  'liliana',
  'thierry',
  'valentin',
]
</script>

<template>
  <h1>Fetch Data</h1>
  <label class="switch">
    <input type="checkbox" checked @click="toggle()"><span class="slider round"></span>
  </label>
  <label v-if="sync">Online</label>
  <label v-else>Offline</label>
<br>
  <button @click="createDoc">Ajouter un document</button>
  <article v-for="post in postsData" v-bind:key="(post as any).id">
    <h2>{{ post.title }}</h2>
    <p>{{ post.content }}</p>
    <button @click="updateDoc(post)">Update</button>
    <button @click="deleteDoc(post)">Effacer</button>
  </article>
</template>

<style scoped>
 /* The switch - the box around the slider */
.switch {
  position: relative;
  display: inline-block;
  width: 30px;
  height: 17px;
}

/* Hide default HTML checkbox */
.switch input {
  opacity: 0;
  width: 0;
  height: 0;
}

/* The slider */
.slider {
  position: absolute;
  cursor: pointer;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-color: #ccc;
  -webkit-transition: .4s;
  transition: .4s;
}

.slider:before {
  position: absolute;
  content: "";
  height: 13px;
  width: 13px;
  left: 2px;
  bottom: 2px;
  background-color: white;
  -webkit-transition: .4s;
  transition: .4s;
}

input:checked + .slider {
  background-color: #2196F3;
}

input:focus + .slider {
  box-shadow: 0 0 1px #2196F3;
}

input:checked + .slider:before {
  -webkit-transform: translateX(13px);
  -ms-transform: translateX(13px);
  transform: translateX(13px);
}

/* Rounded sliders */
.slider.round {
  border-radius: 17px;
}

.slider.round:before {
  border-radius: 50%;
} 
</style>