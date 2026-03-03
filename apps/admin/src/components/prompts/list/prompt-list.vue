<template>
  <v-expansion-panel :value="isOverrideMode ? 'override' : section">
    <v-expansion-panel-title>
      <div class="d-flex flex-row justify-content-space-between">
        <v-avatar class="mr-3" color="primary" size="28">
          <span class="text-white font-weight-medium text-h6">{{ step }}</span>
        </v-avatar>
        <div class="d-flex flex-column">
          <div class="text-h6">
            {{ title }}
          </div>
          <div class="text-subtitle-2">
            {{ subtitle }}
          </div>
        </div>
      </div>
      <template #actions="{ expanded }">
        <div class="d-flex align-center">
          <template v-if="expanded">
            <v-btn
              v-if="!isOverrideMode"
              color="primary"
              icon="$add"
              size="small"
              :title="$t('survey-schemes.prompts.create')"
              @click.stop="create"
            />
            <options-menu>
              <load-prompt-dialog
                :items="isOverrideMode ? templates : undefined"
                :prompt-ids="promptIds"
                :scheme-id="$route.params.id.toString()"
                @load="load"
              />
              <json-editor-dialog v-model="prompts" />
            </options-menu>
          </template>
          <v-icon class="ms-2" :icon="expanded ? '$expand' : '$collapse'" />
        </div>
      </template>
    </v-expansion-panel-title>
    <v-expansion-panel-text>
      <div v-if="!isOverrideMode" class="d-flex justify-end mb-3">
        <v-btn
          color="primary"
          prepend-icon="$add"
          size="small"
          variant="outlined"
          @click="addGroup"
        >
          {{ `${$t('common.action.add')} ${$t('survey-schemes.prompts.internal.group._').toLowerCase()}` }}
        </v-btn>
      </div>

      <vue-draggable
        v-model="groupsState"
        :animation="300"
        handle=".prompt-group__handle"
      >
        <div
          v-for="(group, groupIndex) in groupsState"
          :key="groupKey(groupIndex)"
          class="prompt-group mb-3"
        >
          <div class="prompt-group__title d-flex align-center px-3 py-2">
            <v-avatar class="prompt-group__handle drag-and-drop__handle mr-3" icon="$handle" size="26" />
            <v-text-field
              v-if="!isOverrideMode"
              v-model="group.name"
              class="prompt-group__name mr-2"
              density="compact"
              hide-details
              variant="underlined"
            />
            <span v-else class="text-subtitle-2 font-weight-medium">
              {{ group.name }}
            </span>
            <v-spacer />
            <v-btn
              v-if="!isOverrideMode"
              color="primary"
              icon="$add"
              size="x-small"
              :title="$t('survey-schemes.prompts.create')"
              @click.stop="create(groupIndex)"
            />
            <v-btn
              v-if="!isOverrideMode && !group.prompts.length"
              color="error"
              icon="$delete"
              size="x-small"
              variant="text"
              @click.stop="removeGroup(groupIndex)"
            />
            <v-btn
              color="secondary"
              :icon="group.expanded ? '$expand' : '$collapse'"
              size="x-small"
              variant="text"
              @click.stop="toggleGroup(groupIndex)"
            />
          </div>

          <v-expand-transition>
            <div v-show="group.expanded">
              <v-list class="list-border" lines="two">
                <vue-draggable
                  v-model="group.prompts"
                  :animation="300"
                  :group="{ name: `prompt-group-${section}` }"
                  handle=".drag-and-drop__handle"
                >
                  <prompt-list-item
                    v-for="(prompt, index) in group.prompts"
                    :key="`${prompt.id}:${prompt.name}`"
                    v-bind="{
                      errors: errors.get(`prompts.${fullSection}.${globalIndex(groupIndex, index)}.*`),
                      mode,
                      prompt,
                      index,
                      templates,
                    }"
                    :move-sections="moveSections(prompt)"
                    @prompt-copy="copy(groupIndex, $event)"
                    @prompt-edit="edit(groupIndex, $event)"
                    @prompt-move="move(groupIndex, $event)"
                    @prompt-remove="remove(groupIndex, $event)"
                    @prompt-sync="sync(groupIndex, $event)"
                  />
                </vue-draggable>
              </v-list>
            </div>
          </v-expand-transition>
        </div>
      </vue-draggable>

      <prompt-selector ref="selector" v-bind="{ mode, section, promptIds }" @save="save" />
    </v-expansion-panel-text>
  </v-expansion-panel>
  <v-divider />
</template>

<script lang="ts" setup>
import type { PropType } from 'vue';

import type { ReturnUseErrors } from '@intake24/admin/composables';
import type { SinglePrompt } from '@intake24/common/prompts';
import type {
  MealSection,
  PromptSection,
  PromptSubsection,
  SurveyPromptSection,
} from '@intake24/common/surveys';

