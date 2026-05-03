<template>
  <div class="take-control-root">
    <!-- ─── Toolbar ─────────────────────────────────────────────────────── -->
    <q-bar class="bg-grey-9 text-white take-control-bar">
      <!-- Agent status -->
      <span class="text-caption">
        TRMM Agent:
        <q-badge :color="statusColor" :label="status" />
      </span>

      <q-separator dark vertical class="q-mx-sm" />

      <!-- Connection controls -->
      <q-btn
        dense flat color="white" size="sm"
        icon="refresh" label="Restart"
        @click="restartMeshService"
      />
      <q-btn
        dense flat size="sm"
        :color="dash_negative_color"
        icon="fas fa-first-aid" label="Recover"
        @click="repairMeshCentral"
      />

      <q-space />

      <!-- ── Remote session tools ───────────────────────────────────────── -->
      <q-btn
        dense flat color="white" size="sm"
        icon="content_paste" label="Sync Clipboard"
        @click="syncClipboard"
      >
        <q-tooltip class="bg-grey-8">Copy local clipboard to remote machine</q-tooltip>
      </q-btn>

      <q-btn
        dense flat color="white" size="sm"
        icon="keyboard" label="Send Keys"
        class="q-ml-xs"
        @click="sendKeysDialog = true"
      >
        <q-tooltip class="bg-grey-8">Type text on the remote machine</q-tooltip>
      </q-btn>

      <q-btn
        dense flat size="sm"
        :color="credPanelOpen ? 'positive' : 'white'"
        icon="vpn_key" label="Credentials"
        class="q-ml-xs"
        @click="toggleCredPanel"
      >
        <q-tooltip class="bg-grey-8">Vaultwarden credentials for this client</q-tooltip>
      </q-btn>

      <q-space />
    </q-bar>

    <!-- ─── Main area ─────────────────────────────────────────────────── -->
    <div
      class="take-control-body"
      :style="{ height: `${$q.screen.height - 26}px` }"
    >
      <!-- MeshCentral iframe -->
      <div class="iframe-wrapper">
        <iframe
          ref="meshIframe"
          v-show="control"
          :src="control"
          allow="clipboard-read; clipboard-write"
          allowfullscreen
          frameborder="0"
          class="mesh-iframe"
        />
      </div>

      <!-- ── Credentials panel (slides in from right) ─────────────────── -->
      <transition name="slide-right">
        <div v-if="credPanelOpen" class="cred-panel bg-grey-9">
          <!-- Panel header -->
          <div class="cred-panel-header bg-grey-8 row items-center q-px-sm q-py-xs">
            <q-icon name="vpn_key" color="amber" class="q-mr-xs" />
            <span class="text-caption text-white text-bold col">Credentials</span>
            <q-btn dense flat round color="white" icon="close" size="xs" @click="credPanelOpen = false" />
          </div>

          <!-- Client context -->
          <div v-if="clientName" class="q-px-sm q-pt-xs">
            <q-chip dense icon="business" color="grey-7" text-color="white" size="sm">
              {{ clientName }}
            </q-chip>
          </div>

          <!-- Search -->
          <q-input
            v-model="credSearch"
            dense outlined dark
            placeholder="Search credentials…"
            class="q-ma-sm"
            clearable
          >
            <template v-slot:prepend>
              <q-icon name="search" color="grey-5" />
            </template>
          </q-input>

          <!-- Cred list -->
          <q-scroll-area class="cred-scroll">
            <!-- Loading -->
            <div v-if="credsLoading" class="q-pa-sm">
              <q-skeleton v-for="n in 3" :key="n" type="QBtn" class="q-mb-xs" />
            </div>

            <!-- Error -->
            <div v-else-if="credsError" class="q-pa-sm text-caption text-negative">
              {{ credsError }}
            </div>

            <!-- Empty -->
            <div v-else-if="filteredCreds.length === 0" class="q-pa-sm text-caption text-grey-5 text-center">
              No credentials found
            </div>

            <!-- Items -->
            <q-list v-else dense dark separator>
              <q-expansion-item
                v-for="cred in filteredCreds"
                :key="cred.id"
                :label="cred.name"
                :caption="cred.username || ''"
                icon="lock"
                dense
                dark
                expand-separator
              >
                <q-card dark class="bg-grey-10">
                  <q-card-section class="q-pa-sm">
                    <!-- Username row -->
                    <div v-if="cred.username" class="q-mb-sm">
                      <div class="row items-center">
                        <span class="text-caption text-grey-5 col">Username</span>
                        <q-btn
                          flat dense round icon="content_copy" color="grey-4" size="xs"
                          @click="copyField(cred.username, 'Username')"
                        >
                          <q-tooltip class="bg-grey-8">Copy to clipboard</q-tooltip>
                        </q-btn>
                        <q-btn
                          flat dense round icon="keyboard" color="positive" size="xs"
                          @click="sendAsKeystrokes(cred.username)"
                        >
                          <q-tooltip class="bg-grey-8">Send as keystrokes to remote</q-tooltip>
                        </q-btn>
                      </div>
                      <div class="text-caption text-white text-truncate">{{ cred.username }}</div>
                    </div>

                    <!-- Password row -->
                    <div v-if="cred.password">
                      <div class="row items-center">
                        <span class="text-caption text-grey-5 col">Password</span>
                        <q-btn
                          flat dense round icon="content_copy" color="grey-4" size="xs"
                          @click="copyField(cred.password, 'Password')"
                        >
                          <q-tooltip class="bg-grey-8">Copy to clipboard</q-tooltip>
                        </q-btn>
                        <q-btn
                          flat dense round icon="keyboard" color="positive" size="xs"
                          @click="sendAsKeystrokes(cred.password)"
                        >
                          <q-tooltip class="bg-grey-8">Send as keystrokes to remote</q-tooltip>
                        </q-btn>
                        <q-btn
                          flat dense round
                          :icon="cred._showPass ? 'visibility_off' : 'visibility'"
                          color="grey-4" size="xs"
                          @click="cred._showPass = !cred._showPass"
                        />
                      </div>
                      <div class="text-caption text-white">
                        <span v-if="cred._showPass">{{ cred.password }}</span>
                        <span v-else>••••••••</span>
                      </div>
                    </div>

                    <!-- Notes (if any) -->
                    <div v-if="cred.notes" class="q-mt-xs">
                      <div class="text-caption text-grey-5">Notes</div>
                      <div class="text-caption text-grey-4">{{ cred.notes }}</div>
                    </div>
                  </q-card-section>
                </q-card>
              </q-expansion-item>
            </q-list>
          </q-scroll-area>

          <!-- Refresh footer -->
          <div class="q-pa-sm">
            <q-btn
              outline size="sm" color="grey-5"
              icon="refresh" label="Refresh"
              class="full-width"
              :loading="credsLoading"
              @click="loadCreds"
            />
          </div>
        </div>
      </transition>
    </div>

    <!-- ─── Send Keys dialog ─────────────────────────────────────────── -->
    <q-dialog v-model="sendKeysDialog" position="top">
      <q-card dark class="bg-grey-9" style="min-width: 420px">
        <q-toolbar class="bg-grey-8">
          <q-icon name="keyboard" class="q-mr-sm" />
          <q-toolbar-title class="text-subtitle2">Send Keystrokes to Remote</q-toolbar-title>
          <q-btn flat round dense icon="close" v-close-popup />
        </q-toolbar>
        <q-card-section>
          <p class="text-caption text-grey-4 q-mb-sm">
            Text will be set on the remote clipboard and typed automatically.
          </p>
          <q-input
            v-model="sendKeysText"
            type="textarea"
            outlined dark dense
            :rows="4"
            placeholder="Type or paste the text to send…"
            autofocus
            @keyup.ctrl.enter="sendKeysToRemote"
          />
          <p class="text-caption text-grey-5 q-mt-xs">Tip: Ctrl+Enter to send</p>
        </q-card-section>
        <q-card-actions align="right" class="q-pt-none">
          <q-btn flat label="Cancel" v-close-popup />
          <q-btn
            color="positive" icon="keyboard" label="Send to Remote"
            :disable="!sendKeysText"
            @click="sendKeysToRemote"
          />
        </q-card-actions>
      </q-card>
    </q-dialog>
  </div>
