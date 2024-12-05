<script lang="ts">
import { ref } from 'vue'
import PouchDB from 'pouchdb'

declare interface Post {
  _id: string
  doc: {
    nom_famille: string
    description: number
    details: {
      date_creation: string
      auteur: string
    }
  }
}

export default {
  data() {
    return {
      total: 0,
      postsData: [] as Post[],
      document: null as Post | null,
      storage: null as PouchDB.Database | null
    }
  },

  mounted() {
    this.initDatabase()
    this.fetchData()
    this.detectChanges()
  },

  methods: {
    initDatabase() {
      const db = new PouchDB('http://localhost:5984/cours3')
      if (db) {
        console.log("Connected to collection 'post'")
      } else {
        console.warn('Something went wrong')
      }
      this.storage = db
    },

    fetchData() {
      const storage = ref(this.storage)
      const self = this
      if (storage.value) {
        storage.value
          .allDocs({
            include_docs: true,
            attachments: true
          })
          .then(
            function (result: any) {
              console.log('fetchData success', result)
              self.postsData = result.rows
            }.bind(this)
          )
          .catch(function (error: any) {
            console.log('fetchData error', error)
          })
      }
    },

    addPost() {
      const newPost: Post = {
        _id: new Date().toISOString(), // ID unique basé sur la date
        doc: {
          nom_famille: 'Nom de famille test',
          description: 1,
          details: {
            date_creation: new Date().toISOString(),
            auteur: 'Auteur test'
          }
        }
      }
      this.putDocument(newPost) // Ajouter le nouveau post
    },

    putDocument(document: Post) {
      const db = ref(this.storage).value
      if (db) {
        db.put(document)
          .then(() => {
            console.log('Add ok')
            this.fetchData() // Rafraîchir les données après ajout
          })
          .catch((error) => {
            console.log('Add ko', error)
          })
      }
    },

    syncDatabase() {
      if (this.storage) {
        const dbDistant = new PouchDB('http://adresse_du_serveur_couchdb/nom_base_distant')
        this.storage
          .sync(dbDistant)
          .on('complete', () => {
            console.log('Synchronisation réussie')
          })
          .on('error', (err) => {
            console.error('Erreur de synchronisation', err)
          })
      } else {
        console.warn("La base de données locale n'est pas initialisée")
      }
    },

    detectChanges() {
      if (this.storage) {
        this.storage
          .changes({
            since: 'now',
            live: true
          })
          .on('change', (change) => {
            console.log('Changement détecté', change)
            this.fetchData() // Recharger les données si changement détecté
          })
      } else {
        console.warn("La base de données locale n'est pas initialisée")
      }
    }
  }
}
</script>

<template>
  <h1>Nombre de post: {{ postsData.length }}</h1>
  <button @click="addPost">Ajouter un nouveau post</button>
  <ul>
    <li v-for="post in postsData" :key="post._id">
      <div class="ucfirst">
        {{ post.doc.nom_famille }}
        <em style="font-size: x-small" v-if="post.doc.details?.date_creation">
          - {{ post.doc.details?.date_creation }}
        </em>
      </div>
    </li>
  </ul>
</template>
