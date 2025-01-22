<template>
  <div class="infra-don">
    <!-- Titre principal -->
    <header>
      <h1>InfraDon2 - Mon Projet</h1>
      <p>Gérez vos posts facilement et efficacement.</p>
    </header>

    <!-- Compteur de posts -->
    <section class="post-counter">
      <p>Nombre de posts : {{ postsCount }}</p>
    </section>

    <!-- Boutons d'actions -->
    <section class="actions">
      <button @click="addNewPost">Créer un post</button>
      <button @click="syncToLocal">Synchroniser vers local</button>
      <button @click="syncToRemote">Synchroniser vers distant</button>
      <button @click="initializeIndex">Initialiser un index</button>
    </section>

    <!-- Zone de recherche -->
    <section class="search">
      <input v-model="searchQuery" placeholder="Rechercher..." />
      <button @click="searchPosts">Rechercher</button>
    </section>

    <!-- Liste des posts -->
    <section class="post-list">
      <div v-for="post in posts" :key="post._id || Math.random()" class="post">
        <!-- Informations du post -->
        <p><strong>Nom :</strong> {{ post.name || 'Sans nom' }}</p>
        <p><strong>Description :</strong> {{ post.description || 'Aucune description' }}</p>

        <!-- Actions pour chaque post -->
        <div class="post-actions">
          <button @click="editPost(post._id)">Modifier le nom</button>
          <button @click="editDescription(post._id)">Modifier la description</button>
          <button v-if="post._rev" @click="deletePost(post._id, post._rev)">Supprimer</button>
        </div>

        <!-- Pièces jointes -->
        <div
          v-if="post.attachments && Object.keys(post.attachments).length > 0"
          class="attachments"
        >
          <h4>Pièces jointes :</h4>
          <div v-for="(file, name) in post.attachments" :key="name" class="attachment">
            <!-- Affichage de l'image si c'est un fichier image -->
            <div v-if="file.type.startsWith('image/')">
              <img :src="file.url" :alt="name" class="attachment-image" />
            </div>
            <!-- Lien de téléchargement pour les autres fichiers -->
            <div v-else>
              <a :href="file.url" download>{{ name }}</a>
            </div>
          </div>
        </div>

        <!-- Ajout de pièce jointe -->
        <input type="file" @change="uploadAttachment(post._id, $event)" />
      </div>
    </section>
  </div>
</template>

<script lang="ts">
import PouchDB from 'pouchdb'
import PouchDBFind from 'pouchdb-find'
PouchDB.plugin(PouchDBFind)

// Interface pour les pièces jointes
interface Attachment {
  url: string
  type: string
}

// Définition de l'interface pour les posts
interface Post {
  _id: string
  _rev?: string
  name?: string
  description?: string
  attachments?: Record<string, Attachment>
}

// Interface pour le document PouchDB
interface PouchDocument extends Post {
  _rev: string
}

