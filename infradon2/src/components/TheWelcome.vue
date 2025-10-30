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

onMounted(() => {
  console.log('=> Composant initialisé')
  initDatabase()
  fetchData()
})

const initDatabase = () => {
  console.log('=> Connexion à la base de données')
  const db = new PouchDB('http://calimo:admin@localhost:5984/infradon2')
  if (db) {
    console.log('Connected to collection : ' + db?.name)
    storage.value = db
  } else {
    console.warn('Something went wrong')
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
  <button @click="createDoc">Ajouter un document</button>
  <article v-for="post in postsData" v-bind:key="(post as any).id">
    <h2>{{ post.title }}</h2>
    <p>{{ post.content }}</p>
    <button @click="updateDoc(post)">Update</button>
    <button @click="deleteDoc(post)">Effacer</button>
  </article>
</template>
