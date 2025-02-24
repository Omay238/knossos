<template>
  <div>
    <Modal
      ref="modalSubmit"
      :header="isRejected(project) ? 'Resubmit for review' : 'Submit for review'"
    >
      <div class="modal-submit universal-body">
        <span>
          You're submitting <span class="project-title">{{ project.title }}</span> to be reviewed
          again by the moderators.
        </span>
        <span>
          Make sure you have addressed the comments from the moderation team.
          <span class="known-errors">
            Repeated submissions without addressing the moderators' comments may result in an
            account suspension.
          </span>
        </span>
        <Checkbox
          v-model="submissionConfirmation"
          description="Confirm I have addressed the messages from the moderators"
        >
          I confirm that I have properly addressed the moderators' comments.
        </Checkbox>
        <div class="input-group push-right">
          <button
            class="iconified-button moderation-button"
            :disabled="!submissionConfirmation"
            @click="resubmit()"
          >
            <ModerationIcon /> Resubmit for review
          </button>
        </div>
      </div>
    </Modal>
    <div v-if="cosmetics.developerMode" class="thread-id">
      Thread ID: <CopyCode :text="thread.id" />
    </div>
    <div v-if="sortedMessages.length > 0" class="messages universal-card recessed">
      <ThreadMessage
        v-for="message in sortedMessages"
        :key="'message-' + message.id"
        :thread="thread"
        :message="message"
        :members="members"
        :report="report"
        raised
      />
    </div>
    <span v-if="report && report.closed">
      This thread is closed and new messages cannot be sent to it.
    </span>
    <template v-else-if="!report || !report.closed">
      <div class="resizable-textarea-wrapper">
        <Chips v-model="replyViewMode" class="chips" :items="['source', 'preview']" />
        <textarea
          v-if="replyViewMode === 'source'"
          v-model="replyBody"
          :placeholder="sortedMessages.length > 0 ? 'Reply to thread...' : 'Send a message...'"
        />
        <div v-else class="markdown-body preview" v-html="renderString(replyBody)" />
      </div>
      <div class="input-group">
        <button
          v-if="sortedMessages.length > 0"
          class="iconified-button brand-button"
          :disabled="!replyBody"
          @click="sendReply()"
        >
          <ReplyIcon /> Reply
        </button>
        <button
          v-else
          class="iconified-button brand-button"
          :disabled="!replyBody"
          @click="sendReply()"
        >
          <SendIcon /> Send
        </button>
        <template v-if="currentMember && !isStaff(auth.user)">
          <template v-if="isRejected(project)">
            <button
              v-if="replyBody"
              class="iconified-button moderation-button"
              @click="openResubmitModal(true)"
            >
              <ModerationIcon /> Resubmit for review with reply
            </button>
            <button
              v-else
              class="iconified-button moderation-button"
              @click="openResubmitModal(false)"
            >
              <ModerationIcon /> Resubmit for review
            </button>
          </template>
        </template>
        <div class="spacer"></div>
        <div class="input-group extra-options">
          <template v-if="report">
            <template v-if="isStaff(auth.user)">
              <button
                v-if="replyBody"
                class="iconified-button danger-button"
                @click="closeReport(true)"
              >
                <CloseIcon /> Close with reply
              </button>
              <button v-else class="iconified-button danger-button" @click="closeReport()">
                <CloseIcon /> Close thread
              </button>
            </template>
          </template>
          <template v-if="project">
            <template v-if="isStaff(auth.user)">
              <button
                v-if="replyBody"
                class="iconified-button brand-button"
                :disabled="isApproved(project)"
                @click="sendReply(requestedStatus)"
              >
                <CheckIcon /> Approve with reply
              </button>
              <button
                v-else
                class="iconified-button brand-button"
                :disabled="isApproved(project)"
                @click="setStatus(requestedStatus)"
              >
                <CheckIcon /> Approve project
              </button>
              <button
                v-if="replyBody"
                class="iconified-button moderation-button"
                :disabled="project.status === 'withheld'"
                @click="sendReply('withheld')"
              >
                <EyeOffIcon /> Withhold with reply
              </button>
              <button
                v-else
                class="iconified-button moderation-button"
                :disabled="project.status === 'withheld'"
                @click="setStatus('withheld')"
              >
                <EyeOffIcon /> Withhold project
              </button>
              <button
                v-if="replyBody"
                class="iconified-button danger-button"
                :disabled="project.status === 'rejected'"
                @click="sendReply('rejected')"
              >
                <CrossIcon /> Reject with reply
              </button>
              <button
                v-else
                class="iconified-button danger-button"
                :disabled="project.status === 'rejected'"
                @click="setStatus('rejected')"
              >
                <CrossIcon /> Reject project
              </button>
            </template>
          </template>
        </div>
      </div>
    </template>
  </div>
