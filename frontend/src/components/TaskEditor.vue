<template>
  <div>
    <v-card>
      <v-card-title>{{ id ? "Edit" : "Create" }} task</v-card-title>
      <v-card-text>
        <v-form v-model="isTaskValid">
          <v-text-field
            variant="solo"
            label="Name"
            v-model="task.name"
            :rules="[rules.required]"
          ></v-text-field>
          <v-text-field
            variant="solo"
            type="date"
            label="Start date"
            v-model="task.startDate"
            :rules="[rules.validStartDate]"
          ></v-text-field>
          <v-text-field
            variant="solo"
            type="date"
            label="Finish date"
            v-model="task.finishDate"
          ></v-text-field>
          <v-select
            v-model="selectedProjectName"
            :items="this.projects.map((obj) => obj.name)"
            label="Project"
          ></v-select>
          <v-select
            v-model="task.persons"
            :items="filteredPersons"
            label="Members"
            multiple
          ></v-select>
        </v-form>
      </v-card-text>
      <v-card-actions>
        <v-spacer></v-spacer>
        <v-btn
          variant="elevated"
          color="success"
          @click="add"
          :disabled="!isTaskValid"
          v-if="!id"
          >Add</v-btn
        >
        <v-btn
          variant="elevated"
          color="success"
          @click="modify"
          :disabled="!isTaskValid"
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
        :question="'Are you sure to delete \'' + task.name + '\' ?'"
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

import common from "../mixins/common";
import ConfirmationDialog from "./ConfirmationDialog.vue";

export default {
  name: "ProjectEditor",
  props: ["id"],
  components: {
    ConfirmationDialog,
  },
  emits: ["cancel", "dataChanged", "dataAccessFailed"],
  mixins: [common],
  computed: {
    selectedProject() {
      return this.projects.find(
        (project) => project._id === this.task.project_id
      );
    },
    selectedProjectName: {
      get() {
        return this.selectedProject ? this.selectedProject.name : "";
      },
      set(newValue) {
        const project = this.projects.find((p) => p.name === newValue);
        this.task.project_id = project ? project._id : null;
      },
    },

    selectedTask() {
      return this.persons.find((person) => person.projects);
    },
    selectedTaskMembers: {
      get() {
        return this.selectedProject ? this.selectedProject.name : "";
      },
      set(newValue) {
        const project = this.projects.find(
          (project) => project.name === newValue
        );
        this.task.project_id = project ? project._id : null;
      },
    },
  },
  watch: {
    selectedProject: {
      handler(newProject, oldProject) {
        if (newProject && newProject !== oldProject) {
          this.updateFilteredPersons(newProject);
        }
        console.log("newProject selectedProject", newProject);
      },
      deep: true,
    },
    selectedProjectName(newValue) {
      const project = this.projects.find((p) => p.name === newValue);
      this.updateFilteredPersons(project);
    },
  },

  methods: {
    add() {
      if (Array.isArray(this.task.project_id)) {
        this.task.project_id = this.task.project_id[0];
      }

      fetch("/task", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify(this.task),
      })
        .then((res) => res.json())
        .then((data) => {
          if (data.error) throw new Error(data.error);
          this.$emit("dataChanged");
        })
        .catch((err) => this.$emit("dataAccessFailed", err.message));
    },
    modify() {
      fetch("/task?_id=" + this.id, {
        method: "PUT",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify(this.task),
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
      fetch("/task?_id=" + this.id, { method: "DELETE" })
        .then((res) => res.json())
        .then((data) => {
          if (data.error) throw new Error(data.error);
          this.$emit("dataChanged");
        })
        .catch((err) => this.$emit("dataAccessFailed", err.message));
    },
    cancel() {
      this.$emit("cancel");
    },
    updateFilteredPersons(selectedProject) {
      if (selectedProject) {
        this.filteredPersons = this.persons
          .filter(
            (person) =>
              person.projects &&
              person.projects.some(
                (project) => project._id === selectedProject._id
              )
          )
          .map((person) => ({
            value: person._id,
            title: person.firstName + " " + person.lastName,
          }));

        console.log("Updated Filtered Persons:", this.filteredPersons);
      }
    },
  },

  data() {
    return {
      isTaskValid: false,
      rules: {
        required: (value) => !!value || "empty value is not allowed",
        validStartDate: (value) =>
          !isNaN(new Date(value)) || "valid date required",
      },
      dialog: false,
      confirmation: false,
      ready: false,
      task: {
        name: "",
        startDate: "",
        finishDate: "",
        project_id: "",
        persons: [],
      },
      projects: [],
      filteredPersons: [],
      persons: [],
    };
  },
  async mounted() {
    try {
      const projectsResponse = await fetch(
        "/project" + "?search&limit=1000000",
        {
          method: "GET",
        }
      );
      const projectsData = await projectsResponse.json();

      this.projects = projectsData;

      const personsResponse = await fetch("/person" + "?search&limit=1000000", {
        method: "GET",
      });
      const personsData = await personsResponse.json();
      this.persons = personsData;
      if (this.id) {
        const taskResponse = await fetch("/task?_id=" + this.id, {
          method: "GET",
        });
        const taskData = await taskResponse.json();

        if (taskData.error) {
          throw new Error(taskData.error);
        }
        Object.assign(this.task, taskData);

        const selectedProject = this.projects.find(
          (project) => project._id === this.task.project_id
        );

        if (selectedProject) {
          this.filteredPersons = this.persons
            .filter(
              (person) =>
                person.projects &&
                person.projects.some(
                  (project) => project._id === selectedProject._id
                )
            )
            .map((person) => ({
              value: person._id,
              title: person.firstName + " " + person.lastName,
            }));

          this.task.members = this.task.persons;
          this.task.persons = this.task.members;

          console.log("this.task.members: ", this.task.members);
          console.log("Filtered Persons:", this.filteredPersons);
        }
      }
    } catch (err) {
      this.$emit("dataAccessFailed", err.message);
    }
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
