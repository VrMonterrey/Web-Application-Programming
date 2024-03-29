<template>
  <div>
    <v-card>
      <v-card-title>{{ id ? "Edit" : "Create" }} project</v-card-title>
      <v-card-text>
        <v-form v-model="isProjectValid">
          <v-text-field
            variant="solo"
            label="Name"
            v-model="project.name"
            :rules="[rules.required]"
          ></v-text-field>
          <div class="flex-container">
            <v-text-field
              variant="solo"
              label="Shortcut"
              v-model="project.shortcut"
              :rules="[rules.required]"
            ></v-text-field>
            <v-color-picker
              mode="rgb"
              v-model="project.color"
              hide-canvas
              hide-inputs
            ></v-color-picker>
          </div>
          <v-text-field
            variant="solo"
            type="date"
            label="Start date"
            v-model="project.startDate"
            :rules="[rules.validStartDate]"
          ></v-text-field>

          <v-select
            v-model="project.manager"
            :items="filteredPersons"
            label="Manager"
          ></v-select>

          <div class="flex-container">
            <v-btn
              variant="text"
              size="xx-small"
              rounded="false"
              icon="mdi-image-filter-center-focus"
              @click="centerView"
            ></v-btn>
            Location (click or drag to set)
          </div>

          <VMap
            ref="vmap"
            style="height: 200px"
            :center="center"
            zoom="15"
            @click="setMarker"
          >
            <VMapIconMarker
              ref="vmarker"
              v-model:latlng="project.coords"
              :icon-url="require('leaflet/dist/images/marker-icon.png')"
              :icon-retina-url="
                require('leaflet/dist/images/marker-icon-2x.png')
              "
              :icon-shadow-url="
                require('leaflet/dist/images/marker-shadow.png')
              "
              :icon-size="[28, 46]"
              :icon-anchor="[17, 46]"
              draggable
            ></VMapIconMarker>
            <VMapGoogleTileLayer />
            <VMapZoomControl />
          </VMap>
        </v-form>
      </v-card-text>
      <v-card-actions>
        <v-spacer></v-spacer>
        <v-btn
          variant="elevated"
          color="success"
          @click="add"
          :disabled="!isProjectValid"
          v-if="!id"
          >Add</v-btn
        >
        <v-btn
          variant="elevated"
          color="success"
          @click="modify"
          :disabled="!isProjectValid"
          v-if="id"
          >Modify</v-btn
        >
        <v-btn variant="elevated" color="error" @click="remove" v-if="id"
          >Remove</v-btn
        >
        <v-btn variant="elevated" color="warning" @click="cancel">Cancel</v-btn>
      </v-card-actions>
    </v-card>
    <v-dialog v-model="confirmation" width="auto">
      <ConfirmationDialog
        :question="'Are you sure to delete \'' + project.name + '\' ?'"
        @ok="removeReal"
        @cancel="confirmation = false"
      />
    </v-dialog>
  </div>
</template>

<script>
// import styles of vue-map-ui
import "leaflet/dist/leaflet.css";
import "vue-map-ui/dist/normalize.css";
import "vue-map-ui/dist/style.css";
import "vue-map-ui/dist/theme-all.css";

import {
  VMap,
  VMapGoogleTileLayer,
  VMapZoomControl,
  VMapIconMarker,
} from "vue-map-ui";

import common from "../mixins/common";
import ConfirmationDialog from "./ConfirmationDialog.vue";

