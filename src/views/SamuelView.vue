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
          <button @click="editPost(post._id)">Modifier</button>
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

// Définition de l'interface pour les posts
interface Post {
  _id: string // Identifiant unique
  _rev?: string // Révision du document
  name?: string // Nom du post
  description?: string // Description du post
  attachments?: Record<string, { url: string; type: string }> // Pièces jointes
}

export default {
  data() {
    return {
      // Données principales
      posts: [] as Post[], // Liste des posts
      postsCount: 0, // Nombre total de posts
      searchQuery: '', // Requête de recherche
      localDB: new PouchDB('localDB'), // Base de données locale
      remoteDB: new PouchDB('http://admin:100403@localhost:5984/posts', {
        // Configuration pour la synchronisation distante
        fetch: (url: RequestInfo, opts: RequestInit = {}) => {
          opts.mode = 'cors' // Définit le mode CORS
          return fetch(url, opts)
        }
      })
    }
  },

  methods: {
    /**
     * Crée un nouveau post avec des données par défaut.
     */
    async addNewPost() {
      const newPost: Post = {
        _id: new Date().toISOString(),
        name: 'Nouveau post',
        description: 'Description par défaut',
        attachments: {}
      }
      await this.localDB.put(newPost) // Ajout dans la base locale
      this.refreshPosts() // Rafraîchissement de la liste des posts
    },

    /**
     * Supprime un post à partir de son ID et de sa révision.
     * @param id - Identifiant du post
     * @param rev - Révision du post
     */
    async deletePost(id: string, rev: string) {
      if (!id || !rev) {
        console.error('ID ou révision manquants pour la suppression')
        return
      }
      await this.localDB.remove(id, rev)
      this.refreshPosts()
    },

    /**
     * Permet de modifier un post.
     * @param postId - Identifiant du post à modifier
     */
    async editPost(postId: string) {
      const post = this.posts.find((p) => p._id === postId)
      if (post) {
        const newName = prompt('Modifier le nom du post :', post.name)
        if (newName) {
          post.name = newName
          await this.localDB.put(post) // Met à jour dans la base locale
          this.refreshPosts()
        }
      }
    },

    /**
     * Rafraîchit la liste des posts depuis la base locale.
     */
    async refreshPosts() {
      const allDocs = await this.localDB.allDocs({ include_docs: true })
      this.posts = allDocs.rows.map((row) => {
        const doc = row.doc as Post // Cast pour l'interface `Post`
        return {
          _id: doc._id || '',
          _rev: doc._rev || '',
          name: doc.name || 'Sans nom',
          description: doc.description || 'Aucune description',
          attachments: doc.attachments || {}
        }
      })
      this.postsCount = this.posts.length
    },

    /**
     * Synchronise la base locale avec la base distante (vers local).
     */
    async syncToLocal() {
      await this.localDB.replicate.from(this.remoteDB)
      this.refreshPosts()
    },

    /**
     * Synchronise la base locale avec la base distante (vers distant).
     */
    async syncToRemote() {
      await this.localDB.replicate.to(this.remoteDB)
    },

    /**
     * Initialise un index sur le champ `name` pour des recherches rapides.
     */
    async initializeIndex() {
      await this.localDB.createIndex({ index: { fields: ['name'] } })
    },

    /**
     * Recherche des posts par nom en utilisant une requête regex.
     */
    async searchPosts() {
      const results = await this.localDB.find({
        selector: { name: { $regex: this.searchQuery } }
      })
      this.posts = results.docs
    },

    /**
     * Ajoute une pièce jointe à un post.
     * @param postId - Identifiant du post
     * @param event - Événement contenant le fichier
     */
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

          const post = await this.localDB.get(postId)
          if (!post._rev) {
            console.error('Révision (_rev) manquante pour le post')
            return
          }

          const url = URL.createObjectURL(new Blob([arrayBuffer], { type: file.type }))

          post.attachments = {
            ...(post.attachments || {}),
            [file.name]: { url, type: file.type }
          }

          await this.localDB.put(post)
          console.log('Pièce jointe ajoutée avec succès.')
          this.refreshPosts()
        } catch (error) {
          console.error('Erreur lors de l’ajout de la pièce jointe :', error)
        }
      }

      reader.onerror = (error) => {
        console.error('Erreur lors de la lecture du fichier :', error)
      }

      reader.readAsArrayBuffer(file)
    }
  },

  /**
   * Fonction exécutée au montage du composant pour charger les posts.
   */
  async mounted() {
    this.refreshPosts()
  }
}
</script>

<style scoped>
/* Conteneur principal */
.infra-don {
  font-family: 'Roboto', sans-serif;
  padding: 20px;
  background-color: #121212;
  color: #f0f0f0;
  border-radius: 10px;
}

/* Titre principal */
h1 {
  font-size: 2.5rem;
  color: #66bb6a;
  text-align: center;
}

/* Texte général */
p {
  font-size: 1rem;
  color: #c8e6c9;
  margin: 10px 0;
  text-align: center;
}

/* Boutons d'actions */
.actions,
.search {
  display: flex;
  justify-content: center;
  gap: 10px;
  margin-bottom: 20px;
}

/* Boutons globaux */
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

/* Effets au survol des boutons */
.actions button:hover,
.search button:hover {
  background-color: #4caf50;
}

/* Zone de recherche */
.search input {
  padding: 10px;
  border: 2px solid #66bb6a;
  border-radius: 5px;
  font-size: 1rem;
  color: #121212;
  outline: none;
}

/* Carte de post */
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

/* Effet au survol de la carte */
.post:hover {
  transform: scale(1.02);
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
}

/* Alignement du texte des posts */
.post p {
  margin: 5px 0;
  text-align: left;
}

/* Actions des posts */
.post-actions {
  display: flex;
  gap: 15px; /* Espacement entre les boutons */
  flex-wrap: wrap; /* Permet le retour à la ligne si nécessaire */
  margin-top: 10px;
}

/* Boutons des posts */
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

/* Effets au survol des boutons de post */
.post button:hover {
  background-color: #4caf50;
}

/* Input pour l'ajout de fichier */
input[type='file'] {
  margin-top: 10px; /* Ajout d'espace au-dessus de l'input */
  background-color: #66bb6a;
  color: #121212;
  border: none;
  border-radius: 5px;
  padding: 5px;
  cursor: pointer;
}

/* Style du bouton d'upload */
input[type='file']::-webkit-file-upload-button {
  background-color: #2c3e50;
  color: #fff;
  border: none;
  border-radius: 5px;
  padding: 5px 10px;
  cursor: pointer;
}

/* Effet au survol du bouton d'upload */
input[type='file']::-webkit-file-upload-button:hover {
  background-color: #388e3c;
}

/* Style des images */
.attachment-image {
  max-width: 100%;
  max-height: 200px;
  border-radius: 5px;
  margin-top: 10px;
}
</style>
