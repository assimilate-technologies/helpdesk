<template>
  <div>
    <Dialog :options="{ title: 'Add Agents' }" :show="show" @close="close()">
      <template #body-content>
        <div class="space-y-3">
          <form
            @submit.prevent="onSubmit"
            class="flex flex-row items-center space-x-2"
          >
            <Input
              id="searchInput"
              class="w-full"
              type="text"
              v-model="searchInput"
              placeholder="Type emails"
              @input="(val) => onSearchInputChange(val)"
            />
            <Button
              appearance="primary"
              type="submit"
              :disabled="!currentInputIsValidEmail"
              @click="
                () => {
                  addToInviteQueue(searchInput);
                  clearSearchInput();
                }
              "
            >
              Add
            </Button>
          </form>
          <div
            class="flex max-h-[300px] min-h-[100px] flex-col overflow-y-auto rounded border bg-gray-100 px-2"
            v-if="inviteQueue.length"
          >
            <ul class="flex flex-wrap gap-2 py-2">
              <li
                class="flex items-center space-x-2 rounded bg-white p-1 shadow"
                v-for="email in inviteQueue.slice().reverse()"
                :key="email"
                :title="email"
              >
                <span class="ml-2 text-base">
                  {{ email }}
                </span>
                <button
                  class="grid h-4 w-4 place-items-center rounded text-gray-700 hover:bg-gray-300"
                  @click="removeEmailFromQueue(email)"
                >
                  <FeatherIcon class="w-3" name="x" />
                </button>
              </li>
            </ul>
          </div>
        </div>
      </template>
      <template #actions v-if="inviteQueue.length">
        <Button
          :disabled="inviteQueue.length == 0"
          appearance="primary"
          @click="sentInvites()"
          class="mr-2"
          :loading="$resources.sentInvites.loading"
          >Send Invites</Button
        >
        <Button appearance="secondary" class="mr-2" @click="close()"
          >Cancel</Button
        >
        <div class="grow mt-2">
          <Button
            @click="removeAllEmailFromQueue()"
            v-if="inviteQueue.length > 1"
          >
            Clear All
          </Button>
        </div>
      </template>
    </Dialog>
  </div>
</template>

<script>
import { useAuthStore } from "@/stores/auth";
import { ref } from "@vue/reactivity";
import { Dialog, FeatherIcon, Input } from "frappe-ui";
import { useOnboarding } from "frappe-ui/frappe";

export default {
  name: "AddNewAgentsDialog",
  props: ["show"],
  components: {
    Dialog,
    Input,
    FeatherIcon,
  },
  setup() {
    const searchInput = ref("");
    const inviteQueue = ref([]);

    const currentInputIsValidEmail = ref(false);
    const { updateOnboardingStep } = useOnboarding("helpdesk");
    const { isManager } = useAuthStore();
    return {
      searchInput,
      inviteQueue,
      currentInputIsValidEmail,
      updateOnboardingStep,
      isManager,
    };
  },
  methods: {
    testEmailRegex(val) {
      let emailRegex = /^[\w-\.]+@([\w-]+\.)+[\w-]{2,4}$/;
      return emailRegex.test(val);
    },
    onSearchInputChange(val) {
      val = val.replaceAll(" ", "");

      if (val == "") {
        document.getElementById("searchInput").value = "";
        return;
      }

      const valStr = val;
      const inputs = val.split(",");

      let clearInputFlag = false;
      this.currentInputIsValidEmail = false;
      inputs.forEach((input) => {
        if (this.testEmailRegex(input)) {
          if (inputs.length > 1) {
            this.addToInviteQueue(input);
            clearInputFlag = true;
          } else {
            if (valStr.includes(",")) {
              this.addToInviteQueue(input);
              clearInputFlag = true;
            } else {
              this.currentInputIsValidEmail = true;
            }
          }
        }
      });
      if (clearInputFlag) {
        this.clearSearchInput();
      }
    },
    addToInviteQueue(email) {
      this.inviteQueue = [...new Set([...this.inviteQueue, email])];
    },
    removeEmailFromQueue(email) {
      this.inviteQueue = this.inviteQueue.filter((item) => item !== email);
    },
    removeAllEmailFromQueue() {
      this.inviteQueue = [];
    },
    clearSearchInput() {
      this.currentInputIsValidEmail = false;
      this.searchInput = "";

      const input = document.getElementById("searchInput");
      input.value = "";
      input.focus();
    },
    close() {
      this.searchInput = "";
      this.inviteQueue = [];
      this.$emit("close");
    },
    sentInvites() {
      this.$resources.sentInvites.submit({
        emails: this.inviteQueue,
      });
    },
  },
  resources: {
    sentInvites() {
      return {
        url: "helpdesk.api.agent.sent_invites",
        onSuccess: (res) => {
          this.currentInputIsValidEmail = false;
          this.searchInput = "";
          this.inviteQueue = [];
          if (this.isManager) {
            this.updateOnboardingStep("invite_agents");
          }

          this.$toast({
            title: "Invites Sent Successfully!",
            icon: "check",
            iconClasses: "text-green-500",
          });

          this.close();
        },
      };
    },
  },
};
</script>
