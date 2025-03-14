<template>
  <div>
    <portal to="toolbar">
      <div v-if="stream" class="d-flex align-center">
        <div class="text-truncate">
          <router-link
            v-tooltip="stream.name"
            class="text-decoration-none space-grotesk mx-1"
            :to="`/streams/${stream.id}`"
          >
            <v-icon small class="primary--text mb-1 mr-1">mdi-folder</v-icon>
            <b>{{ stream.name }}</b>
          </router-link>
        </div>
        <div class="text-truncate flex-shrink-0">
          /
          <v-icon small class="mr-1 mb-1 hidden-xs-only">mdi-arrow-up</v-icon>
          <span class="space-grotesk">Import File - Alpha Feature</span>
        </div>
      </div>
    </portal>
    <v-row>
      <v-col cols="12">
        <section-card>
          <v-card-text>
            Speckle can now process files and store them as a commit (snapshot). You can
            then access it from the Speckle API, and receive it in other applications.
            Current supported formats are: IFC, STL and OBJ. Thanks to the Open Source
            <a
              href="https://ifcjs.github.io/info/docs/Guide/web-ifc/Introduction"
              target="_blank"
            >
              IFC.js Project
            </a>
            for making this possible.
          </v-card-text>
        </section-card>
        <v-alert
          v-if="stream && (stream.role === 'stream:reviewer' || !stream.role)"
          type="warning"
        >
          Your permission level ({{ stream.role ? stream.role : 'none' }}) is not high
          enough to access this feature.
        </v-alert>
      </v-col>
      <v-col cols="12" md="4">
        <section-card>
          <template slot="header">Previous Uploads</template>
          <v-card-text>
            Here are the previously uploaded files in this stream. Please note,
            currently processing time is restricted to 5 minutes - if a file takes
            longer to process, it will be ignored.
          </v-card-text>
          <div v-if="!$apollo.loading && streamUploads.length !== 0">
            <template v-for="file in streamUploads">
              <file-processing-item :key="file.id" :file-id="file.id" />
            </template>
          </div>
          <div v-else>
            <v-card-text>No uploads.</v-card-text>
          </div>
        </section-card>
      </v-col>
      <v-col cols="12" md="8">
        <div v-if="stream && !(stream.role === 'stream:reviewer' || !stream.role)">
          <v-card
            elevation="0"
            style="height: 420px; transition: all 0.2s ease"
            :class="` mb-4 d-flex justify-center
        ${dragover && !$vuetify.theme.dark ? 'grey lighten-4' : ''}
        ${dragover && $vuetify.theme.dark ? 'grey darken-4' : ''}
        `"
            @drop.prevent="onFileDrop($event)"
            @dragover.prevent="dragover = true"
            @dragenter.prevent="dragover = true"
            @dragleave.prevent="dragover = false"
          >
            <div v-if="!dragError" class="align-self-center text-center">
              <input
                id="myid"
                type="file"
                accept=".ifc,.IFC,.stl,.STL,.obj,.OBJ,.mtl,.MTL"
                style="display: none"
                multiple
                @change="onFileSelect($event)"
              />
              <v-icon
                x-large
                color="primary"
                :class="`hover-tada ${dragover ? 'tada' : ''}`"
                style="cursor: pointer"
                onclick="document.getElementById('myid').click()"
              >
                mdi-cloud-upload
              </v-icon>
              <br />
              <span class="primary--text">Drag and drop your file here!</span>
              <br />
              <span class="caption">
                Maximum 5 files at a time. Size is restricted to 50mb each.
              </span>
            </div>
            <v-alert
              v-if="dragError"
              dismissible
              class="align-self-center text-center"
              type="error"
              @click="dragError = null"
            >
              {{ dragError }}
            </v-alert>
          </v-card>

          <!-- {{ uploads }} -->
          <template v-for="file in files">
            <file-upload-item
              :key="file.fileName"
              :file="file"
              :branches="stream.branches.items"
              @done="uploadCompleted"
            />
          </template>
        </div>
      </v-col>
    </v-row>
  </div>
</template>

<script>
import gql from 'graphql-tag'

export default {
  name: 'TheUploads',
  components: {
    FileUploadItem: () => import('@/main/components/stream/uploads/FileUploadItem'),
    FileProcessingItem: () =>
      import('@/main/components/stream/uploads/FileProcessingItem'),
    SectionCard: () => import('@/main/components/common/SectionCard')
  },
  apollo: {
    stream: {
      query: gql`
        query stream($id: String!) {
          stream(id: $id) {
            id
            role
            name
            branches {
              totalCount
              items {
                name
              }
            }
          }
        }
      `,
      variables() {
        return { id: this.$route.params.streamId }
      }
    },
    streamUploads: {
      query: gql`
        query streamUploads($streamId: String!) {
          stream(id: $streamId) {
            id
            role
            fileUploads {
              id
            }
          }
        }
      `,
      update: (data) => data.stream.fileUploads,
      variables() {
        return {
          streamId: this.$route.params.streamId
        }
      }
    }
  },
  data() {
    return {
      dragover: false,
      loading: false,
      stream: null,
      files: [],
      showUploadDialog: false,
      error: null,
      dragError: null
    }
  },
  computed: {},
  methods: {
    onFileSelect(e) {
      this.parseFiles(e.target.files)
    },
    onFileDrop(e) {
      this.parseFiles(e.dataTransfer.files)
    },
    parseFiles(files) {
      this.dragover = false
      this.dragError = null
      for (const file of files) {
        const extension = file.name.split('.')[1]
        if (
          !extension ||
          !['ifc', 'stl', 'obj', 'mtl'].includes(extension.toLowerCase())
        ) {
          this.dragError = `The ${extension.toLowerCase()} file extension is not yet supported`
          return
        }

        if (file.size > 104857600) {
          this.dragError =
            'Your files are too powerful (for now). Maximum upload size is 100mb!'
          return
        }

        if (this.files.findIndex((f) => f.name === file.name) !== -1) {
          this.dragError = 'This file is already primed for upload.'
          return
        }
      }
      if (files.length > 5) {
        this.dragError = 'Maximum five files at a time allowed.'
        return
      }
      this.dragError = null

      for (const file of files) {
        this.files.push(file)
      }
    },
    uploadCompleted(file) {
      const index = this.files.findIndex((f) => f.name === file)
      this.files.splice(index, 1)
      this.$apollo.queries.streamUploads.refetch()
      this.$mixpanel.track('File Action', {
        type: 'action',
        name: 'upload',
        count: this.files.length
      })
    }
  }
}
</script>
