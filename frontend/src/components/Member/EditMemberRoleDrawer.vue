<template>
  <Drawer @close="$emit('close')">
    <DrawerContent
      class="w-[40rem] max-w-[100vw]"
      :closable="true"
      :title="$t('settings.members.grant-access')"
    >
      <template #default>
        <div class="space-y-4">
          <MembersBindingSelect
            v-if="isCreating"
            v-model:value="state.memberList"
            :required="true"
            :include-all-users="true"
            :include-service-account="true"
          />
          <div v-else class="w-full space-y-2">
            <div class="flex items-center gap-x-1">
              {{ $t("common.email") }}
              <span class="text-red-600">*</span>
            </div>
            <EmailInput :readonly="true" :value="email" />
          </div>

          <div class="w-full space-y-2">
            <div class="flex items-center gap-x-1 text-main">
              {{ $t("settings.members.assign-role", 2 /* multiply*/) }}
              <span class="text-red-600">*</span>
            </div>
            <RoleSelect v-model:value="state.roles" :multiple="true" />
          </div>
        </div>
      </template>

      <template #footer>
        <div class="w-full flex justify-between items-center">
          <div>
            <BBButtonConfirm
              v-if="!isCreating"
              ref="confirmRevokeAccessRef"
              :type="'DELETE'"
              :confirm-title="$t('settings.members.revoke-access-alert')"
              :ok-text="$t('settings.members.revoke-access')"
              :button-text="$t('settings.members.revoke-access')"
              :require-confirm="true"
              @confirm="handleRevoke"
            />
          </div>

          <div class="flex flex-row items-center justify-end gap-x-3">
            <NButton @click="$emit('close')">
              {{ $t("common.cancel") }}
            </NButton>
            <NButton
              type="primary"
              :disabled="!allowConfirm"
              :loading="state.isRequesting"
              @click="updateRoleBinding"
            >
              {{ $t("common.confirm") }}
            </NButton>
          </div>
        </div>
      </template>
    </DrawerContent>
  </Drawer>
</template>

<script setup lang="ts">
import { isEqual } from "lodash-es";
import { NButton } from "naive-ui";
import { computed, reactive, ref } from "vue";
import { useI18n } from "vue-i18n";
import { BBButtonConfirm } from "@/bbkit";
import EmailInput from "@/components/EmailInput.vue";
import { Drawer, DrawerContent } from "@/components/v2";
import { RoleSelect } from "@/components/v2/Select";
import {
  extractGroupEmail,
  extractUserId,
  pushNotification,
  useWorkspaceV1Store,
} from "@/store";
import { PresetRoleType } from "@/types";
import MembersBindingSelect from "./MembersBindingSelect.vue";
import { type MemberBinding } from "./types";

interface LocalState {
  isRequesting: boolean;
  memberList: string[];
  roles: string[];
}

const props = defineProps<{
  member?: MemberBinding;
}>();

const emit = defineEmits<{
  (event: "close"): void;
}>();

const initMemberList = () => {
  if (!props.member) {
    return [];
  }
  return [props.member.binding];
};

const state = reactive<LocalState>({
  isRequesting: false,
  memberList: initMemberList(),
  roles: [...(props.member?.workspaceLevelRoles ?? new Set<string>())],
});

const { t } = useI18n();
const workspaceStore = useWorkspaceV1Store();
const confirmRevokeAccessRef = ref<InstanceType<typeof BBButtonConfirm>>();

const isCreating = computed(() => !props.member);

const email = computed(() => {
  if (!props.member) {
    return "";
  }
  if (props.member.type === "users") {
    return extractUserId(props.member.binding);
  }
  return extractGroupEmail(props.member.binding);
});

const allowConfirm = computed(() => {
  if (state.memberList.length === 0) {
    return false;
  }

  if (!isCreating.value) {
    return !isEqual(props.member?.workspaceLevelRoles, state.roles);
  }

  return state.roles.length !== 0;
});

const memberListInBinding = computed(() => {
  if (props.member) {
    return [props.member.binding];
  }
  return state.memberList;
});

const updateRoleBinding = async () => {
  if (state.roles.length === 0) {
    confirmRevokeAccessRef.value?.showAlert();
    return;
  }

  const batchPatch = [];
  if (props.member) {
    batchPatch.push({
      member: props.member.binding,
      roles: [...state.roles],
    });
  } else {
    for (const member of memberListInBinding.value) {
      const existedRoles = workspaceStore.findRolesByMember({
        member,
        ignoreGroup: true,
      });
      batchPatch.push({
        member,
        roles: [...new Set([...state.roles, ...existedRoles])],
      });
    }
  }
  await workspaceStore.patchIamPolicy(batchPatch);
  pushNotification({
    module: "bytebase",
    style: "SUCCESS",
    title: t("common.updated"),
  });
  emit("close");
};

const handleRevoke = async () => {
  if (!props.member || memberListInBinding.value.length !== 1) {
    return;
  }
  await workspaceStore.patchIamPolicy([
    {
      member: memberListInBinding.value[0],
      // TODO(ed): no default member role.
      roles: [PresetRoleType.WORKSPACE_MEMBER],
    },
  ]);
  pushNotification({
    module: "bytebase",
    style: "INFO",
    title: t("settings.members.revoked"),
  });
  emit("close");
};
</script>
