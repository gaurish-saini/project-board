<template>
  <div class="d-flex flex-column board mt-6">
    <div class="d-flex flex-row justify-space-between">
      <h1>{{ board.title }}</h1>
      <v-icon large @click="deleteBoard()">mdi-delete-outline</v-icon>
    </div>
    <small>Created On : {{ board.dateCreated | formatDate }}</small>
    <div class="d-flex flex-row pr-6 pt-8">
      <div
        v-for="list in board.lists"
        @drop="drop($event, list.id)"
        @dragover="allowDrop($event)"
        class="d-flex flex-column pt-3 mr-6 list"
        v-bind:key="list.id"
      >
        <div class="d-flex flex-row justify-space-between">
          <div class="d-flex flex-row">
            <h4 class="purple darken-1 rounded-lg text-center">
              <span class="px-2 white--text">{{ list.title }} </span>
            </h4>
            <h4 class="text--secondary font-weight-bold ps-2">
              {{ list.cards.length }}
            </h4>
          </div>
          <v-icon small @click="deleteList(list.id)">mdi-delete-outline</v-icon>
        </div>

        <!--display cards-->
        <v-card
          v-for="card in list.cards"
          class="mt-2"
          @click="editCard(card)"
          v-bind:key="card.id"
        >
          <div
            :draggable="true"
            @dragover.prevent
            @dragstart="drag($event, card)"
          >
            <v-card-text> {{ card.title }} </v-card-text>
          </div>
        </v-card>
        <div class="d-flex flex-row align-baseline">
          <v-icon
            small
            @click="
              dialogCard = true
              listId = list.id
            "
            class="mt-4"
            >mdi-plus
          </v-icon>
          <h4 class="ps-1 text--secondary font-weight-bold">New</h4>
        </div>
      </div>
      <v-dialog v-model="dialogCard" persistent max-width="600px">
        <v-card elevation="0">
          <v-card-title>
            <span class="headline">Card name</span>
          </v-card-title>
          <v-card-text>
            <v-container>
              <v-row>
                <v-col cols="12">
                  <v-text-field
                    label="Stuff to do"
                    v-model="card.title"
                    required
                  ></v-text-field>
                </v-col>
              </v-row>
            </v-container>
          </v-card-text>
          <v-card-actions>
            <v-spacer></v-spacer>
            <v-btn color="blue darken-1" text @click="dialogCard = false">
              Close
            </v-btn>
            <v-btn color="blue darken-1" text @click="createCard()">
              Save
            </v-btn>
          </v-card-actions>
        </v-card>
      </v-dialog>
      <div class="d-flex flex-row">
        <v-btn depressed @click="dialog = true" class="create-list"
          >Create new Status</v-btn
        >
        <v-dialog v-model="dialog" persistent max-width="600px">
          <v-card elevation="0">
            <v-card-title>
              <span class="headline">List name</span>
            </v-card-title>
            <v-card-text>
              <v-container>
                <v-row>
                  <v-col cols="12">
                    <v-text-field
                      label="Stuff to do"
                      v-model="list.title"
                      required
                    ></v-text-field>
                  </v-col>
                </v-row>
              </v-container>
            </v-card-text>
            <v-card-actions>
              <v-spacer></v-spacer>
              <v-btn color="blue darken-1" text @click="dialog = false">
                Close
              </v-btn>
              <v-btn color="blue darken-1" text @click="createList()">
                Save
              </v-btn>
            </v-card-actions>
          </v-card>
        </v-dialog>
      </div>
      <v-dialog v-model="dialogEditCard" persistent max-width="600px">
        <v-card elevation="0">
          <v-card-title>
            <span class="headline">{{ currentCard.title }}</span>
          </v-card-title>
          <v-card-text>
            <v-container>
              <v-row>
                <v-col cols="12">
                  <v-text-field
                    label="Edit title"
                    v-model="currentCard.title"
                    required
                  ></v-text-field>
                </v-col>
              </v-row>
            </v-container>
          </v-card-text>
          <v-card-actions>
            <v-spacer></v-spacer>
            <v-btn color="red darken-1" text @click="deleteCard()">
              Delete
            </v-btn>
            <v-btn color="blue darken-1" text @click="dialogEditCard = false">
              Close
            </v-btn>
            <v-btn color="blue darken-1" text @click="updateCard()">
              Save
            </v-btn>
          </v-card-actions>
        </v-card>
      </v-dialog>
    </div>
  </div>
</template>

