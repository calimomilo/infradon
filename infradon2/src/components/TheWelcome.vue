<script setup lang="ts">
import { ref, onMounted, Comment } from 'vue'
import PouchDB from 'pouchdb'
import PouchDBFind from 'pouchdb-find'

PouchDB.plugin(PouchDBFind)

declare interface Post {
  _conflicts: null
  _id: string
  _rev: string
  message: string
  author: string
  likes: number
  attributes: {
    creation_date: any
    update_date: any
  }
}

declare interface Comment {
  _conflicts: null
  _id: string
  _rev: string
  comment: string
  author: string
  post_id: string
  attributes: {
    creation_date: any
    update_date: any
  }
}

const storage = ref()
const storageComm = ref()
const postsData = ref<Post[]>([])
const commsData = ref<Comment[]>([])
const sync = ref()
const syncComm = ref()

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
  'benoît',
  'chloé',
]

const newPost = ref({
  message: '',
  author: words[0],
})

const newComm = ref({
  comment: '',
  author: words[0],
})

onMounted(() => {
  console.log('=> Composant initialisé')
  initDatabase()
})

const initDatabase = () => {
  console.log('=> Connexion à la base de données')
  const localdb = new PouchDB('collection_infradon2')
  const localdbcomm = new PouchDB('collection_commentaires')
  if (localdb && localdbcomm) {
    console.log('Connected to collections : ' + localdb?.name + ' and ' + localdbcomm.name)
    storage.value = localdb
    storageComm.value = localdbcomm

    storage.value
      .createIndex({
        index: {
          fields: ['author'],
        },
      })
      .then(console.log('post author index created'))

    storage.value
      .createIndex({
        index: {
          fields: ['likes'],
        },
      })
      .then(console.log('post likes index created'))

    storageComm.value
      .createIndex({
        index: {
          fields: ['post_id'],
        },
      })
      .then(console.log('comment post_id index created'))

    localdb.replicate
      .from('http://calimo:admin@localhost:5984/infradon2')
      .on('complete', syncPostsData)
      .then((_result) => {
        fetchPostData()
      })

    localdbcomm.replicate
      .from('http://calimo:admin@localhost:5984/infradon2-comms')
      .on('complete', syncCommsData)
      .then((_result) => {
        //fetchCommsData()
      })
  } else {
    console.warn('Something went wrong')
  }
}

const syncPostsData = () => {
  sync.value = storage.value
    .sync('http://calimo:admin@localhost:5984/infradon2', { live: true, retry: true })
    .on('change', fetchPostData)
}

const syncCommsData = () => {
  syncComm.value = storageComm.value
    .sync('http://calimo:admin@localhost:5984/infradon2-comms', { live: true, retry: true })
    .on('change', fetchCommsData)
}

const toggle = () => {
  if (sync.value) {
    sync.value.cancel()
    sync.value = null
  } else {
    syncPostsData()
  }
  if (syncComm.value) {
    syncComm.value.cancel()
    syncComm.value = null
  } else {
    syncCommsData()
  }
}

const search = (event: Event) => {
  event.target.blur()

  if (event.target.value === '') {
    fetchPostData()
  } else {
    storage.value
      .find({
        selector: { author: event.target.value },
      })
      .then((result: any) => {
        console.log('=> Données récupérées :', result.docs)
        postsData.value = result.docs
        // console.log(postsData)
      })
      .catch((error: any) => {
        console.error('Erreur lors de la récupération des données :', error)
      })
  }
}

const searchReset = () => {
  document.querySelector('.search').value = ''
  fetchPostData()
}

const fetchPostData = (): any => {
  storage.value
    .allDocs({
      include_docs: true,
      conflicts: true,
    })
    .then((result: any) => {
      console.log('=> Données récupérées :', result.rows)
      postsData.value = result.rows
        .filter((el: any) => !el.doc._id.startsWith('_design'))
        .map((row: any) => row.doc)
      // console.log(postsData)
    })
    .catch((error: any) => {
      console.error('Erreur lors de la récupération des données :', error)
    })
}

const fetchCommsData = (post: Post): any => {
  storageComm.value
    .find({
      selector: { post_id: post._id },
    })
    .then((result: any) => {
      console.log('=> Données récupérées :', result.docs)
      commsData.value = result.docs
        //.filter((el: any) => !el.doc._id.startsWith('_design'))
        //.map((row: any) => row.doc)
      // console.log(commsData)
      return commsData
    })
    .catch((error: any) => {
      console.error('Erreur lors de la récupération des données :', error)
    })
}

const createPost = (): any => {
  storage.value
    .post({
      message: newPost.value.message,
      author: newPost.value.author,
      likes: 0,
      attributes: {
        creation_date: Date.now(),
        update_date: Date.now(),
      },
    })
    .then(function (response: any) {
      fetchPostData()
      console.log(response)
    })
    .catch(function (err: any) {
      console.log(err)
    })
}

