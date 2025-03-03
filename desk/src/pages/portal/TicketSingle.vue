<template>
  <span>
    <TopBar
      :title="ticket.doc?.subject"
      :back-to="{ name: CUSTOMER_PORTAL_LANDING }"
    >
      <template #bottom>
        <div class="flex items-center gap-1 text-base italic">
          <span># {{ ticket.doc?.name }}</span>
          <IconDot class="h-4 w-4" />
          <span>{{ ticket.doc?.status }}</span>
        </div>
      </template>
      <template #right>
        <span v-if="ticket.doc?.status !== 'Closed'">
          <Button
            v-if="ticket.doc?.status == 'Resolved'"
            label="Reopen"
            theme="gray"
            variant="outline"
            @click="ticket.reopen.submit()"
          >
            <template #prefix>
              <IconRepeat class="h-4 w-4" />
            </template>
          </Button>
          <Button
            v-else
            label="Mark as resolved"
            theme="gray"
            variant="outline"
            @click="ticket.resolve.submit()"
          >
            <template #prefix>
              <IconCheck class="h-4 w-4" />
            </template>
          </Button>
        </span>
      </template>
    </TopBar>
    <div class="flex flex-col gap-6 px-9 py-4 text-base text-gray-900">
      <div v-for="c in ticket.getCommunications.data?.message" :key="c.name">
        <CommunicationItem
          :content="c.content"
          :date="c.creation"
          :sender="c.sender.full_name"
          :sender-image="c.sender.image"
          :cc="c.cc"
          :bcc="c.bcc"
          :attachments="c.attachments"
        />
      </div>
      <TextEditor
        ref="textEditor"
        :placeholder="placeholder"
        :content="editorContent"
        @change="(v) => (editorContent = v)"
      >
        <template #bottom="{ editor }">
          <TextEditorBottom v-model:attachments="attachments" :editor="editor">
            <template #actions-right>
              <Button
                label="Send"
                :disabled="editor.isEmpty"
                theme="gray"
                variant="solid"
                @click="newCommunication"
              />
            </template>
          </TextEditorBottom>
        </template>
      </TextEditor>
    </div>
  </span>
</template>

<script setup lang="ts">
import { onMounted, onUnmounted, ref } from "vue";
import { createDocumentResource, debounce } from "frappe-ui";
import { CUSTOMER_PORTAL_LANDING } from "@/router";
import { socket } from "@/socket";
import { useConfigStore } from "@/stores/config";
import CommunicationItem from "@/components/CommunicationItem.vue";
import TextEditor from "@/components/text-editor/TextEditor.vue";
import TextEditorBottom from "@/components/text-editor/TextEditorBottom.vue";
import TopBar from "@/components/TopBar.vue";
import IconCheck from "~icons/lucide/check";
import IconDot from "~icons/lucide/dot";
import IconRepeat from "~icons/lucide/repeat-2";

const props = defineProps({
  ticketId: {
    type: String,
    required: true,
  },
});

const configStore = useConfigStore();
const ticket = createDocumentResource({
  doctype: "HD Ticket",
  name: props.ticketId,
  auto: true,
  onSuccess() {
    configStore.setTitle(ticket.doc?.subject);
  },
  whitelistedMethods: {
    getCommunications: {
      method: "get_communications",
      auto: true,
    },
    newCommunication: {
      method: "create_communication_via_contact",
      onSuccess() {
        clearEditor();
      },
    },
    reopen: "reopen",
    resolve: "resolve",
  },
});

const textEditor = ref(null);
const placeholder = "Type a message";
const editorContent = ref("");
const attachments = ref([]);

const newCommunication = debounce(() => {
  const message = editorContent.value;
  const attachmentNames = Array.from(attachments.value).map((a) => a.name);

  ticket.newCommunication.submit({
    message,
    attachments: attachmentNames,
  });
}, 500);

function clearEditor() {
  textEditor.value?.editor.commands.clearContent();
  attachments.value = [];
}

onMounted(() => {
  socket.on("helpdesk:new-communication", (data) => {
    if (data.ticket_id === parseInt(props.ticketId))
      ticket.getCommunications.reload();
  });
});

onUnmounted(() => {
  configStore.setTitle();
  socket.off("helpdesk:new-communication");
});
</script>
