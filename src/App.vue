<template>
  <div id="app">
    <v-app theme="dark">
      <v-app-bar app>
        <div
          class="font-weight-bold site-title text-truncate"
          title="Community Sound Library"
        >
          Community Sound Library
        </div>
        <v-dialog v-model="showInfo" scrollable max-width="512px">
          <template v-slot:activator="{ on, attrs }">
            <v-btn
              class="show-info-btn"
              dark
              x-small
              fab
              color="green"
              v-bind="attrs"
              v-on="on"
              ><v-icon>mdi-help</v-icon></v-btn
            >
          </template>
          <BlipIntroduction @close="showInfo = false" />
        </v-dialog>

        <v-spacer />
        <v-text-field
          class="search-bar"
          v-model="search"
          append-icon="mdi-magnify"
          label="Search for a sound..."
          single-line
          hide-details
        ></v-text-field>
        <v-spacer />
        <v-btn
          class="upload"
          small
          color="green"
          dark
          href="https://forms.gle/LC1xt1YiVccrZxSe7"
          target="_blank"
        >
          <span class="upload-inner">Upload</span>
          <v-icon right>mdi-cloud-upload</v-icon>
        </v-btn>
      </v-app-bar>

      <v-main style="padding-top: 5px;">
        <v-container fluid>
          <v-card>
            <v-data-table
              :headers="headers"
              :items="sheetData"
              :search="search"
              :expanded="expanded"
              item-key="link"
              single-expand
              disable-pagination
              hide-default-footer
            >
              <template v-slot:item.author="{ item }">
                <!-- :set attr is a hack to set local template variable -->
                <div
                  class="text-truncate field-author"
                  :title="item.author"
                  :set="(authorWebsite = getAuthorWebsite(item.author))"
                >
                  <template v-if="authorWebsite">
                    <a :href="authorWebsite" target="_blank">{{
                      item.author
                    }}</a>
                  </template>
                  <template v-else>{{ item.author }}</template>
                </div>
              </template>
              <template v-slot:item.filename="{ item }">
                <div
                  class="text-truncate field-filename"
                  :title="item.filename"
                >
                  {{ item.filename }}
                </div>
              </template>
              <template v-slot:item.tags="{ item }">
                <div class="field-tags" :title="item.tags">{{ item.tags }}</div>
              </template>
              <template v-slot:expanded-item="{ item, headers }">
                <td :colspan="headers.length">
                  <iframe
                    :src="createIframeSrc(item.link)"
                    width="100%"
                    height="120"
                    frameborder="0"
                  />
                </td>
              </template>
              <template v-slot:item.actions="{ item }">
                <div style="margin: 4px 0;" data-iframe-height>
                  <v-btn
                    style="margin: 2px;"
                    rounded
                    small
                    color="primary"
                    :href="createDownloadLink(item.link)"
                    target="_blank"
                    >Download
                    <v-icon right>mdi-cloud-download</v-icon>
                  </v-btn>
                  <v-btn
                    style="margin: 2px;"
                    rounded
                    small
                    my-2
                    color="secondary"
                    @click="expandItem(item)"
                    >Preview
                    <v-icon right>mdi-volume-high</v-icon>
                  </v-btn>
                </div>
              </template>
            </v-data-table>
          </v-card>
        </v-container>
      </v-main>
    </v-app>
  </div>
</template>

<script>
import BlipIntroduction from './components/BlipIntroduction'

export default {
  name: 'App',
  components: {
    BlipIntroduction
  },
  data() {
    return {
      sheetDataRaw: null,
      loadError: false,
      showInfo: false,
      search: '',
      expanded: [],
      headers: [
        {
          text: 'Author',
          align: 'start',
          value: 'author'
        },
        { text: 'Filename', value: 'filename' },
        { text: 'Tags', value: 'tags' },
        { text: 'Info', value: 'info' },
        { text: 'Actions', value: 'actions', sortable: false, align: 'end' }
      ]
    }
  },
  computed: {
    sheetData() {
      if (!this.sheetDataRaw) return []
      return this.sheetDataRaw.feed.entry.map((entry) => {
        return {
          author: entry.gsx$author.$t,
          filename: entry.gsx$filename.$t,
          link: entry.gsx$link.$t,
          info: entry.gsx$otherinfo.$t,
          tags: entry.gsx$tags.$t,
          website: entry.gsx$website.$t
        }
      })
    },
    authorWebsites() {
      let authorWebsitesTable = {}
      if (this.sheetData.length) {
        this.sheetData.forEach((item) => {
          if (item.author && item.website && item.website.trim() !== '') {
            if (!authorWebsitesTable[item.author]) {
              let websiteUrl = item.website
              if (websiteUrl.search(/^http[s]?:\/\//) === -1) {
                websiteUrl = 'https://' + websiteUrl
              }
              authorWebsitesTable[item.author] = websiteUrl
            }
          }
        })
      }
      return authorWebsitesTable
    }
  },
  mounted() {
    let gSheetId = '1fp6ImVu1je8F-PWgvVY-aKT2utQ57nrqHamd7R-bcBM'
    let gSheetIndex = 3
    let gSheetJson = `https://spreadsheets.google.com/feeds/list/${gSheetId}/${gSheetIndex}/public/full?alt=json`

    if (!this.$cookies.isKey('has-seen-info')) {
      this.showInfo = true
      this.$cookies.set('has-seen-info', true, Infinity)
    }

    fetch(gSheetJson)
      .then((response) => {
        if (response.status !== 200) {
          console.log(
            'Looks like there was a problem. Status Code: ' + response.status
          )
          return
        }
        response.json().then((data) => {
          data.feed.entry.reverse()
          this.sheetDataRaw = data
        })
      })
      .catch((err) => {
        console.log('Fetch error', err)
        this.loadError = true
      })
  },
  methods: {
    getAuthorWebsite(author) {
      return this.authorWebsites[author] || ''
    },
    expandItem(item) {
      this.expanded = [item]
    },
    createDownloadLink(link) {
      try {
        var fileId = link.split('=')[1]
        return `https://drive.google.com/uc?export=download&id=${fileId}`
      } catch (error) {
        // Shh
      }
    },
    createIframeSrc(link) {
      try {
        var fileId = link.split('=')[1]
        return `https://drive.google.com/file/d/${fileId}/preview`
      } catch (error) {
        // Shh
      }
    }
  }
}
</script>

<style>
#app {
  font-family: 'Roboto', Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  color: #2c3e50;
  margin-top: 60px;
}

.field-author {
  max-width: 125px;
}

.field-filename {
  max-width: 175px;
}

.field-tags {
  min-width: 230px;
}

.show-info-btn {
  margin: 0 1em;
}

@media (max-width: 550px) {
  .upload.upload.upload {
    padding: 0 5px;
    margin-left: 1em;
    min-width: 0;
  }

  .upload.upload.upload i {
    margin: 0;
  }

  .upload-inner {
    display: none;
  }
}

@media (max-width: 710px) {
  .site-title {
    font-size: 14px;
  }

  .search-bar {
    max-width: 150px;
  }
}
</style>