import { deepEqual } from 'fast-equals';
import { computed, ref, watch } from 'vue';
import { VueDraggable } from 'vue-draggable-plus';

import { OptionsMenu } from '@intake24/admin/components/dialogs';
import { JsonEditorDialog } from '@intake24/admin/components/editors';
import { promptSettings } from '@intake24/admin/components/prompts';
import { isMealSection } from '@intake24/common/surveys';
import { copy as copyObject } from '@intake24/common/util';
import { useI18n } from '@intake24/ui';

import PromptSelector from '../prompt-selector.vue';
import LoadPromptDialog from './load-prompt-dialog.vue';
import PromptListItem from './prompt-list-item.vue';

export type MoveSection = { value: string; title: string };

export type PromptEvent = {
  index: number;
  prompt: SinglePrompt;
};

type PromptGroup = {
  name: string;
  expanded: boolean;
  prompts: SinglePrompt[];
};

export interface PromptMoveEvent extends PromptEvent {
  section: MealSection | SurveyPromptSection;
}

defineOptions({ name: 'PromptList' });

const props = defineProps({
  errors: {
    type: Object as PropType<ReturnUseErrors>,
    required: true,
  },
  mode: {
    type: String as PropType<'full' | 'override'>,
    default: 'full',
  },
  section: {
    type: String as PropType<PromptSection>,
  },
  step: {
    type: Number,
    default: 1,
  },
  promptIds: {
    type: Array as PropType<string[]>,
    default: () => [],
  },
  templates: {
    type: Array as PropType<SinglePrompt[]>,
    default: () => [],
  },
  subsections: {
    type: Array as PropType<PromptSubsection[]>,
    default: () => [],
  },
  modelValue: {
    type: Array as PropType<SinglePrompt[]>,
    required: true,
  },
});

const emit = defineEmits(['update:modelValue', 'update:groups', 'move']);

const { i18n } = useI18n();

const selector = ref<InstanceType<typeof PromptSelector>>();
const pendingGroupIndex = ref<number | null>(null);

const isOverrideMode = computed(() => props.mode === 'override');
const title = computed(() => i18n.t(
  isOverrideMode.value
    ? `survey-schemes.overrides.prompts.title`
    : `survey-schemes.prompts.${props.section}.title`,
));
const subtitle = computed(() => i18n.t(
  isOverrideMode.value
    ? `survey-schemes.overrides.prompts.subtitle`
    : `survey-schemes.prompts.${props.section}.subtitle`,
));
const fullSection = computed(() => isMealSection(props.section) ? `meals.${props.section}` : props.section);
const groupsState = ref<PromptGroup[]>(buildGroups(copyObject(props.modelValue), copyObject(props.subsections)));
const prompts = computed<SinglePrompt[]>({
  get: () => flattenGroups(groupsState.value),
  set: (value) => {
    groupsState.value = buildGroups(copyObject(value), copyObject(props.subsections));
  },
});

function defaultGroupName(index: number) {
  return `${i18n.t('survey-schemes.prompts.internal.group._')} ${index + 1}`;
}

function groupKey(index: number) {
  return `${props.section}:${index}`;
}

function buildGroups(prompts: SinglePrompt[], groups: PromptSubsection[]): PromptGroup[] {
  if (isOverrideMode.value)
    return [{ name: defaultGroupName(0), expanded: true, prompts: [...prompts] }];

  if (!groups.length)
    return [{ name: defaultGroupName(0), expanded: true, prompts: [...prompts] }];

  let offset = 0;
  const items = groups.map((item, index) => {
    const size = Math.max(0, item.size || 0);
    const groupPrompts = prompts.slice(offset, offset + size);
    offset += size;

    return {
      name: item.name || defaultGroupName(index),
      expanded: item.expanded ?? true,
      prompts: groupPrompts,
    };
  });

  if (offset < prompts.length)
    items.push({ name: defaultGroupName(items.length), expanded: true, prompts: prompts.slice(offset) });

  return items.length ? items : [{ name: defaultGroupName(0), expanded: true, prompts: [...prompts] }];
}

function flattenGroups(groups: PromptGroup[]) {
  return groups.flatMap(group => group.prompts);
}

function serializedGroups(groups: PromptGroup[]): PromptSubsection[] {
  return groups.map((group, index) => ({
    id: groupKey(index),
    size: group.prompts.length,
    expanded: group.expanded,
    name: group.name,
  }));
}

function locate(groupIndex: number, index: number) {
  const group = groupsState.value[groupIndex];
  if (!group)
    return;

  return { group, index };
}

function globalIndex(groupIndex: number, index: number) {
  let offset = 0;
  for (let idx = 0; idx < groupsState.value.length; idx++) {
    if (idx === groupIndex)
      return offset + index;

    offset += groupsState.value[idx].prompts.length;
  }

  return index;
}

