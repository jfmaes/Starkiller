<template>
  <div class="pa-4">
    <div class="d-flex align-center mb-4">
      <h3>Background Jobs</h3>
      <v-spacer />
      <v-btn
        small
        color="primary"
        :loading="loading"
        @click="refreshJobs"
      >
        <v-icon left small>fa-sync</v-icon>
        Refresh
      </v-btn>
      <v-switch
        v-model="autoRefresh"
        label="Auto-refresh"
        class="ml-4 mt-0"
        hide-details
      />
    </div>

    <v-alert
      v-if="error"
      type="error"
      dismissible
      class="mb-4"
      @input="error = null"
    >
      {{ error }}
    </v-alert>

    <v-alert
      v-if="successMessage"
      type="success"
      dismissible
      class="mb-4"
      @input="successMessage = null"
    >
      {{ successMessage }}
    </v-alert>

    <v-data-table
      :headers="headers"
      :items="jobs"
      :loading="loading"
      :items-per-page="10"
      class="elevation-1"
      no-data-text="No background jobs found. Jobs will appear here when you run modules with Background=true."
    >
      <template #item.status="{ item }">
        <v-chip
          :color="getStatusColor(item.status)"
          small
          dark
        >
          {{ item.status }}
        </v-chip>
      </template>

      <template #item.created_at="{ item }">
        {{ formatDate(item.created_at) }}
      </template>

      <template #item.actions="{ item }">
        <v-btn
          v-if="item.status === 'pulled' || item.status === 'continuous'"
          small
          color="error"
          :loading="killingJob === item.id"
          @click="killJob(item)"
        >
          <v-icon left small>fa-stop</v-icon>
          Kill
        </v-btn>
        <span v-else class="grey--text">
          {{ item.status === 'completed' ? 'Completed' : 'N/A' }}
        </span>
      </template>

      <template #item.input="{ item }">
        <span class="text-truncate" style="max-width: 300px; display: inline-block;">
          {{ truncateInput(item.input) }}
        </span>
      </template>
    </v-data-table>

    <v-dialog v-model="confirmDialog" max-width="400">
      <v-card>
        <v-card-title>Confirm Kill Job</v-card-title>
        <v-card-text>
          Are you sure you want to kill job #{{ jobToKill?.id }}?
        </v-card-text>
        <v-card-actions>
          <v-spacer />
          <v-btn text @click="confirmDialog = false">Cancel</v-btn>
          <v-btn color="error" @click="confirmKillJob">Kill Job</v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>
  </div>
</template>

<script>
import moment from "moment";
import * as agentTaskApi from "@/api/agent-task-api";

export default {
  name: "AgentJobs",
  props: {
    agent: {
      type: Object,
      required: true,
    },
  },
  data() {
    return {
      jobs: [],
      loading: false,
      error: null,
      successMessage: null,
      autoRefresh: true,
      autoRefreshInterval: null,
      killingJob: null,
      confirmDialog: false,
      jobToKill: null,
      headers: [
        { text: "ID", value: "id", width: "80px" },
        { text: "Task Name", value: "task_name" },
        { text: "Status", value: "status", width: "120px" },
        { text: "Input", value: "input" },
        { text: "Created", value: "created_at", width: "180px" },
        { text: "Actions", value: "actions", sortable: false, width: "120px" },
      ],
    };
  },
  watch: {
    autoRefresh(val) {
      if (val) {
        this.startAutoRefresh();
      } else {
        this.stopAutoRefresh();
      }
    },
    agent: {
      handler() {
        this.refreshJobs();
      },
      immediate: true,
    },
  },
  mounted() {
    this.refreshJobs();
    if (this.autoRefresh) {
      this.startAutoRefresh();
    }
  },
  beforeDestroy() {
    this.stopAutoRefresh();
  },
  methods: {
    async refreshJobs() {
      if (!this.agent?.session_id) return;

      this.loading = true;
      this.error = null;

      try {
        // Request the agent to report its jobs
        await agentTaskApi.getJobs(this.agent.session_id);

        // Fetch tasks that are background jobs (pulled or continuous status)
        const response = await agentTaskApi.getTasks(this.agent.session_id, {
          limit: 50,
          page: 1,
          sortBy: "id",
          sortOrder: "desc",
          status: null,
        });

        // Filter for background job task types
        const backgroundTaskTypes = [
          "TASK_CSHARP_CMD_JOB",
          "TASK_POWERSHELL_CMD_JOB",
          "TASK_PYTHON_CMD_JOB",
        ];

        this.jobs = response.records.filter((task) =>
          backgroundTaskTypes.includes(task.task_name) ||
          task.status === "continuous"
        );
      } catch (err) {
        this.error = err.message || "Failed to fetch jobs";
      } finally {
        this.loading = false;
      }
    },
    killJob(job) {
      this.jobToKill = job;
      this.confirmDialog = true;
    },
    async confirmKillJob() {
      if (!this.jobToKill) return;

      const job = this.jobToKill;
      this.confirmDialog = false;
      this.killingJob = job.id;
      this.error = null;

      try {
        await agentTaskApi.killJob(this.agent.session_id, job.id);
        this.successMessage = `Kill command sent for job #${job.id}`;
        // Refresh the jobs list after a short delay
        setTimeout(() => this.refreshJobs(), 1000);
      } catch (err) {
        this.error = err.message || `Failed to kill job #${job.id}`;
      } finally {
        this.killingJob = null;
        this.jobToKill = null;
      }
    },
    startAutoRefresh() {
      this.stopAutoRefresh();
      this.autoRefreshInterval = setInterval(() => {
        this.refreshJobs();
      }, 10000); // Refresh every 10 seconds
    },
    stopAutoRefresh() {
      if (this.autoRefreshInterval) {
        clearInterval(this.autoRefreshInterval);
        this.autoRefreshInterval = null;
      }
    },
    getStatusColor(status) {
      const colors = {
        queued: "orange",
        pulled: "blue",
        completed: "green",
        error: "red",
        continuous: "purple",
      };
      return colors[status] || "grey";
    },
    formatDate(date) {
      return moment(date).format("YYYY-MM-DD HH:mm:ss");
    },
    truncateInput(input) {
      if (!input) return "";
      return input.length > 50 ? input.substring(0, 50) + "..." : input;
    },
  },
};
</script>

<style scoped>
.text-truncate {
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}
</style>