<script>
import { v4 as uuidv4 } from 'uuid'
export default {
  layout: 'board',
  data() {
    return {
      listId: '',
      list: {
        title: '',
      },
      card: {
        title: '',
      },
      currentCard: {},
      cardDraggedId: '',
      cardDraggedListId: '',
      dialog: false,
      dialogCard: false,
      dialogEditCard: false,
    }
  },
  async asyncData({ params }) {
    let boardRef = $nuxt.$fire.firestore.collection('boards').doc(params.id)
    let boardData = {}
    await boardRef
      .get()
      .then(function (doc) {
        if (doc.exists) {
          boardData = doc.data()
          boardData.id = params.id
        }
      })
      .catch(function (error) {})
    return { board: boardData }
  },
  created() {
    let that = this
    let tempId = this.board.id
    let boardRef = $nuxt.$fire.firestore
      .collection('boards')
      .doc(tempId)
      .onSnapshot((doc) => {
        if (doc.exists) {
          that.board = doc.data()
          that.board.id = tempId
        }
      })
  },
  methods: {
    async createList() {
      let that = this
      that.dialog = false
      if (that.list.title != '') {
        that.list.id = uuidv4()
        that.list.cards = []
        that.list.dateCreated = Date.now()
        if (that.board.lists) {
          that.board.lists.push(that.list)
        } else {
          that.board.lists = []
          that.board.lists.push(that.list)
        }
        await that.updateBoard()
        that.list = {}
      }
    },
    async updateCardList(newlistId) {
      let that = this

      let tempListIndex = -1
      let tempCardIndex = -1
      let newListIndex = -1
      let tempListCount = 0
      let tempCardCount = 0

      for (const list of that.board.lists) {
        if (list.id === newlistId) {
          newListIndex = tempListCount
        }
        if (that.currentCard.listId === list.id) {
          tempListIndex = tempListCount
          for (const card of list.cards) {
            if (card.id === that.currentCard.id) {
              tempCardIndex = tempCardCount
            }
            tempCardCount++
          }
        }
        tempListCount++
      }

      that.board.lists[tempListIndex].cards.splice(tempCardIndex, 1)
      that.currentCard.listId = newlistId
      that.board.lists[newListIndex].cards.push(that.currentCard)

      await that.updateBoard()
    },
    allowDrop(ev) {
      ev.preventDefault()
    },
    drag(ev, card) {
      this.currentCard = card
    },
    drop(ev, listId) {
      ev.preventDefault()
      this.updateCardList(listId)
    },
    async deleteList(listId) {
      let that = this
      let index = -1
      let count = 0
      for (const list of that.board.lists) {
        if (list.id == listId) {
          index = count
        }
        count++
      }
      if (index > -1) {
        that.board.lists.splice(index, 1)
        await that.updateBoard()
      }
    },
    async createCard() {
      let that = this
      that.dialogCard = false
      if (that.card.title != '') {
        that.card.id = uuidv4()
        that.card.dateCreated = Date.now()
        that.card.listId = that.listId
        if (that.board.lists) {
          let index = -1
          let count = 0
          for (const list of that.board.lists) {
            if (list.id === that.listId) {
              index = count
            }
            count++
          }
          if (index != -1) {
            if (that.board.lists[index].cards) {
              that.board.lists[index].cards.push(that.card)
            } else {
              that.board.lists[index].cards = []
              that.board.lists[index].cards.push(that.card)
            }
          }
        }
        await that.updateBoard()
        that.card = {}
        that.listId = ''
      }
    },
    editCard(card) {
      this.dialogEditCard = true
      this.currentCard = card
    },
    async updateCard() {
      let that = this
      that.dialogEditCard = false
      for (const list of that.board.lists) {
        if (that.currentCard.listId === list.id) {
          for (const card of list.cards) {
            if (card.id === that.currentCard.id) {
              card = that.currentCard
            }
          }
        }
      }
      await that.updateBoard()
    },
    async deleteCard() {
      let that = this
      that.dialogEditCard = false
      let i = 0
      let j = 0
      let index = {
        list: -1,
        card: -1,
      }
      for (const list of that.board.lists) {
        if (that.currentCard.listId === list.id) {
          for (const card of list.cards) {
            if (card.id === that.currentCard.id) {
              index.list = i
              index.card = j
            }
            j++
          }
        }
        i++
      }
      if (index.list > -1) {
        that.board.lists[index.list].cards.splice(index.card, 1)
        await that.updateBoard()
      }
    },
    async deleteBoard() {
      let that = this
      try {
        await that.$fire.firestore
          .collection('boards')
          .doc(that.board.id)
          .delete()
          .then(() => {
            $nuxt.$router.push('/')
          })
          .catch(() => {})
      } catch (error) {
        $nuxt.$router.push('/')
      }
    },
    async updateBoard() {
      let that = this
      await that.$fire.firestore
        .collection('boards')
        .doc(that.board.id)
        .update(that.board, { merge: true })
    },
  },
}
</script>

<style lang="scss" scoped>
.board {
  padding: 12px;
  height: 100vh;
  .list {
    min-width: 250px;
    background-color: rgb(228 228 228 / 35%);
    padding: 25px;
    border-radius: 12px;
    min-height: 70vh;
  }
  .create-list {
    background-color: rgb(228 228 228 / 35%);
  }
  a {
    text-decoration: none;
  }
  .v-card__text {
    padding: 8px !important;
  }
}
</style>