export default {
  data() {
    return {
      posts: [] as Post[],
      postsCount: 0,
      searchQuery: '',
      localDB: new PouchDB('localDB'),
      remoteDB: new PouchDB('http://admin:100403@localhost:5984/posts', {
        fetch: (url: RequestInfo, opts: RequestInit = {}) => {
          opts.mode = 'cors'
          return fetch(url, opts)
        }
      })
    }
  },

  methods: {
    async addNewPost() {
      const newPost: Post = {
        _id: new Date().toISOString(),
        name: 'Nouveau post',
        description: 'Description par défaut',
        attachments: {}
      }
      await this.localDB.put(newPost)
      this.refreshPosts()
    },

    async deletePost(id: string, rev: string) {
      if (!id || !rev) {
        console.error('ID ou révision manquants pour la suppression')
        return
      }
      await this.localDB.remove(id, rev)
      this.refreshPosts()
    },

    async editPost(postId: string) {
      const post = this.posts.find((p) => p._id === postId)
      if (post) {
        const newName = prompt('Modifier le nom du post :', post.name)
        if (newName) {
          const updatedPost = { ...post, name: newName }
          await this.localDB.put(updatedPost)
          this.refreshPosts()
        }
      }
    },

    async editDescription(postId: string) {
      const post = this.posts.find((p) => p._id === postId)
      if (post) {
        const newDescription = prompt('Modifier la description du post :', post.description)
        if (newDescription) {
          const updatedPost = { ...post, description: newDescription }
          await this.localDB.put(updatedPost)
          this.refreshPosts()
        }
      }
    },

    async refreshPosts() {
      const allDocs = await this.localDB.allDocs({ include_docs: true })
      this.posts = allDocs.rows.map((row) => {
        const doc = row.doc as PouchDocument
        return {
          _id: doc._id,
          _rev: doc._rev,
          name: doc.name || 'Sans nom',
          description: doc.description || 'Aucune description',
          attachments: doc.attachments || {}
        }
      })
      this.postsCount = this.posts.length
    },

    async syncToLocal() {
      await this.localDB.replicate.from(this.remoteDB)
      this.refreshPosts()
    },

    async syncToRemote() {
      await this.localDB.replicate.to(this.remoteDB)
    },

    async initializeIndex() {
      await this.localDB.createIndex({ index: { fields: ['name'] } })
    },

    async searchPosts() {
      const results = await this.localDB.find({
        selector: { name: { $regex: this.searchQuery } }
      })
      this.posts = results.docs as Post[]
    },

    async uploadAttachment(postId: string, event: Event) {
      const file = (event.target as HTMLInputElement).files?.[0]
      if (!file) {
        console.error('Aucun fichier sélectionné')
        return
      }

      const reader = new FileReader()

      reader.onload = async () => {
        try {
          const arrayBuffer = reader.result as ArrayBuffer
          if (!arrayBuffer) {
            console.error('Erreur : arrayBuffer est null ou invalide')
            return
          }

          const post = (await this.localDB.get(postId)) as PouchDocument
          if (!post._rev) {
            console.error('Révision (_rev) manquante pour le post')
            return
          }

          const url = URL.createObjectURL(new Blob([arrayBuffer], { type: file.type }))

          const updatedPost = {
            ...post,
            attachments: {
              ...(post.attachments || {}),
              [file.name]: {
                url: url as string,
                type: file.type
              } as Attachment
            }
          }

          await this.localDB.put(updatedPost)
          console.log('Pièce jointe ajoutée avec succès.')
          this.refreshPosts()
        } catch (error) {}
      }

      reader.onerror = (error) => {
        console.error('Erreur lors de la lecture du fichier :', error)
      }

      reader.readAsArrayBuffer(file)
    }
  },

  async mounted() {
    this.refreshPosts()
  }
}
</script>

<style scoped>
.infra-don {
  font-family: 'Roboto', sans-serif;
  padding: 20px;
  background-color: #121212;
  color: #f0f0f0;
  border-radius: 10px;
}

h1 {
  font-size: 2.5rem;
  color: #66bb6a;
  text-align: center;
}

p {
  font-size: 1rem;
  color: #c8e6c9;
  margin: 10px 0;
  text-align: center;
}

.actions,
.search {
  display: flex;
  justify-content: center;
  gap: 10px;
  margin-bottom: 20px;
}

.actions button,
.search button {
  background-color: #66bb6a;
  color: #121212;
  border: none;
  padding: 10px 15px;
  font-size: 1rem;
  border-radius: 5px;
  cursor: pointer;
  transition: background-color 0.3s;
}

.actions button:hover,
.search button:hover {
  background-color: #4caf50;
}

.search input {
  padding: 10px;
  border: 2px solid #66bb6a;
  border-radius: 5px;
  font-size: 1rem;
  color: #121212;
  outline: none;
}

.post {
  background-color: #1e1e1e;
  border: 1px solid #4caf50;
  border-radius: 10px;
  padding: 15px;
  margin-top: 15px;
  transition:
    transform 0.2s,
    box-shadow 0.2s;
}

.post:hover {
  transform: scale(1.02);
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
}

.post p {
  margin: 5px 0;
  text-align: left;
}

.post-actions {
  display: flex;
  gap: 15px;
  flex-wrap: wrap;
  margin-top: 10px;
}

.post button {
  background-color: #66bb6a;
  color: #121212;
  border: none;
  padding: 5px 10px;
  border-radius: 5px;
  font-size: 0.9rem;
  cursor: pointer;
  transition: background-color 0.3s;
}

.post button:hover {
  background-color: #4caf50;
}

input[type='file'] {
  margin-top: 10px;
  background-color: #66bb6a;
  color: #121212;
  border: none;
  border-radius: 5px;
  padding: 5px;
  cursor: pointer;
}

input[type='file']::-webkit-file-upload-button {
  background-color: #2c3e50;
  color: #fff;
  border: none;
  border-radius: 5px;
  padding: 5px 10px;
  cursor: pointer;
}

input[type='file']::-webkit-file-upload-button:hover {
  background-color: #388e3c;
}

.attachment-image {
  max-width: 100%;
  max-height: 200px;
  border-radius: 5px;
  margin-top: 10px;
}
</style>
