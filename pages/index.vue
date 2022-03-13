<template>
  <v-container>
    <v-dialog v-model="dialog" max-width="355" persistent>
      <v-container class="d-block">
        <v-row no-gutters align="center" justify="space-between">
          <v-row no-gutters>
            <h3>Add Board</h3>
          </v-row>
          <v-icon @click="dialog = false">mdi-close</v-icon>
        </v-row>
        <v-form ref="form" v-model="valid">
          <div class="d-flex flex-column">
            <v-text-field
              label="Board title"
              name="title"
              type="text"
              :rules="[(v) => !!v || 'Board title is required']"
              required
              v-model="board.title"
            ></v-text-field>
            <br />
            <v-btn :disabled="!valid" color="primary" @click="createBoard"
              >Submit</v-btn
            >
          </div>
        </v-form>
      </v-container>
    </v-dialog>
    <div class="d-flex flex-row align-center justify-space-between mt-6">
      <h1>My Boards</h1>
      <v-btn small depressed @click="addBoard">ADD BOARD</v-btn>
    </div>
    <div class="d-flex flex-wrap align-center justify-start pt-8">
      <p v-if="boards.length === 0">You have no boards yet.</p>
      <v-card
        @click="$router.push('/boards/' + board.id)"
        class="mx-4"
        v-for="board in boards"
        v-bind:key="board.id"
      >
        <v-card-title>
          {{ board.title }}
        </v-card-title>
        <v-card-subtitle>
          created {{ board.dateCreated | formatDate }}
        </v-card-subtitle>
      </v-card>
    </div>

    <v-snackbar
      :timeout="3000"
      v-model="snackbar"
      absolute
      bottom
      color="primary"
    >
      {{ snackbarText }}
    </v-snackbar>
  </v-container>
</template>

<script>
import { v4 as uuidv4 } from 'uuid'
export default {
  async asyncData() {
    let boardsRef = $nuxt.$fire.firestore.collection('boards')
    let boardData = []
    await boardsRef
      .get()
      .then(function (querySnapshot) {
        if (querySnapshot.docs.length > 0) {
          try {
            for (const doc of querySnapshot.docs) {
              let data = doc.data()
              data.id = doc.id
              boardData.push(data)
            }
          } catch (err) {}
        }
      })
      .catch(function (error) {})
    return { boards: boardData }
  },
  data() {
    return {
      enableColor: false,
      dialog: false,
      valid: false,
      board: {
        title: '',
      },
      snackbar: false,
      snackbarText: 'No error message',
      currentImageId: '',
    }
  },
  created() {},
  methods: {
    addBoard() {
      this.currentImageId = uuidv4()
      this.dialog = true
    },
    createBoard() {
      let that = this
      if (this.$refs.form.validate()) {
        this.board.dateCreated = Date.now()
        this.$fire.firestore
          .collection('boards')
          .doc(this.currentImageId)
          .set(this.board)
          .then(function (docRef) {
            that.dialog = false
            that.$refs.form.reset()
            that.snackbarText = 'Successfully created your board'
            that.snackbar = true
          })
          .catch(function (error) {})
      }
    },
  },
}
</script>
<style lang="scss">
.v-dialog {
  border-radius: 15px;
  background-color: white;
  padding: 15px;
}
</style>
