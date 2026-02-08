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
      :expanded.sync="expanded"
      item-key="id"
      show-expand
      class="elevation-1"
      no-data-text="No background jobs found. Jobs will appear here when you run modules with Background=true."
    >
      <template #item.status="{ item }">
        <v-chip
          :color="getStatusColor(item.agentStatus)"
          small
          dark
        >
          {{ item.agentStatus }}
        </v-chip>
      </template>

      <template #item.created_at="{ item }">
        {{ formatDate(item.created_at) }}
      </template>

      <template #item.actions="{ item }">
        <v-btn
          v-if="item.agentStatus === 'running' || item.agentStatus === 'started' || item.agentStatus === 'continuous'"
          small
          color="error"
          :loading="killingJob === item.id"
          @click="killJob(item)"
        >
          <v-icon left small>fa-stop</v-icon>
          Kill
        </v-btn>
        <span v-else class="grey--text">
          {{ item.agentStatus === 'completed' ? 'Completed' : 'N/A' }}
        </span>
      </template>

      <template #item.input="{ item }">
        <span class="text-truncate" style="max-width: 300px; display: inline-block;">
          {{ truncateInput(item.input) }}
        </span>
      </template>

      <template #expanded-item="{ headers, item }">
        <td :colspan="headers.length" class="pa-4">
          <div>
            <div class="d-flex align-center mb-2">
              <v-btn
                x-small
                text
                @click="downloadTaskInput(item)"
              >
                <v-icon left small>fa-download</v-icon>
                Download Input
              </v-btn>
              <v-btn
                x-small
                text
                class="ml-2"
                @click="downloadTaskOutput(item)"
              >
                <v-icon left small>fa-download</v-icon>
                Download Output
              </v-btn>
              <v-spacer />
              <v-switch
                v-model="expandedJobs[item.id].backgroundColor"
                false-value="white"
                true-value="black"
                label="Dark Background"
                class="mt-0"
                hide-details
              />
            </div>

            <p><b>Full Input:</b></p>
            <p
              :class="'mono ' + (expandedJobs[item.id].backgroundColor === 'white' ? 'font-black' : 'font-white')"
              :style="'background-color: ' + expandedJobs[item.id].backgroundColor + '; padding: 8px;'"
            >
              {{ item.input || 'No input' }}
            </p>

            <p><b>Output:</b></p>
            <div
              :class="'mono ' + (expandedJobs[item.id].backgroundColor === 'white' ? 'font-black' : 'font-white')"
              :style="'background-color: ' + expandedJobs[item.id].backgroundColor + '; padding: 8px;'"
            >
              <div v-if="expandedJobs[item.id].htmlOutput" v-html="expandedJobs[item.id].htmlOutput" />
              <div v-else>{{ item.output || 'No output yet' }}</div>
            </div>
          </div>
        </td>
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
import AnsiUp from "ansi_up";
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
      autoRefresh: false,
      autoRefreshInterval: null,
      killingJob: null,
      confirmDialog: false,
      jobToKill: null,
      expanded: [],
      expandedJobs: {},
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

        // Poll for the new TASK_GETJOBS to complete (wait up to 10 seconds)
        let getJobsTask = null;
        const startTime = Date.now();
        const maxWaitTime = 10000; // 10 seconds max
        const pollInterval = 500; // Check every 500ms

        while (Date.now() - startTime < maxWaitTime) {
          await new Promise(resolve => setTimeout(resolve, pollInterval));

          // Fetch all tasks to find the latest TASK_GETJOBS
          const response = await agentTaskApi.getTasks(this.agent.session_id, {
            limit: 100,
            page: 1,
            sortBy: "id",
            sortOrder: "desc",
            status: null,
          });

          // Find the most recent TASK_GETJOBS task that has completed
          const latestGetJobs = response.records.find(
            task => task.task_name === "TASK_GETJOBS" && task.status === "completed" && task.output
          );

          // Check if this is a new task (created in the last 15 seconds)
          if (latestGetJobs && latestGetJobs.output) {
            const taskAge = Date.now() - new Date(latestGetJobs.created_at).getTime();
            if (taskAge < 15000) { // Task created within last 15 seconds
              getJobsTask = latestGetJobs;
              break;
            }
          }
        }

        if (!getJobsTask || !getJobsTask.output) {
          // Fallback: use the most recent completed TASK_GETJOBS even if it's old
          const response = await agentTaskApi.getTasks(this.agent.session_id, {
            limit: 100,
            page: 1,
            sortBy: "id",
            sortOrder: "desc",
            status: null,
          });

          getJobsTask = response.records.find(
            task => task.task_name === "TASK_GETJOBS" && task.status === "completed" && task.output
          );

          if (!getJobsTask || !getJobsTask.output) {
            this.jobs = [];
            return;
          }
        }

        // Fetch all tasks again to get the full list for matching
        const response = await agentTaskApi.getTasks(this.agent.session_id, {
          limit: 100,
          page: 1,
          sortBy: "id",
          sortOrder: "desc",
          status: null,
        });

        // Parse the TASK_GETJOBS output to extract background job information
        const agentJobs = this.parseGetJobsOutput(getJobsTask.output);

        // Build jobs array by matching agent jobs with their corresponding tasks
        this.jobs = agentJobs.map(agentJob => {
          // Find the task that created this job (look for "Job started: X" in output)
          const creatorTask = response.records.find(task => {
            if (!task.output) return false;
            return task.output.includes(`Job started: ${agentJob.jobId}`);
          });

          // If we found the creator task, use its details; otherwise use minimal info
          if (creatorTask) {
            return {
              id: agentJob.jobId,
              jobId: agentJob.jobId,
              agentStatus: agentJob.status, // Status from agent's perspective
              task_name: creatorTask.task_name,
              module_name: creatorTask.module_name,
              input: creatorTask.input,
              output: creatorTask.output,
              created_at: creatorTask.created_at,
              updated_at: creatorTask.updated_at,
              taskId: creatorTask.id, // Store the Empire task ID separately
            };
          } else {
            // Job exists on agent but we couldn't find the task that created it
            return {
              id: agentJob.jobId,
              jobId: agentJob.jobId,
              agentStatus: agentJob.status,
              task_name: 'Unknown',
              module_name: null,
              input: 'Unknown',
              output: '',
              created_at: null,
              updated_at: null,
              taskId: null,
            };
          }
        }).filter(job => {
          // Only show jobs that are still active on the agent
          return job.agentStatus === 'running' ||
                 job.agentStatus === 'started' ||
                 job.agentStatus === 'queued';
        });

        // Process each job for display
        this.jobs.forEach((job) => {
          if (!this.expandedJobs[job.id]) {
            this.$set(this.expandedJobs, job.id, {
              backgroundColor: 'white',
              htmlOutput: this.ansiToHtml(job.output),
            });
          } else {
            // Update existing job's output
            this.expandedJobs[job.id].htmlOutput = this.ansiToHtml(job.output);
          }
        });
      } catch (err) {
        this.error = err.message || "Failed to fetch jobs";
      } finally {
        this.loading = false;
      }
    },
    parseGetJobsOutput(output) {
      // Parse the table output from TASK_GETJOBS
      // Expected format:
      // Task ID | Status
      // --------------------
      // 27 | started
      // 1 | running
      const jobs = [];
      const lines = output.split('\n');

      for (const line of lines) {
        // Skip header and separator lines
        if (line.includes('Task ID') || line.includes('---') || !line.trim()) {
          continue;
        }

        // Parse job lines (format: "ID | status")
        const match = line.match(/^\s*(\d+)\s*\|\s*(\w+)\s*$/);
        if (match) {
          jobs.push({
            jobId: parseInt(match[1], 10),
            status: match[2],
          });
        }
      }

      return jobs;
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
        // Wait for agent to process kill and update job list (increased delay)
        setTimeout(() => this.refreshJobs(), 3000);
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
        running: "blue",      // Agent-side running status
        started: "orange",    // Agent-side started status
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
    ansiToHtml(text) {
      if (!text) return "";
      const ansi_up = new AnsiUp();
      return ansi_up.ansi_to_html(text);
    },
    downloadTaskInput(task) {
      const blob = new Blob([task.input || ""], { type: "text/plain" });
      const url = window.URL.createObjectURL(blob);
      const a = document.createElement("a");
      a.href = url;
      a.download = `task-${task.id}-input.txt`;
      a.click();
      window.URL.revokeObjectURL(url);
    },
    downloadTaskOutput(task) {
      const blob = new Blob([task.output || ""], { type: "text/plain" });
      const url = window.URL.createObjectURL(blob);
      const a = document.createElement("a");
      a.href = url;
      a.download = `task-${task.id}-output.txt`;
      a.click();
      window.URL.revokeObjectURL(url);
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

.mono {
  font-family: monospace;
  white-space: pre-wrap;
  word-break: break-word;
}

.font-black {
  color: black;
}

.font-white {
  color: white;
}
</style>
