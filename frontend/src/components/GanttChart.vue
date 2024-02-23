<template>
  <div class="gantt-chart">
    <svg width="100%" :height="chartHeight + 50">
      <g v-for="(task, index) in processedTasks" :key="index">
        <rect :x="getX(task.startDate)" :y="getY(index)" :width="getWidth(task.startDate, task.finishDate)" :height="barHeight" :fill="getFillColor(task.hasEndDate)"></rect>
        <text :x="getTextX(task.startDate, task.finishDate)" :y="getY(index) + barHeight / 2" alignment-baseline="middle" :fill="textColor">{{ task.name }}</text>
      </g>
      <g transform="translate(0,20)">
        <line x1="0" :y1="chartHeight" x2="100%" :y2="chartHeight" style="stroke: #000;"/>
        <g v-for="date in dates" :key="date.getTime()"> 
          <text :x="getX(date)" :y="chartHeight + 20" style="font-size: 10px;">{{ formatDate(date) }}</text>
        </g>
      </g>
    </svg>
  </div>
</template>


<script>
export default {
  name: 'GanttChart',
  props: {
    tasks: {
      type: Array,
      required: true,
      default: () => ([]),
    },
  },
  data() {
    return {
      barHeight: 20,
      textColor: '#fff',
    };
  },
  computed: {
    chartHeight() {
      return this.processedTasks.length * (this.barHeight + 10);
    },
    startDate() {
      return new Date(Math.min(...this.tasks.map(task => new Date(task.startDate))));
    },
    endDate() {
      return new Date(Math.max(...this.tasks.map(task => task.finishDate ? new Date(task.finishDate) : new Date())));
    },
    processedTasks() {
  return this.tasks.map(task => ({
    ...task,
    startDate: new Date(task.startDate),
    hasEndDate: !!task.finishDate,
    finishDate: task.finishDate ? new Date(task.finishDate) : new Date(new Date().setDate(new Date().getDate() + 1)).setHours(0, 0, 0, 0),
  }));
},
    dates() {
    const dates = [];
    let currentDate = new Date(this.startDate);
    const endDate = new Date(this.endDate);

    const msPerDay = 1000 * 60 * 60 * 24;
    const totalDays = Math.ceil((endDate - currentDate) / msPerDay);

    let stepDays = 1;

    if (totalDays > 730) { 
      stepDays = Math.ceil(totalDays / 365 / 2); 
    } else if (totalDays > 365) { 
      stepDays = Math.ceil(totalDays / 365 / 4); 
    } else if (totalDays > 90) { 
      stepDays = 30;
    } else if (totalDays > 30) { 
      stepDays = 7;
    }

    while (currentDate <= endDate) {
      dates.push(new Date(currentDate));
      currentDate.setDate(currentDate.getDate() + stepDays);
    }

    return dates;
  },
  },
  methods: {
    getX(date) {
      const diff = date - this.startDate;
      return diff * (1000 / (this.endDate - this.startDate)); 
    },
    getY(index) {
      return index * (this.barHeight + 10);
    },
    getWidth(startDate, finishDate) {
      const diff = finishDate - startDate;
      return diff * (1000 / (this.endDate - this.startDate));
    },
    getTextX(startDate, finishDate) {
      return this.getX(startDate) + (this.getWidth(startDate, finishDate) / 2) - 20; 
    },
    getFillColor(hasEndDate) {
  return hasEndDate ? '#4caf50' : '#ff9800';
},
formatDate(date) {
    return `${date.getDate()}/${date.getMonth() + 1}`;
  },
},
  }
</script>

<style scoped>
.gantt-chart {
  background-color: white;
  overflow-x: scroll;
  border: 1px solid #ccc;
  padding: 20px;
}
</style>