</template>

<script setup>
import Chips from '~/components/ui/Chips.vue'
import CopyCode from '~/components/ui/CopyCode.vue'
import ReplyIcon from '~/assets/images/utils/reply.svg'
import SendIcon from '~/assets/images/utils/send.svg'
import CloseIcon from '~/assets/images/utils/check-circle.svg'
import CrossIcon from '~/assets/images/utils/x.svg'
import EyeOffIcon from '~/assets/images/utils/eye-off.svg'
import CheckIcon from '~/assets/images/utils/check.svg'
import ModerationIcon from '~/assets/images/sidebar/admin.svg'
import { renderString } from '~/helpers/parse.js'
import ThreadMessage from '~/components/ui/thread/ThreadMessage.vue'
import { isStaff } from '~/helpers/users.js'
import { isApproved, isRejected } from '~/helpers/projects.js'
import Modal from '~/components/ui/Modal.vue'
import Checkbox from '~/components/ui/Checkbox.vue'

const props = defineProps({
  thread: {
    type: Object,
    required: true,
  },
  report: {
    type: Object,
    required: false,
    default: null,
  },
  project: {
    type: Object,
    required: false,
    default: null,
  },
  updateThread: {
    type: Function,
    required: false,
    default: () => {},
  },
  setStatus: {
    type: Function,
    required: false,
    default: () => {},
  },
  currentMember: {
    type: Object,
    default() {
      return null
    },
  },
  auth: {
    type: Object,
    required: true,
  },
})
const app = useNuxtApp()
const cosmetics = useCosmetics()

const members = computed(() => {
  const members = {}
  for (const member of props.thread.members) {
    members[member.id] = member
  }
  return members
})

const replyViewMode = ref('source')
const replyBody = ref('')

const sortedMessages = computed(() => {
  if (props.thread !== null) {
    return props.thread.messages
      .slice()
      .sort((a, b) => app.$dayjs(a.created) - app.$dayjs(b.created))
  }
  return []
})

const modalSubmit = ref(null)

async function updateThreadLocal() {
  let threadId = null
  if (props.project) {
    threadId = props.project.thread_id
  } else if (props.report) {
    threadId = props.report.thread_id
  }
  let thread = null
  if (threadId) {
    thread = await useBaseFetch(`thread/${threadId}`)
  }
  props.updateThread(thread)
}

async function sendReply(status = null) {
  try {
    await useBaseFetch(`thread/${props.thread.id}`, {
      method: 'POST',
      body: {
        body: {
          type: 'text',
          body: replyBody.value,
        },
      },
    })
    replyBody.value = ''
    await updateThreadLocal()
    if (status !== null) {
      props.setStatus(status)
    }
  } catch (err) {
    app.$notify({
      group: 'main',
      title: 'Error sending message',
      text: err.data ? err.data.description : err,
      type: 'error',
    })
  }
}

async function closeReport(reply) {
  if (reply) {
    await sendReply()
  }

  try {
    await useBaseFetch(`report/${props.report.id}`, {
      method: 'PATCH',
      body: {
        closed: true,
      },
    })
    await updateThreadLocal()
  } catch (err) {
    app.$notify({
      group: 'main',
      title: 'Error closing report',
      text: err.data ? err.data.description : err,
      type: 'error',
    })
  }
}

const replyWithSubmission = ref(false)
const submissionConfirmation = ref(false)

function openResubmitModal(reply) {
  submissionConfirmation.value = false
  replyWithSubmission.value = reply
  modalSubmit.value.show()
}

async function resubmit() {
  if (replyWithSubmission.value) {
    await sendReply('processing')
  } else {
    await props.setStatus('processing')
  }
  modalSubmit.value.hide()
}

const requestedStatus = computed(() => props.project.requested_status ?? 'approved')
</script>

<style lang="scss" scoped>
.messages {
  display: flex;
  flex-direction: column;
  padding: var(--spacing-card-md);
}

.resizable-textarea-wrapper {
  margin-bottom: var(--spacing-card-sm);

  textarea {
    padding: var(--spacing-card-bg);
    width: 100%;
  }

  .chips {
    margin-bottom: var(--spacing-card-md);
  }

  .preview {
    overflow-y: auto;
  }
}

.thread-id {
  margin-bottom: var(--spacing-card-md);
  font-weight: bold;
  color: var(--color-heading);
}

.input-group {
  .spacer {
    flex-grow: 1;
    flex-shrink: 1;
  }

  .extra-options {
    flex-basis: fit-content;
  }
}

.modal-submit {
  padding: var(--spacing-card-bg);
  display: flex;
  flex-direction: column;
  gap: var(--spacing-card-lg);

  .project-title {
    font-weight: bold;
  }
}
</style>
