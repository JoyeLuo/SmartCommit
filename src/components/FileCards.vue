 /* eslint-disable no-console */

<template>
  <div>
    <!-- refreshKey is used as a trick to forcely refresh this component -->
    <div :key="refreshKey" class="card-scene">
      <div v-if="loadingStatus">
        <center>
          <b-button disabled variant="primary">
            <b-spinner small type="grow"></b-spinner>Loading...
          </b-button>
        </center>
      </div>
      <Container
        :drop-placeholder="upperDropPlaceholderOptions"
        @drop="onColumnDrop($event)"
        drag-handle-selector=".column-drag-handle"
        orientation="horizontal"
        v-else
      >
        <Draggable :key="column.id" v-for="column in scene.children">
          <div :class="column.props.className">
            <div class="card-column-header">
              <span class="column-drag-handle" title="Drag to Move" v-b-tooltip.hover>
                <v-icon name="hand-spock" />
              </span>
              {{ column.name }}
            </div>
            <div id="message-container">
              <b-input-group :id="`message-${column.id}`" class="mt-3" prepend>
                <b-form-input placeholder="Commit Message" v-model="column.message"></b-form-input>

                <b-input-group-append>
                  <!-- disable button if the message is empty with: :disabled="!column.message" -->
                  <b-button
                    @click="readyToCommit(column.id, column.message, column.children)"
                    title="Ok"
                    v-b-tooltip.hover
                    variant="outline-success"
                  >
                    <v-icon name="check" />
                  </b-button>
                  <b-button
                    @click="column.message=''"
                    title="Clear"
                    v-b-tooltip.hover
                    variant="info"
                  >
                    <v-icon name="eraser" />
                  </b-button>
                </b-input-group-append>
              </b-input-group>
              <b-popover
                :target="`message-${column.id}`"
                @hidden="onHidden"
                @show="onShow"
                @shown="onShown"
                container="message-container"
                placement="auto"
                ref="popover"
                triggers="focus"
              >
                <template slot="title">
                  <!-- <b-button @click="onClose" aria-label="Close" class="close">
                    <span aria-hidden="true" class="d-inline-block">&times;</span>
                  </b-button>-->
                  Recommended Words
                </template>
                <div class="words">
                  <b-badge pill variant="primary">Primary</b-badge>
                  <b-badge pill variant="secondary">Secondary</b-badge>
                  <b-badge pill variant="success">Success</b-badge>
                  <b-badge pill variant="danger">Danger</b-badge>
                  <b-badge pill variant="warning">Warning</b-badge>
                  <b-badge pill variant="info">Info</b-badge>
                  <b-badge pill variant="light">Light</b-badge>
                  <b-badge pill variant="dark">Dark</b-badge>
                </div>
              </b-popover>
            </div>

            <Container
              :drop-placeholder="dropPlaceholderOptions"
              :get-child-payload="getCardPayload(column.id)"
              @drop="(e) => onCardDrop(column.id, e)"
              drag-class="card-ghost"
              drop-class="card-ghost-drop"
              group-name="col"
            >
              <Draggable
                :key="card.id"
                title="Drag to Move"
                v-b-tooltip.hover
                v-for="card in column.children"
              >
                <!-- <b-tooltip :target="() => $refs['card']" placement="bottom">Drag to Move</b-tooltip> -->
                <div
                  :class="card.props.className"
                  :style="card.props.style"
                  @dblclick="showDiffWithSweet(card.path, card.abs_path, card.language)"
                  class="no-select"
                >
                  <p class="no-select" title="Double Click to Show Diff" v-b-tooltip.hover>
                    {{ card.path }}
                    <b-badge :variant="card.badgeType" pill>{{card.operation}}</b-badge>
                  </p>
                </div>
              </Draggable>
            </Container>
          </div>
        </Draggable>
      </Container>
    </div>

    <!-- dialog to confirm commit -->
    <sweet-modal ref="commitModal" title="Ready to Commit?">
      <b-card :header="commitMessage" border-variant="success">
        <b-list-group>
          <b-list-group-item :key="file.id" v-for="file in commitFiles">{{file.path}}</b-list-group-item>
        </b-list-group>
      </b-card>
      <b-button @click="reallyCommit()" class="right-button" variant="success">Commit!</b-button>
      <b-button @click="cancelCommit()" class="right-button" variant="outline-primary">Cancel</b-button>
    </sweet-modal>

    <!-- use 'hide-close-button blocking' to force user action -->
    <sweet-modal icon="success" ref="success" title="Success">{{successMessage}}</sweet-modal>
    <sweet-modal icon="warning" ref="alert" title="Alert">{{alertMessage}}</sweet-modal>
    <sweet-modal icon="error" ref="error" title="Error">{{errorMessage}}</sweet-modal>

    <!-- modal to show diff view-->
    <!-- sweet-modal-vue -->
    <sweet-modal :title="diffViewTitle" ref="diffViewModal" width="80%">
      <template slot="box-action">
        <b-badge pill variant="info">{{language}}</b-badge>&nbsp;
      </template>
      <sweet-modal-tab id="sideDiff" title="Side by Side">
        <div v-if="loadingDiff">
          <b-spinner label="Spinning" variant="success"></b-spinner>
        </div>
        <MonacoEditor
          :diffEditor="true"
          :language="language"
          :options="sideOptions"
          :original="codeRight"
          :value="codeLeft"
          class="editor"
          ref="sideDiffEditor"
          v-else
        />
      </sweet-modal-tab>
      <sweet-modal-tab id="inlineDiff" title="Inline Diff">
        <div v-if="loadingDiff">
          <b-spinner label="Spinning" variant="success"></b-spinner>
        </div>
        <MonacoEditor
          :diffEditor="true"
          :language="language"
          :options="inlineOptions"
          :original="codeRight"
          :value="codeLeft"
          class="editor"
          ref="inlineDiffEditor"
          v-else
        />
      </sweet-modal-tab>
    </sweet-modal>
  </div>
