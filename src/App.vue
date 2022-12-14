<template>
  <div class="flex w-screen h-screen text-gray-700">
    <div class="flex flex-col flex-shrink-0 w-64 border-r border-gray-300 bg-gray-100">
      <!-- Sidebar -->
      <div class="h-0 overflow-auto flex-grow">
        <div class="flex items-center h-8 px-3 group mt-4">
          <div class="flex items-center flex-grow focus:outline-none">
            <a
              class="ml-2 leading-none font-medium text-sm hover:text-gray-900"
              href="#"
              @click.prevent="showAllNotes"
              >All Notes</a
            >
          </div>
          <button class="add-note" @click="addNewNote">
            <svg
              class="h-5 w-5"
              xmlns="http://www.w3.org/2000/svg"
              fill="none"
              viewBox="0 0 24 24"
              stroke="currentColor"
            >
              <path
                stroke-linecap="round"
                stroke-linejoin="round"
                stroke-width="2"
                d="M12 6v6m0 0v6m0-6h6m-6 0H6"
              />
            </svg>
          </button>
          <a class="flex items-center h-8 text-sm pl-8 pr-3"
          href="#"
          v-for="note in notes"
          :key="note.created"
          @click.prevent="openNote(note)"
          >

          <span class="ml-2 leading-none">{{ new Date(note.created).toLocaleDateString()}} </span>
          </a>
        </div>
      </div>

    </div>

    <div class="flex flex-col flex-grow" v-if="Object.keys(activeNote).length">
      <!-- Main content - Editor-->

      <div class="flex flex-col flex-grow overflow-auto">
        <editor-content :editor="editor" />
      </div>
      <div class="h-16 bg-gray-100 border-t border-gray-300 text-right">
        <button @click="saveNote">Save Note</button>
      </div>

    </div>
    <div class="flex flex-col flex-grow" v-else>
      <!-- Main content - Notes List -->
      <div class="flex flex-col flex-grow overflow-auto">
        <div v-for="note in notes" :key="note.created">
          <div class="flex px-4 pt-3 pb-4">
            <div class="prose my-2 mx-auto">
              <div>
                <span class="ml-1 text-xs text-gray-500">Created on {{ new Date(note.created).toLocaleString() }}</span>
              </div>
              <div v-html="note.content"></div>
            </div>
          </div>
          <hr class="w-full">
        </div>
      </div>
    </div>
  </div>

</template>

<script>
import { Editor, EditorContent } from '@tiptap/vue-3'
import StarterKit from '@tiptap/starter-kit'


export default {
  name: 'App',

  components: {
    EditorContent,
  },

  data() {
    return {
      editor: null,
      database: null,
      notes: [],
      activeNote: {}
    }
  },

  async created() {
    this.database = await this.getDatabase();
    const notes = await this.getNotes();
    this.notes = notes.reverse();
  },

  mounted() {
    this.editor = new Editor({
      content: '',
      extensions: [
        StarterKit,
      ],
      editorProps: {
        attributes: {
          class: "prose my-6 mx-auto focus:outline-none"
        }
      }
    })
  },

  beforeUnmount() {
    this.editor.destroy()
  },

  methods: {
    async getDatabase() {
      return new Promise((resolve, reject) => {
        const db = window.indexedDB.open('notes', 2);

        db.onerror = e => {
          reject('Error opening the database');
        }
        db.onsuccess = e => {
          console.log('db.onsuccess', e);
          resolve(e.target.result);
        }
        db.onupgradeneeded = e => {
          console.log('db.onupgradeneeded', e);
          e.target.result.deleteObjectStore('notes');
          e.target.result.createObjectStore('notes', {keyPath: 'created' });
        }
      })
    },

    async saveNote() {
      return new Promise((resolve, reject) => {
        let noteStore = this.database.transaction('notes', 'readwrite')
          .objectStore('notes');

        let noteRequest = noteStore.get(this.activeNote.created);

        noteRequest.onerror = e => {
          reject('Error saving the note in the database');
        }

        noteRequest.onsuccess = e => {
          let note = e.target.result;
          note.content = this.editor.getHTML();

          let updateRequest = noteStore.put(note);

          updateRequest.onerror = e => {
            reject('Error updating the note');
          }

          updateRequest.onsuccess = e => {
            let noteIndex = this.notes.findIndex(n => n.created === note.created);
            this.notes[noteIndex] = note;
            resolve();
          }
        }

      })
    },

    async getNotes() {
      return new Promise((resolve, reject) => {
        this.database.transaction('notes')
          .objectStore('notes')
          .getAll()
          .onsuccess = e => {
            console.log('getNotes()', e.target.result);
            resolve(e.target.result);
          }

      })
    },

    openNote(note) {
      this.editor.commands.setContent(note.content);
      this.activeNote = note;
    },

    showAllNotes() {
      this.editor.commands.clearContent();
      this.activeNote = {};
    },

    addNewNote() {
      return new Promise((resolve, reject) => {
        const transaction = this.database.transaction('notes', 'readwrite');
        transaction.oncomplete = e => {
          resolve();
        }

        const now = new Date();
        const note = {
          content: '',
          created: now.getTime()
        }

        this.notes.unshift(note);

        this.activeNote = note;
        transaction.objectStore('notes').add(note);

      })
    }

  }

}
</script>

<style scoped lang="postcss">
button {
  @apply bg-none border border-gray-900 rounded py-1 px-4 mr-4 mt-3 hover:bg-gray-900 hover:text-white;
}
</style>