</template>

<script>
import { ref, computed, onMounted, reactive } from "vue";
import { useStore } from "vuex";
import { useRoute } from "vue-router";
import { useMeta, useQuasar } from "quasar";
import { fetchAgentMeshCentralURLs, sendAgentRecoverMesh, fetchAgentVaultCreds } from "@/api/agents";
import { fetchDashboardInfo } from "@/api/core";
import { sendAgentServiceAction } from "@/api/services";
import { notifySuccess, notifyError } from "@/utils/notify";

export default {
  name: "TakeControl",
  setup() {
    onMounted(() => {
      dashInfo();
      getDashInfo();
      getMeshURLs();
    });

    const $q = useQuasar();
    const store = useStore();
    const dash_positive_color = computed(() => store.state.dash_positive_color);
    const dash_negative_color = computed(() => store.state.dash_negative_color);
    const dash_warning_color = computed(() => store.state.dash_warning_color);

    const { params } = useRoute();

    // ── Mesh session ────────────────────────────────────────────────────
    const meshIframe = ref(null);
    const control = ref("");
    const status = ref(null);
    const clientName = ref("");
    const siteName = ref("");

    const statusColor = computed(() => {
      switch (status.value) {
        case "online":  return dash_positive_color.value;
        case "offline": return dash_warning_color.value;
        default:        return dash_negative_color.value;
      }
    });

    const dashInfo = () => store.dispatch("getDashInfo", false);

    async function getMeshURLs() {
      $q.loading.show();
      try {
        const data = await fetchAgentMeshCentralURLs(params.agent_id);
        control.value   = data.control;
        status.value    = data.status;
        clientName.value = data.client || "";
        siteName.value  = data.site || "";
        useMeta({
          title: `${data.hostname} - ${data.client} - ${data.site} | Take Control`,
        });
      } catch (e) {
        console.error(e);
      }
      $q.loading.hide();
    }

    async function getDashInfo() {
      const { dark_mode, loading_bar_color } = await fetchDashboardInfo();
      $q.dark.set(dark_mode);
      $q.loadingBar.setDefaults({ color: loading_bar_color });
    }

    async function repairMeshCentral() {
      control.value = "";
      $q.loading.show({ message: "Attempting to repair Mesh Agent" });
      try {
        const data = await sendAgentRecoverMesh(params.agent_id);
        await getMeshURLs();
        setTimeout(() => notifySuccess(data), 500);
      } catch (e) {
        console.error(e);
      }
      $q.loading.hide();
    }

    async function restartMeshService() {
      $q.loading.show({ message: "Restarting Mesh Agent" });
      try {
        await sendAgentServiceAction(params.agent_id, "mesh agent", { sv_action: "restart" });
        setTimeout(() => notifySuccess("Mesh agent service was restarted"), 500);
      } catch (e) {
        console.error(e);
      }
      $q.loading.hide();
    }

    // ── Clipboard sync ───────────────────────────────────────────────────
    // Reads the local browser clipboard and pushes it to the remote machine
    // via MeshCentral's iframe postMessage API.
    async function syncClipboard() {
      try {
        const text = await navigator.clipboard.readText();
        if (!text) {
          $q.notify({ type: "warning", message: "Local clipboard is empty" });
          return;
        }
        pushToRemoteClipboard(text);
        $q.notify({ type: "positive", message: "Clipboard synced to remote", timeout: 1500 });
      } catch (e) {
        $q.notify({
          type: "warning",
          message: "Could not read clipboard — browser permission required",
          caption: "Click inside the page first, then try again",
        });
      }
    }

    // Send text to the MeshCentral iframe remote clipboard via postMessage.
    // MeshCentral listens for { type: 'clipboard', data } and
    // { action: 'setClipboard', data } (version-dependent).
    function pushToRemoteClipboard(text) {
      const iframe = meshIframe.value;
      if (!iframe || !iframe.contentWindow) return;
      iframe.contentWindow.postMessage({ type: "clipboard", data: text }, "*");
      iframe.contentWindow.postMessage({ action: "setClipboard", data: text }, "*");
    }

    // ── Send as keystrokes ───────────────────────────────────────────────
    const sendKeysDialog = ref(false);
    const sendKeysText = ref("");

    function openSendKeys(prefill = "") {
      sendKeysText.value = prefill;
      sendKeysDialog.value = true;
    }

    // Pushes text to the remote clipboard, then injects a synthetic Ctrl+V
    // via MeshCentral's postMessage paste action so the text is typed without
    // the user having to manually paste.
    function sendKeysToRemote() {
      if (!sendKeysText.value) return;
      _sendText(sendKeysText.value);
      sendKeysDialog.value = false;
      sendKeysText.value = "";
    }

    function sendAsKeystrokes(text) {
      _sendText(text);
    }

    function _sendText(text) {
      // 1. Write to local clipboard as a fallback
      navigator.clipboard.writeText(text).catch(() => {});

      // 2. Push to remote clipboard via MeshCentral postMessage
      pushToRemoteClipboard(text);

      // 3. Ask MeshCentral to paste (Ctrl+V equivalent) on the remote
      const iframe = meshIframe.value;
      if (iframe && iframe.contentWindow) {
        iframe.contentWindow.postMessage({ action: "pasteClipboard" }, "*");
        iframe.contentWindow.postMessage({ type: "keypress", key: "v", ctrlKey: true }, "*");
      }

      $q.notify({
        type: "positive",
        icon: "keyboard",
        message: "Text sent to remote",
        caption: "If it didn't auto-paste, press Ctrl+V in the remote window",
        timeout: 3000,
      });
    }

    // ── Vaultwarden credentials panel ────────────────────────────────────
    const credPanelOpen = ref(false);
    const credSearch = ref("");
    const creds = ref([]);
    const credsLoading = ref(false);
    const credsError = ref("");

    const filteredCreds = computed(() => {
      const q = credSearch.value.toLowerCase().trim();
      if (!q) return creds.value;
      return creds.value.filter(
        (c) =>
          c.name.toLowerCase().includes(q) ||
          (c.username && c.username.toLowerCase().includes(q)),
      );
    });

    async function loadCreds() {
      credsLoading.value = true;
      credsError.value = "";
      try {
        const data = await fetchAgentVaultCreds(params.agent_id);
        // Attach reactive _showPass flag to each credential
        creds.value = (data || []).map((c) => reactive({ ...c, _showPass: false }));
      } catch (e) {
        credsError.value = "Failed to load credentials. Is Vaultwarden configured?";
        console.error(e);
      }
      credsLoading.value = false;
    }

    function toggleCredPanel() {
      credPanelOpen.value = !credPanelOpen.value;
      if (credPanelOpen.value && creds.value.length === 0) {
        loadCreds();
      }
    }

    function copyField(text, label) {
      navigator.clipboard.writeText(text).then(() => {
        $q.notify({ type: "positive", message: `${label} copied to clipboard`, timeout: 1500 });
      });
    }

    return {
      // refs
      meshIframe,
      control,
      status,
      clientName,
      siteName,
      statusColor,
      dash_negative_color,
      // clipboard
      syncClipboard,
      // send keys
      sendKeysDialog,
      sendKeysText,
      sendKeysToRemote,
      // credentials
      credPanelOpen,
      credSearch,
      creds,
      credsLoading,
      credsError,
      filteredCreds,
      loadCreds,
      toggleCredPanel,
      copyField,
      sendAsKeystrokes,
      // mesh controls
      repairMeshCentral,
      restartMeshService,
    };
  },
};
</script>

<style scoped>
.take-control-root {
  display: flex;
  flex-direction: column;
  height: 100vh;
  overflow: hidden;
}

.take-control-bar {
  flex-shrink: 0;
  height: 26px;
}

.take-control-body {
  display: flex;
  flex-direction: row;
  flex: 1;
  overflow: hidden;
}

.iframe-wrapper {
  flex: 1;
  overflow: hidden;
  position: relative;
}

.mesh-iframe {
  width: 100%;
  height: 100%;
  display: block;
  border: none;
}

/* ── Credentials panel ─────────────────────────────────────────────── */
.cred-panel {
  width: 320px;
  flex-shrink: 0;
  display: flex;
  flex-direction: column;
  border-left: 1px solid #444;
  overflow: hidden;
}

.cred-panel-header {
  flex-shrink: 0;
  min-height: 36px;
}

.cred-scroll {
  flex: 1;
}

/* ── Slide-in transition ───────────────────────────────────────────── */
.slide-right-enter-active,
.slide-right-leave-active {
  transition: width 0.2s ease, opacity 0.2s ease;
  overflow: hidden;
}

.slide-right-enter-from,
.slide-right-leave-to {
  width: 0 !important;
  opacity: 0;
}
</style>
