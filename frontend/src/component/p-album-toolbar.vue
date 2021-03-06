<template>
  <v-form lazy-validation dense
          ref="form" autocomplete="off" class="p-photo-toolbar p-album-toolbar" accept-charset="UTF-8"
          @submit.prevent="filterChange">
    <v-toolbar flat color="secondary">
      <v-edit-dialog
              :return-value.sync="album.Title"
              lazy
              @save="updateAlbum()"
              class="p-inline-edit">
        <v-toolbar-title>
          {{ album.Title }}
        </v-toolbar-title>
        <template v-slot:input>
          <v-text-field
                  v-model="album.Title"
                  :rules="[titleRule]"
                  :label="labels.title"
                  color="secondary-dark"
                  single-line
                  autofocus
                  class="input-title"
          ></v-text-field>
        </template>
      </v-edit-dialog>

      <v-spacer></v-spacer>

      <v-btn icon @click.stop="refresh" class="hidden-xs-only action-reload">
        <v-icon>refresh</v-icon>
      </v-btn>

      <v-btn icon @click.stop="dialog.share = true" v-if="$config.feature('share')">
        <v-icon>share</v-icon>
      </v-btn>

      <v-btn icon v-if="settings.view === 'cards'" @click.stop="setView('list')">
        <v-icon>view_list</v-icon>
      </v-btn>
      <v-btn icon v-else-if="settings.view === 'list'" @click.stop="setView('mosaic')">
        <v-icon>view_comfy</v-icon>
      </v-btn>
      <v-btn icon v-else @click.stop="setView('cards')">
        <v-icon>view_column</v-icon>
      </v-btn>

      <v-btn icon @click.stop="showUpload()" v-if="!$config.values.readonly && $config.feature('upload')"
             class="hidden-sm-and-down">
        <v-icon>cloud_upload</v-icon>
      </v-btn>

      <v-btn icon @click.stop="expand" class="p-expand-search">
        <v-icon>{{ searchExpanded ? 'keyboard_arrow_up' : 'keyboard_arrow_down' }}</v-icon>
      </v-btn>
    </v-toolbar>

    <v-card class="pt-1"
            flat
            color="secondary-light"
            v-show="searchExpanded">
      <v-card-text>
        <v-layout row wrap>
          <v-flex xs12 sm6 md3 pa-2 class="p-countries-select">
            <v-combobox flat solo hide-details color="secondary-dark"
                        v-model="album.Category"
                        :items="categories"
                        :label="labels.category"
                        @change="updateAlbum"
                        :allow-overflow="false"
                        return-masked-value
                        class="input-category"
            ></v-combobox>
          </v-flex>
          <v-flex xs12 sm6 md3 pa-2 class="p-camera-select">
            <v-select @change="dropdownChange"
                      :label="labels.camera"
                      flat solo hide-details
                      color="secondary-dark"
                      item-value="ID"
                      item-text="Name"
                      v-model="filter.camera"
                      :items="options.cameras">
            </v-select>
          </v-flex>
          <v-flex xs12 sm6 md3 pa-2 class="p-view-select">
            <v-select @change="dropdownChange"
                      :label="labels.view"
                      flat solo hide-details
                      color="secondary-dark"
                      v-model="settings.view"
                      :items="options.views"
                      id="viewSelect">
            </v-select>
          </v-flex>
          <v-flex xs12 sm6 md3 pa-2 class="p-time-select">
            <v-select @change="dropdownChange"
                      :label="labels.sort"
                      flat solo hide-details
                      color="secondary-dark"
                      v-model="filter.order"
                      :items="options.sorting">
            </v-select>
          </v-flex>
          <v-flex xs12 pa-2>
            <v-textarea flat solo auto-grow
                        browser-autocomplete="off"
                        :label="labels.description"
                        :rows="2"
                        :key="growDesc"
                        color="secondary-dark"
                        style="background-color: white"
                        v-model="album.Description"
                        @change="updateAlbum"
                        class="input-description"
            >
            </v-textarea>
          </v-flex>
        </v-layout>
      </v-card-text>
    </v-card>
    <p-share-dialog :show="dialog.share" :model="album" @upload="webdavUpload"
                    @close="dialog.share = false"></p-share-dialog>
    <p-share-upload-dialog :show="dialog.upload" :selection="[album.getId()]" @cancel="dialog.upload = false"
                           @confirm="dialog.upload = false"></p-share-upload-dialog>
  </v-form>
</template>
<script>
    import Event from "pubsub-js";

    export default {
        name: 'p-album-toolbar',
        props: {
            album: Object,
            filter: Object,
            settings: Object,
            refresh: Function,
            filterChange: Function,
        },
        data() {
            const cameras = [{
                ID: 0,
                Name: this.$gettext('All Cameras')
            }].concat(this.$config.get('cameras'));
            const countries = [{
                ID: '',
                Name: this.$gettext('All Countries')
            }].concat(this.$config.get('countries'));

            const configValues = this.$config.values;

            return {
                experimental: this.$config.get("experimental"),
                isFullScreen: !!document.fullscreenElement,
                categories: configValues.albumCategories ? configValues.albumCategories : [],
                searchExpanded: false,
                options: {
                    'views': [
                        {value: 'mosaic', text: this.$gettext('Mosaic')},
                        {value: 'cards', text: this.$gettext('Cards')},
                        {value: 'list', text: this.$gettext('List')},
                    ],
                    'countries': countries,
                    'cameras': cameras,
                    'sorting': [
                        {value: 'imported', text: this.$gettext('Recently added')},
                        {value: 'newest', text: this.$gettext('Newest first')},
                        {value: 'oldest', text: this.$gettext('Oldest first')},
                        {value: 'name', text: this.$gettext('Sort by file name')},
                        {value: 'similar', text: this.$gettext('Group by similarity')},
                        {value: 'relevance', text: this.$gettext('Most relevant')},
                    ],
                },
                dialog: {
                    share: false,
                    upload: false,
                },
                labels: {
                    title: this.$gettext("Album Name"),
                    description: this.$gettext("Description"),
                    search: this.$gettext("Search"),
                    view: this.$gettext("View"),
                    country: this.$gettext("Country"),
                    camera: this.$gettext("Camera"),
                    sort: this.$gettext("Sort By"),
                    category: this.$gettext("Category"),
                },
                titleRule: v => v.length <= this.$config.get('clip') || this.$gettext("Name too long"),
                growDesc: false,
            };
        },
        methods: {
            webdavUpload() {
                this.dialog.share = false;
                this.dialog.upload = true;
            },
            showUpload() {
                Event.publish("dialog.upload");
            },
            expand() {
                this.searchExpanded = !this.searchExpanded;
                this.growDesc = !this.growDesc;
            },
            updateAlbum() {
                if (this.album.wasChanged()) {
                    this.album.update();
                }
            },
            dropdownChange() {
                this.filterChange();

                if (window.innerWidth < 600) {
                    this.searchExpanded = false;
                }

                if (this.filter.order !== this.album.Order) {
                    this.album.Order = this.filter.order;
                    this.updateAlbum()
                }
            },
            setView(name) {
                this.settings.view = name;
                this.filterChange();
            },
            clearQuery() {
                this.filter.q = '';
                this.filterChange();
            },
        }
    };
</script>