</template>

<script>
import { Container, Draggable } from 'vue-smooth-dnd'
import { applyDrag, generateItems } from './utils/helpers'
import {
  getRootPath,
  analyzeStatus,
  doCommit,
  showAtHEAD
} from './utils/gitutils'
import { SweetModal, SweetModalTab } from 'sweet-modal-vue'
import MonacoEditor from './vue-monaco'

const fs = require('fs')
var git = require('simple-git')

const columnNames = ['Lorem', 'Ipsum', 'Consectetur', 'Eiusmod']
const badgeTypeMap = new Map([
  ['Modified', 'primary'],
  ['Untracked', 'success'],
  ['Conflicted', 'danger'],
  ['Deleted', 'secondary'],
  ['Renamed', 'info']
])

const cardColors = [
  'azure',
  'beige',
  'bisque',
  'blanchedalmond',
  'burlywood',
  'cornsilk',
  'gainsboro',
  'ghostwhite',
  'ivory',
  'khaki'
]

const pickColor = () => {
  const rand = Math.floor(Math.random() * 10)
  return cardColors[rand]
}

var scene = {
  type: 'container',
  props: {
    orientation: 'horizontal'
  },
  children: []
}

export default {
  name: 'Cards',

  components: { Container, Draggable, MonacoEditor, SweetModal, SweetModalTab },

  data() {
    return {
      scene,
      refreshKey: 0,
      upperDropPlaceholderOptions: {
        className: 'cards-drop-preview',
        animationDuration: '150',
        showOnTop: true
      },
      dropPlaceholderOptions: {
        className: 'drop-preview',
        animationDuration: '150',
        showOnTop: true
      },

      REPO_PATH: '',
      // async analyzing git repo
      loadingStatus: true,
      successMessage: '',
      alertMessage: '',
      errorMessage: '',

      // diff editor options
      sideOptions: {
        // selectOnLineNumbers: true
        readOnly: true,
        renderSideBySide: true
      },
      inlineOptions: {
        // selectOnLineNumbers: true
        readOnly: true,
        renderSideBySide: false
      },

      // diff view contents
      diffViewTitle: '',

      loadingDiff: true,
      language: 'javascript',
      codeLeft: '',
      codeRight: '',

      // commit data
      columnID: -1, // to remove and refresh the successfully committed column
      commitMessage: '',
      commitFiles: []
    }
  },

  methods: {
    // handle drag&drop
    onColumnDrop(dropResult) {
      const scene = Object.assign({}, this.scene)
      scene.children = applyDrag(scene.children, dropResult)
      this.scene = scene
    },

    onCardDrop(columnId, dropResult) {
      if (dropResult.removedIndex !== null || dropResult.addedIndex !== null) {
        const scene = Object.assign({}, this.scene)
        const column = scene.children.filter(p => p.id === columnId)[0]
        const columnIndex = scene.children.indexOf(column)

        const newColumn = Object.assign({}, column)
        newColumn.children = applyDrag(newColumn.children, dropResult)
        scene.children.splice(columnIndex, 1, newColumn)

        this.scene = scene
      }
    },

    getCardPayload(columnId) {
      return index => {
        return this.scene.children.filter(p => p.id === columnId)[0].children[
          index
        ]
      }
    },

    // methods for commit message tags popover
    onClose() {
      // this.$refs.popover.$emit('close')
      // this.popoverShow = false
    },
    onOk() {},
    onShow() {
      // This is called just before the popover is shown
      // Reset our popover form variables
    },
    onShown() {
      // Called just after the popover has been shown
      // Transfer focus to the first input
    },
    onHidden() {
      // Called just after the popover has finished hiding
      // Bring focus back to the button
    },
    focusRef(ref) {
      // Some references may be a component, functional component, or plain element
      // This handles that check before focusing, assuming a `focus()` method exists
      // We do this in a double `$nextTick()` to ensure components have
      // updated & popover positioned first
      this.$nextTick(() => {
        this.$nextTick(() => {
          // ;(ref.$el || ref).focus()
        })
      })
    },

    // a trick to forcely refresh the component
    refreshCards() {
      this.refreshKey += 1
    },

    log(...params) {
      console.log(...params)
    },

    getBadgeType(operation) {
      return badgeTypeMap.get(operation)
    },

    //  prepare data by analyzing git repo
    init() {
      // console.log("Analyzing git repo "+ __dirname);
      getRootPath('')
        .then(rootPath => {
          this.REPO_PATH = rootPath + '/'
          console.log('Repo root path: ' + this.REPO_PATH)
          this.analyzeGitRepo()
        })
        .catch(err => {
          this.loadingStatus = false
          this.errorMessage = err.message
          this.$refs.error.open()
        })
    },
    filterStatus(status) {
      let res = []
      for (let k in status) {
        if (status[k].length > 0) {
          res.push(status[k])
        }
      }
      return res
    },
    analyzeGitRepo() {
      analyzeStatus(this.REPO_PATH)
        .then(status => {
          let res = this.filterStatus(status)
          this.scene = {
            type: 'container',
            props: {
              orientation: 'horizontal'
            },
            // group changes according to the operation, then generate the code
            children: generateItems(res.length, i => ({
              id: `column${i}`,
              type: 'container',
              name: `Group ${i}`,
              props: {
                orientation: 'vertical',
                className: 'card-container'
              },
              message: '',
              committed: 0, // whether the group has been committed, 0 or 1
              children: generateItems(res[i].length, j => ({
                type: 'draggable',
                id: `${i}${j}`,
                props: {
                  className: 'card',
                  style: { backgroundColor: pickColor() }
                },
                operation: res[i][j].operation,
                badgeType: this.getBadgeType(res[i][j].operation),
                path: res[i][j].path,
                abs_path: res[i][j].abs_path,
                language: res[i][j].lang
              }))
            }))
          }
          this.loadingStatus = false
          // console.log(this.scene)
          this.refreshCards()
        })
        .catch(err => {
          this.loadingStatus = false
          this.errorMessage = err.message
          this.$refs.error.open()
        })
    },

    // show diffs with alternative modal
    showDiffWithSweet(path, abs_path, language) {
      this.$refs.diffViewModal.open('sideDiff')
      this.loadingDiff = true
      this.diffViewTitle = abs_path
      fs.readFile(abs_path, 'utf-8', (err, data) => {
        if (err) {
          this.errorMessage =
            'An error ocurred reading the file :' + err.message
          this.$refs.diffViewModal.close()
          this.$refs.error.open()
          return
        }
        this.language = language
        // this.$refs.sideDiffEditor.setLanguage(language)

        this.codeRight = data

        showAtHEAD(this.REPO_PATH, path)
          .then(res => {
            this.codeLeft = res
            this.loadingDiff = false
          })
          .catch(err => {
            this.errorMessage =
              'An error ocurred reading the file :' + err.message
            this.$refs.diffViewModal.close()
            this.$refs.error.open()
          })
      })
    },

    // handle commit action
    readyToCommit(id, message, list) {
      if (list.length == 0) {
        this.alertMessage = 'No files to commit in this group!'
        this.$refs.alert.open()
      } else if (message === '') {
        this.alertMessage = 'The commit message cannot be empty!'
        this.$refs.alert.open()
      } else {
        this.commitMessage = message
        this.commitFiles = list
        this.columnID = id
        this.$refs.commitModal.open()
      }
    },
    reallyCommit() {
      let filePaths = new Array()
      console.log(this.commitFiles)
      for (let file of this.commitFiles) {
        console.log(file)
        filePaths.push(file.path)
      }
      doCommit(this.REPO_PATH, this.commitMessage, filePaths)
        .then(res => {
          this.successMessage =
            'Successfully commit ' + res.commit + ' to branch ' + res.branch
          this.$refs.commitModal.close()
          this.$refs.success.open()
          // clear data (no necessary but for safety)
          this.commitMessage = ''
          this.commitFiles = []
          // remove the committed column from scene
          this.removeColumnByID(this.columnID)
          this.refreshCards()
          console.log(this.refreshKey)
        })
        .catch(err => {
          this.errorMessage = err
          this.$refs.error.open()
        })
    },
    removeColumnByID(id) {
      const scene = Object.assign({}, this.scene)
      scene.children = []
      for (let child of this.scene.children) {
        if (child.id != id) {
          scene.children.push(child)
        }
      }
      this.scene = scene
    },
    cancelCommit() {
      this.$refs.commitModal.close()
    }
  },

  created() {
    this.init()
  }
}
</script>

<style>
.no-select {
  -webkit-touch-callout: none;
  -webkit-user-select: none;
  -khtml-user-select: none;
  -moz-user-select: none;
  -ms-user-select: none;
  user-select: none;
}

.column-drag-handle:hover {
  cursor: grab;
  cursor: -moz-grab;
  cursor: -webkit-grab;
}

.no-select:hover {
  cursor: grab;
  cursor: -moz-grab;
  cursor: -webkit-grab;
}

.no-select:active {
  cursor: grabbing;
  cursor: -moz-grabbing;
  cursor: -webkit-grabbing;
}

.words {
  cursor: pointer;
}

.editor {
  height: 800px;
}

.sweet-modal .sweet-title h2 {
  line-height: inherit;
}

.right-button {
  float: right;
  margin-top: 10px;
  margin-right: 10px;
}
</style>