function ensureGroup(targetGroupIndex?: number | null) {
  if (typeof targetGroupIndex === 'number') {
    const item = groupsState.value[targetGroupIndex];
    if (item)
      return item;
  }

  if (!groupsState.value.length)
    groupsState.value.push({ name: defaultGroupName(0), expanded: true, prompts: [] });

  return groupsState.value[groupsState.value.length - 1];
};

function clearErrors(index?: number) {
  props.errors.clear(typeof index === 'undefined' ? `prompts.${fullSection.value}.*` : `prompts.${fullSection.value}.${index}.*`);
}

function create(groupIndex?: number) {
  if (isOverrideMode.value)
    return;

  pendingGroupIndex.value = typeof groupIndex === 'number' ? groupIndex : null;
  selector.value?.create();
};

function load(prompt: SinglePrompt) {
  clearErrors();
  ensureGroup().prompts.push(prompt);
};

function copy(groupIndex: number, { prompt, index }: PromptEvent) {
  const target = locate(groupIndex, index);
  if (!target)
    return;

  target.group.prompts.splice(index + 1, 0, { ...copyObject(prompt), id: `${prompt.id}-copy`, name: `${prompt.name} (copy)` });
};

function edit(groupIndex: number, { prompt, index }: PromptEvent) {
  const itemIndex = globalIndex(groupIndex, index);
  const errors = props.errors.get(`prompts.${fullSection.value}.${itemIndex}.*`);
  selector.value?.edit(itemIndex, prompt, errors);
};

function save({ prompt, index }: PromptEvent) {
  if (index === -1) {
    ensureGroup(pendingGroupIndex.value).prompts.push(prompt);
    pendingGroupIndex.value = null;
  }
  else {
    let offset = 0;
    for (const group of groupsState.value) {
      if (index < offset + group.prompts.length) {
        group.prompts.splice(index - offset, 1, prompt);
        break;
      }

      offset += group.prompts.length;
    }
  }

  clearErrors(index);
};

function moveSections(prompt: SinglePrompt): MoveSection[] {
  return promptSettings[prompt.component].sections
    .filter(item => item !== props.section)
    .map(item => ({
      value: item,
      title: i18n.t(`survey-schemes.prompts.${item}.title`),
    }));
};

function move(groupIndex: number, event: PromptMoveEvent) {
  if (isOverrideMode.value)
    return;

  const target = locate(groupIndex, event.index);
  if (!target)
    return;

  emit('move', { ...event, index: globalIndex(groupIndex, event.index) });
  target.group.prompts.splice(event.index, 1);
};

function remove(groupIndex: number, index: number) {
  const target = locate(groupIndex, index);
  if (!target)
    return;

  clearErrors(globalIndex(groupIndex, index));
  target.group.prompts.splice(index, 1);
};

function sync(groupIndex: number, { prompt, index }: PromptEvent) {
  if (isOverrideMode.value)
    return;

  const target = locate(groupIndex, index);
  if (!target)
    return;

  target.group.prompts.splice(index, 1, prompt);
};

function addGroup() {
  groupsState.value.push({ name: defaultGroupName(groupsState.value.length), expanded: true, prompts: [] });
}

function removeGroup(groupIndex: number) {
  const group = groupsState.value[groupIndex];
  if (!group || group.prompts.length)
    return;

  groupsState.value.splice(groupIndex, 1);
}

function toggleGroup(groupIndex: number) {
  const group = groupsState.value[groupIndex];
  if (!group)
    return;

  group.expanded = !group.expanded;
}

function update() {
  const prompts = flattenGroups(groupsState.value);
  if (!deepEqual(prompts, props.modelValue))
    emit('update:modelValue', prompts);

  const groups = serializedGroups(groupsState.value);
  if (!deepEqual(groups, props.subsections))
    emit('update:groups', groups);
};

function syncFromProps() {
  if (deepEqual(props.modelValue, flattenGroups(groupsState.value)) && deepEqual(props.subsections, serializedGroups(groupsState.value)))
    return;

  groupsState.value = buildGroups(copyObject(props.modelValue), copyObject(props.subsections));
}

watch(() => props.modelValue, syncFromProps, { deep: true });
watch(() => props.subsections, syncFromProps, { deep: true });

watch(groupsState, () => {
  update();
}, { deep: true });
</script>

<style lang="scss">
.prompt-group {
  border: 1px solid rgba(var(--v-theme-on-surface), 0.12);
  border-radius: 4px;
}

.prompt-group__title {
  background: rgba(var(--v-theme-on-surface), 0.03);
}

.prompt-group__name {
  max-width: 260px;
}

.prompt-group__handle {
  cursor: grab;
}
</style>
