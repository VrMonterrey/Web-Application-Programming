<template>
    <v-card>
      <v-card-title>Tasks</v-card-title>
      <v-card-subtitle>
        Filtering
        <v-row>
          <v-col>
            <v-text-field
              v-model="search"
              @input="retrieveTasks"
              variant="solo"
              label="Match name"
            ></v-text-field>
          </v-col>
          <v-col cols="2">
            <div>Limit</div>
            <v-slider
              density="compact"
              v-model="limit"
              min="5"
              max="100"
              step="5"
              thumb-label
              @update:modelValue="retrieveTasks"
            ></v-slider>
          </v-col>
          <v-col cols="1">
            <v-btn
              variant="elevated"
              color="success"
              @click="add"
              v-if="checkIfInRole(user, [0, 1])"
              >Add</v-btn
            >
          </v-col>
          <v-col cols="1">
            <v-btn
              variant="elevated"
              color="cyan"
              @click="finish"
              v-if="checkIfInRole(user, [0, 1])"
              >Finish</v-btn
            >
          </v-col>
        </v-row>
      </v-card-subtitle>
  
      <v-card-text>
        <v-table density="compact" hover>
          <thead>
            <tr>
              <th class="text-center">Project name</th>
              <th class="text-center">Shortcut</th>
              <th class="text-center">Start date</th>
              <th class="text-center">Members</th>
              <th class="text-center">Tasks</th>
            </tr>
          </thead>
          <tbody>
  <tr v-for="(project, index) in projects" :key="index">
    <td class="text-center">{{ project.name }}</td>
    <td class="text-center">
      <v-chip :color="project.color">{{ project.shortcut }}</v-chip>
    </td>
    <td class="text-center">{{ new Date(project.startDate).toLocaleDateString() }}</td>
    <td class="text-center">{{ project.members }}</td>
    <td class="text-center">
      <v-btn @click="toggleTasks(project._id)">
        <v-icon>
          {{ expandedProjects.includes(project._id) ? 'mdi-chevron-right' : 'mdi-chevron-left' }}
        </v-icon>
      </v-btn>
    </td>
    <td colspan="5" v-if="expandedProjects.includes(project._id)">
              <v-table dense>
  <thead>
    <tr>
      <th class="text-center">Name</th>
      <th class="text-center">Start date</th>
      <th class="text-center">Finish date</th>
      <th class="text-center">Members</th>
      <th></th>
    </tr>
  </thead>
  <tbody>
    <tr v-for="(task, tindex) in tasks.filter(task => task.project_id === project._id)" :key="tindex">
      <td class="text-center" @click="click(task)">{{ task.name }}</td>
      <td class="text-center" @click="click(task)">{{ new Date(task.startDate).toLocaleDateString() }}</td>
      <td class="text-center" @click="click(task)">
        {{ task.finishDate ? new Date(task.finishDate).toLocaleDateString() : "Unfinished" }}
      </td>
      <td class="text-center" @click="click(task)">{{ task.persons.length }}</td>
      <td class="text-center">
        <v-checkbox class="d-inline-flex"
          v-model="task.checked"
          color="cyan"
          :disabled="task.finishDate !== null"
          :model-value="task.finishDate !== null ? true : task.checked"
        ></v-checkbox>
      </td>
    </tr>
  </tbody>
</v-table>

    </td>
  </tr>
</tbody>
        </v-table>
      </v-card-text>
    </v-card>
  
    <v-dialog v-model="editor" width="50%">
      <TaskEditor
        :id="id"
        @dataChanged="retrieveTasks"
        @cancel="cancel"
        @dataAccessFailed="onDataAccessFailed"
      />
    </v-dialog>
  
    <v-snackbar v-model="dataAccessError" color="error" timeout="3000">{{
      dataAccessErrorMsg
    }}</v-snackbar>
  </template>
  
  <script>
  import common from "../mixins/common";
  import TaskEditor from "./TaskEditor.vue";
  
  export default {
    name: "TasksLister",
    components: { TaskEditor },
    mixins: [common],
    props: ["user", "websocket", "eventSet"],
    methods: {
      retrieve() {
        this.id = null;
        this.editor = false;
        fetch("/project", {
          method: "GET",
        })
          .then((res) => res.json())
          .then((data) => {
            if (data.error) throw new Error(data.error);
            this.projects = data;
          })
          .catch((err) => this.onDataAccessFailed(err.message));
      },
      retrieveTasks() {
        this.id = null;
        this.editor = false;
          fetch(`/task?search=${encodeURIComponent(this.search)}&limit=${this.limit}`)
    .then(response => {
      if (!response.ok) {
        throw new Error(`HTTP error! status: ${response.status}`);
      }
      return response.json();
    })
    .then(data => this.tasks = data)
    .catch(err => this.onDataAccessFailed(err.message));
},
toggleTasks(projectId) {
    const index = this.expandedProjects.indexOf(projectId);
    if (index > -1) {
      this.expandedProjects.splice(index, 1); // Remove if already expanded
    } else {
      this.expandedProjects.push(projectId); // Add to expanded list
    }
  },
      finish() {
        const selectedTasks = this.tasks.filter((task) => task.checked);
  
        if (selectedTasks.length === 0) {
          // Brak zaznaczonych zadań
          return;
        }
  
        const currentDate = new Date().toISOString();
  
        selectedTasks.forEach((task) => {
          const updateData = {
            finishDate: currentDate,
          };
  
          // Wysłanie osobnego żądania PUT dla każdego zaznaczonego zadania
          fetch(`/task?_id=${task._id}`, {
            method: "PUT",
            headers: {
              "Content-Type": "application/json",
            },
            body: JSON.stringify(updateData),
          })
            .then((response) => response.json())
            .then((data) => {
              if (data.error) {
                throw new Error(data.error);
              }
              this.retrieveTasks();
            })
            .catch((err) => this.onDataAccessFailed(err.message));
        });
      },
      add() {
        this.id = null;
        this.editor = true;
      },
      click(row) {
        if (!this.checkIfInRole(this.user, [0, 1])) return;
        this.id = row._id;
        this.editor = true;
      },
      cancel() {
        this.id = null;
        this.editor = false;
      },
      onDataAccessFailed(data) {
        this.dataAccessErrorMsg = data;
        this.dataAccessError = true;
      },
    },
    data() {
      return {
        editor: false,
        projects: [],
        tasks: [],
        id: null,
        limit: 10,
        search: "",
        dataAccessError: false,
        dataAccessErrorMsg: "",
        expandedProjects: [],
      };
    },
    mounted() {
      this.retrieve();
      this.retrieveTasks();
    },
  };
  </script>
  