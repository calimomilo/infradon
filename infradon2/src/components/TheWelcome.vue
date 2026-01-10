<script setup lang="ts">
import { ref, onMounted, Comment } from 'vue'
import PouchDB from 'pouchdb'
import PouchDBFind from 'pouchdb-find'
import { classicNameResolver } from 'typescript'

PouchDB.plugin(PouchDBFind)

declare interface Post {
  _conflicts: null
  _id: string
  _rev: string
  message: string
  author: string
  likes: number
  comments: Comment[]
  loaded_comments: string
  creation_date: any
  update_date: any
}

declare interface Comment {
  _conflicts: null
  _id: string
  _rev: string
  comment: string
  author: string
  post_id: string
  creation_date: any
  update_date: any
}

const storage = ref()
const storageComm = ref()
const postsData = ref<Post[]>([])
const sync = ref()
const syncComm = ref()

let postsLoaded = 10
let postsReplicated = false
let commsReplicated = false

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

const newComms = ref(new Map())

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
          fields: ['post_id', 'update_date'],
        },
      })
      .then(console.log('comment post_id index created'))

    localdb.replicate
      .from('http://calimo:admin@localhost:5984/posts-db-camilo')
      .on('complete', syncPostsData)
      .then((_result) => {
        postsReplicated = true
        checkReplicated()
      })

    localdbcomm.replicate
      .from('http://calimo:admin@localhost:5984/comments-db-camilo')
      .on('complete', syncCommsData)
      .then((_result) => {
        commsReplicated = true
        checkReplicated()
      })
  } else {
    console.warn('Something went wrong')
  }
}

const checkReplicated = () => {
  if (postsReplicated && commsReplicated) {
    fetchPostData()
  }
}

const syncPostsData = () => {
  sync.value = storage.value.sync('http://calimo:admin@localhost:5984/posts-db-camilo', {
    live: true,
    retry: true,
  })
  //.on('change', fetchPostData)
}

const syncCommsData = () => {
  syncComm.value = storageComm.value.sync('http://calimo:admin@localhost:5984/comments-db-camilo', {
    live: true,
    retry: true,
  })
  //.on('change', fetchAllCommsData)
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

        postsData.value.forEach((post: Post) => {
          newComms.value.set(post._id, { comment: '', author: words[0] })

          if (post.loaded_comments === undefined) {
            post.loaded_comments = 'last'
          }

          fetchCommsData(post)
        })
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
    .find({
      selector: { likes: { $gte: null } },
      conflicts: true,
      sort: [{ likes: 'desc' }],
      limit: postsLoaded,
    })
    .then((result: any) => {
      console.log('=> Données récupérées :', result.rows)
      postsData.value = result.docs

      postsData.value.forEach((post: Post) => {
        newComms.value.set(post._id, { comment: '', author: words[0] })

        if (post.loaded_comments === undefined) {
          post.loaded_comments = 'last'
        }

        fetchCommsData(post)
      })
    })
    .catch((error: any) => {
      console.error('Erreur lors de la récupération des données :', error)
    })
}

const fetchCommsData = (post: Post): any => {
  if (post.loaded_comments === 'last') {
    storageComm.value
      .find({
        selector: { post_id: post._id, update_date: { $gte: null } },
        conflicts: true,
        sort: ['post_id', { update_date: 'desc' }],
        limit: 1,
      })
      .then((result: any) => {
        console.log('=> Données commentaires récupérées :', result.docs)
        post.comments = result.docs
      })
      .catch((error: any) => {
        console.error('Erreur lors de la récupération des données :', error)
      })
  } else if (post.loaded_comments === 'all') {
    storageComm.value
      .find({
        selector: { post_id: post._id, update_date: { $gte: null } },
        conflicts: true,
        sort: ['post_id', { update_date: 'asc' }],
      })
      .then((result: any) => {
        console.log('=> Données commentaires récupérées :', result.docs)
        post.comments = result.docs
      })
      .catch((error: any) => {
        console.error('Erreur lors de la récupération des données :', error)
      })
  }
}

