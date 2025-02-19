<template>
  <v-card
    @drop.prevent="onDrop($event)"
    @dragover.prevent="dragover = true"
    @dragenter.prevent="dragover = true"
    @dragleave.prevent="dragover = false"
    flat
    tile
    color="transparent"
  >
    <v-card-text class="pa-0">
      <v-row class="d-flex flex-column" dense align="center" justify="center">
        <v-progress-linear
          color="rgb(213, 61, 14)"
          indeterminate
          v-if="loading"
          height="20"
        ></v-progress-linear>
        <v-icon
          v-if="!loading"
          :class="[dragover ? 'mt-0, mb-1' : 'mt-0']"
          size="20"
        >
          mdi-cloud-upload
        </v-icon>
        <p class="mb-0 pb-0" v-if="!loaded && !loading">
          Drop your Json-File here
        </p>
        <p v-if="!loaded && loading" class="mb-0 pb-0">
          Loading file from uri...
        </p>
        <p class="mb-0 pb-0" v-if="loaded">
          File loaded
        </p>
      </v-row>
      <v-virtual-scroll
        v-if="false"
        :items="uploadedFiles"
        height="70"
        item-height="50"
      >
        <template v-slot:default="{ item }">
          <v-list-item :key="item.name">
            <v-list-item-content>
              <v-list-item-title>
                {{ item.name }}
                <span class="ml-3 text--secondary"> {{ item.size }} bytes</span>
              </v-list-item-title>
            </v-list-item-content>

            <v-list-item-action>
              <v-btn @click.stop="removeFile(item.name)" icon>
                <v-icon> mdi-close-circle </v-icon>
              </v-btn>
            </v-list-item-action>
          </v-list-item>

          <v-divider></v-divider>
        </template>
      </v-virtual-scroll>
      <v-snackbar
        fixed
        v-model="snackbar.status"
        :timeout="snackbar.timeout"
        color="red"
        bottom
        right
      >
        {{ snackbar.text }}
        <template v-slot:action="{ attrs }">
          <v-btn
            color="white"
            text
            v-bind="attrs"
            @click="snackbar.status = false"
          >
            Close
          </v-btn>
        </template>
      </v-snackbar>
    </v-card-text>
  </v-card>
</template>

<script>
import axios from "axios";

export default {
  name: "Upload",
  props: {
    multiple: {
      type: Boolean,
      default: false
    }
  },
  data() {
    return {
      dragover: false,
      snackbar: {
        status: false,
        text: "",
        timeout: 5000
      },
      loading: false,
      loaded: false,
      uploadedFiles: []
    };
  },
  mounted: function() {
    if (this.$route.query.json) {
      this.loading = true;

      let uri = this.$route.query.json;

      // if private_token is set, it should be a gitlab uri
      if (this.$route.query.private_token) {
        let parts = this.$route.query.json.split("/repository/files/");
        let subparts = parts[parts.length - 1].split("/raw?ref");
        uri =
          parts[0] +
          "/repository/files/" +
          encodeURIComponent(encodeURIComponent(subparts[0])) +
          "/raw?ref" +
          subparts[1] +
          "&private_token=" +
          this.$route.query.private_token;
      }

      axios
        .get("/json?uri=" + uri, {
          private_token: this.$route.query.private_token
        })
        .then(response => {
          this.$store.dispatch("file/saveFile", { json: response.data.data });
          this.$emit("data-complete");
          this.loading = false;
          this.loaded = true;
        })
        .catch(error => {
          this.snackbar.text = "Data could not be retrieved";
          this.snackbar.status = true;
          this.loading = false;
          return Promise.reject(error);
        });
    }
  },
  methods: {
    removeFile(fileName) {
      const index = this.uploadedFiles.findIndex(
        file => file.name === fileName
      );
      if (index > -1) this.uploadedFiles.splice(index, 1);
    },
    onDrop(e) {
      this.dragover = false;
      if (this.uploadedFiles.length > 0) this.uploadedFiles = [];

      if (!this.multiple && e.dataTransfer.files.length > 1) {
        this.$store.dispatch("log/addNotification", {
          message: "Only one file can be uploaded at a time..",
          colour: "error"
        });
      }
      // Add each file to the array of uploaded files
      else {
        Array.from(e.dataTransfer.files).forEach(element =>
          this.uploadedFiles.push(element)
        );

        this.submit();
      }
    },
    submit() {
      if (!this.uploadedFiles.length > 0) {
        this.$store.dispatch("log/addNotification", {
          message: "There are no files to upload",
          colour: "error"
        });
      } else {
        var reader = new FileReader();
        reader.onload = event => {
          var parsedObj = JSON.parse(event.target.result);
          this.$store.dispatch("file/saveFile", { json: parsedObj });
          this.loaded = true;
          this.$emit("data-complete");
        };
        reader.readAsText(this.uploadedFiles[0]);
      }
    }
  }
};
</script>
