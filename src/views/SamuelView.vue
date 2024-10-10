<script lang="ts">
import PouchDB from 'pouchdb'

declare interface Post {
  _id: string
  doc: {
    nom_famille: string
    description: int
    details: {
      date_creation: string
      auteur: string
    }
  }
}

export default {
  data() {
    return {
      datas: [],
      databaseReference: null
    }
  },

  methods: {
    inc() {
      // old
      // this.total++;
    },

    initDatabase() {
      const db = new PouchDB('http://localhost:5984/cours3')
      if (db) {
        console.log("Connected to collection 'post'")
      } else {
        console.warn('Something went wrong')
      }
      this.storage = db
    },

    fetchData() {}
  },

  mounted() {
    this.initDatabase()
  }
}
</script>

<template>
  <h1>InfraDon2</h1>
  <p>Counter: {{ datas }}</p>
  <button type="button" role="button" @click="inc">+1</button>
</template>