const toggleComms = (post: Post): any => {
  post.loaded_comments = post.loaded_comments === 'last' ? 'all' : 'last'
  console.log(post.loaded_comments)
  fetchCommsData(post)
}

const loadMorePosts = (): any => {
  postsLoaded += 10
  fetchPostData()
}

const collapsePosts = (): any => {
  postsLoaded = 10
  fetchPostData()
}

const createPost = (): any => {
  storage.value
    .post({
      message: newPost.value.message,
      author: newPost.value.author,
      likes: 0,
      creation_date: Date.now(),
      update_date: Date.now(),
    })
    .then(function (response: any) {
      newPost.value.message = ''
      newPost.value.author = words[0]
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
      comment: newComms.value.get(post._id).comment,
      author: newComms.value.get(post._id).author,
      post_id: post._id,
      creation_date: Date.now(),
      update_date: Date.now(),
    })
    .then(function (response: any) {
      console.log(response)
      newComms.value.set(post._id, { comment: '', author: words[0] })
      fetchPostData()
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
            //fetchCommsData()
            console.log(response)
          })
          .catch((err: any) => {
            console.log(err)
          })
      })
    })
}

const deleteComm = (comm: Comment, post: Post): any => {
  storageComm.value
    .remove(comm)
    .then((response: any) => {
      fetchCommsData(post)
      console.log(response)
    })
    .catch((err: any) => {
      console.log(err)
    })
}

const updatePost = (post: Post): any => {
  post.update_date = Date.now()
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

const updateComm = (comm: Comment, post: Post): any => {
  comm.update_date = Date.now()
  storageComm.value
    .put(comm)
    .then(function (response: any) {
      fetchCommsData(post)
      console.log(response)
    })
    .catch(function (err: any) {
      console.log(err)
    })
}

const likePost = (post: Post): any => {
  post.likes += 1
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

const populatePosts = (amount: number): any => {
  for (let i = 0; i < amount; i++) {
    newPost.value.message = Math.random() * 1000000000 + 'a'
    newPost.value.author = words[Math.floor(Math.random() * words.length)]
    createPost()
  }
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
  <button @click="populatePosts(10)">Add 10 messages</button>
  <button @click="populatePosts(50)">Add 50 messages</button>
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
      <p class="date">{{ new Date(post.update_date).toLocaleString() }}</p>
      <p class="date">{{ post.likes }} likes</p>
      <button @click="updatePost(post)">Update</button>
      <button @click="deletePost(post)">Delete</button>
      <button @click="likePost(post)">Like</button>
      <hr />
      <p>Comments :</p>
      <input type="text" placeholder="new comment" v-model="newComms.get(post._id).comment" />
      <select v-model="newComms.get(post._id).author">
        <option v-for="name in words" v-bind:key="name" v-bind:value="name">
          {{ name }}
        </option></select
      ><br />
      <button @click="createComm(post)">Add comment</button>
      <br />
      <article v-for="comm in post.comments" v-bind:key="(comm as any).id">
        <br />
        <div class="box">
          <input type="text" v-model="comm.comment" />
          <select v-model="comm.author">
            <option v-for="name in words" v-bind:key="name" v-bind:value="name">{{ name }}</option>
          </select>
          <span class="conflicts" v-if="comm._conflicts">Attention, conflits !</span>
          <br />
          <p class="date">{{ new Date(comm.update_date).toLocaleString() }}</p>
          <button @click="updateComm(comm, post)">Update</button>
          <button @click="deleteComm(comm, post)">Delete</button>
        </div>
      </article>
      <br />
      <button @click="toggleComms(post)">
        <p v-if="post.loaded_comments === 'last'">Load all comments</p>
        <p v-else>Collapse comments</p>
      </button>
    </div>
  </article>
  <br />
  <button v-if="postsData.length === postsLoaded" @click="loadMorePosts">Load more posts</button>
  <button v-if="postsLoaded !== 10" @click="collapsePosts">Collapse posts</button>
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

hr {
  margin: 15px 0 5px 0;
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