export default {
  name: "ProjectEditor",
  props: ["id"],
  components: {
    ConfirmationDialog,
    VMap,
    VMapGoogleTileLayer,
    VMapZoomControl,
    VMapIconMarker,
  },
  emits: ["cancel", "dataChanged", "dataAccessFailed"],
  mixins: [common],
  methods: {
    add() {
      fetch("/project", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify(this.project),
      })
        .then((res) => res.json())
        .then((data) => {
          if (data.error) throw new Error(data.error);
          this.$emit("dataChanged");
        })
        .catch((err) => this.$emit("dataAccessFailed", err.message));
    },
    modify() {
      console.log("Project Manager modify:", this.project.manager);
      console.log("this.project", this.project);

      fetch("/project?_id=" + this.id, {
        method: "PUT",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify(this.project),
      })
        .then((res) => res.json())
        .then((data) => {
          if (data.error) throw new Error(data.error);
          this.$emit("dataChanged");
        })
        .catch((err) => this.$emit("dataAccessFailed", err.message));
    },
    remove() {
      this.confirmation = true;
    },
    removeReal() {
      this.confirmation = false;
      fetch("/project?_id=" + this.id, { method: "DELETE" })
        .then((res) => res.json())
        .then((data) => {
          if (data.error) throw new Error(data.error);
          this.removeTasks(this.id);
          this.$emit("dataChanged");
        })
        .catch((err) => this.$emit("dataAccessFailed", err.message));
    },
    cancel() {
      this.$emit("cancel");
    },
    setMarker(event) {
      this.project.coords = event.latlng;
    },
    centerView() {
      this.center = this.project.coords;
      this.$refs.vmap.map.flyTo(this.center);
    },
    removeTasks(projectId) {
  fetch(`/task?projectId=${encodeURIComponent(projectId)}`)
    .then(response => response.json())
    .then(tasks => {
      const updatePromises = tasks.map(task => {
        return fetch(`/task?_id=${encodeURIComponent(task._id)}`, {
          method: 'DELETE',
          headers: { 'Content-Type': 'application/json' },
        });
      });

      return Promise.all(updatePromises);
    })
    .then(responses => Promise.all(responses.map(res => {
      if (!res.ok) {
        throw new Error(`HTTP status ${res.status}`);
      }
      return res.json();
    })))
    .then(results => {
      console.log('All tasks deleted:', results);
    })
    .catch(err => console.error('Failed to delete tasks', err));
},
  },
  data() {
    return {
      isProjectValid: false,
      rules: {
        required: (value) => !!value || "empty value is not allowed",
        validStartDate: (value) =>
          !isNaN(new Date(value)) || "valid date required",
      },
      project: {
        color: this.defaultColor(),
        coords: this.defaultCoords(),
      },
      center: this.defaultCoords(),
      dialog: false,
      confirmation: false,
      ready: false,
      persons: [],
      filteredPersons: [],
    };
  },
  mounted() {
  fetch("/person?search&limit=1000000", {
    method: "GET",
  })
  .then(response => response.json())
  .then(personsData => {
    this.persons = personsData;

    if (this.id) {
      fetch("/project?_id=" + this.id, {
        method: "GET",
      })
      .then(projectResponse => {
        console.log("Project Editor mounted: ", projectResponse);
        return projectResponse.json();
      })
      .then(projectData => {
        if (projectData.error) {
          throw new Error(projectData.error);
        }

        if (!projectData.coords) {
          projectData.coords = this.defaultCoords();
        }
        Object.assign(this.project, projectData);
        Object.assign(this.center, this.project.coords);

        if (this.$refs.vmap && this.$refs.vmap.map) {
          this.$refs.vmap.map.panTo(this.center);
        }

        this.filteredPersons = this.persons
          .filter(person => 
            person.projects &&
            person.projects.some(project => project._id === this.project._id)
          )
          .map(person => ({
            value: person._id,
            title: person.firstName + " " + person.lastName,
          }));

        console.log("Project Manager mounted:", this.project.manager);
      })
      .catch(err => {
        this.$emit("dataAccessFailed", err.message);
      });
    }
  })
  .catch(err => {
    this.$emit("dataAccessFailed", err.message);
  });
},
};
</script>

<style scoped>
.flex-container {
  display: flex;
}
.flex-container > div {
  padding: 20px;
}
</style>