const createComm = (post: Post): any => {
  storageComm.value
    .post({
      comment: newComm.value.comment,
      author: newComm.value.author,
      post_id: post._id,
      attributes: {
        creation_date: Date.now(),
        update_date: Date.now(),
      },
    })
    .then(function (response: any) {
      fetchCommsData()
      console.log(response)
    })
    .catch(function (err: any) {
      console.log(err)
    })
}

const deletePost = (post: Post): any => {
  storage.value
    .remove(post)
    .then((response: any) => {
      fetchPostData()
      console.log(response)
    })
    .catch((err: any) => {
      console.log(err)
    })

  storageComm.value
    .find({
      selector: { post_id: post._id },
    })
    .then((result: any) => {
      result.docs.map((el: any) => {
        storageComm.value
          .remove(el)
          .then((response: any) => {
            fetchCommsData()
            console.log(response)
          })
          .catch((err: any) => {
            console.log(err)
          })
      })
    })
}

const deleteComm = (comm: Comment): any => {
  storageComm.value
    .remove(comm)
    .then((response: any) => {
      fetchCommsData()
      console.log(response)
    })
    .catch((err: any) => {
      console.log(err)
    })
}

const updatePost = (post: Post): any => {
  post.attributes.update_date = Date.now()
  storage.value
    .put(post)
    .then(function (response: any) {
      fetchPostData()
      console.log(response)
    })
    .catch(function (err: any) {
      console.log(err)
    })
}

const updateComm = (comm: Comment): any => {
  comm.attributes.update_date = Date.now()
  storageComm.value
    .put(comm)
    .then(function (response: any) {
      fetchCommsData()
      console.log(response)
    })
    .catch(function (err: any) {
      console.log(err)
    })
}
</script>

<template>
  <h1>Fetch Data</h1>
  <label class="switch">
    <input type="checkbox" checked @click="toggle" /><span class="slider round"></span>
  </label>
  <label v-if="sync"> Online</label>
  <label v-else> Offline</label>
  <br /><br />
  <input type="text" placeholder="new message" v-model="newPost.message" />
  <select v-model="newPost.author">
    <option v-for="name in words" v-bind:key="name" v-bind:value="name">{{ name }}</option></select
  ><br />
  <button @click="createPost">Add message</button>
  <br /><br />
  <hr />
  <br />
  <input type="text" placeholder="Search" @keyup.enter="search" class="search" />
  <button @click="searchReset">X</button>
  <br />
  <article v-for="post in postsData" v-bind:key="(post as any).id">
    <br />
    <div class="box">
      <input type="text" v-model="post.message" />
      <select v-model="post.author">
        <option v-for="name in words" v-bind:key="name" v-bind:value="name">{{ name }}</option>
      </select>
      <span class="conflicts" v-if="post._conflicts">Attention, conflits !</span>
      <br />
      <p class="date">{{ new Date(post.attributes.update_date).toLocaleString() }}</p>
      <p class="date">{{ post.likes}} likes</p>
      <button @click="updatePost(post)">Update</button>
      <button @click="deletePost(post)">Delete</button>
      <button @click="post.likes+=1">Like</button>
      <article v-for="comm in fetchCommsData(post)" v-bind:key="(comm as any).id">
        <br />
        <div class="box">
          <input type="text" v-model="comm.message" />
          <select v-model="comm.author">
            <option v-for="name in words" v-bind:key="name" v-bind:value="name">{{ name }}</option>
          </select>
          <span class="conflicts" v-if="comm._conflicts">Attention, conflits !</span>
          <br />
          <p class="date">{{ new Date(comm.attributes.update_date).toLocaleString() }}</p>
          <button @click="updateComm(comm)">Update</button>
          <button @click="deleteComm(comm)">Delete</button>
        </div>
      </article>
    </div>
  </article>
</template>

<style scoped>
.conflicts {
  color: rgb(140, 3, 3);
  font-style: italic;
  font-size: small;
  margin-left: 10px;
}

.date {
  color: gray;
  font-size: small;
}

.box {
  padding: 10px;
  border-radius: 2px;
  background-color: rgba(250, 250, 250, 0.1);
}

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
  -webkit-transition: 0.4s;
  transition: 0.4s;
}

.slider:before {
  position: absolute;
  content: '';
  height: 13px;
  width: 13px;
  left: 2px;
  bottom: 2px;
  background-color: white;
  -webkit-transition: 0.4s;
  transition: 0.4s;
}

input:checked + .slider {
  background-color: #2196f3;
}

input:focus + .slider {
  box-shadow: 0 0 1px #2196f3;
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
