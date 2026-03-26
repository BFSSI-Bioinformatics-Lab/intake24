<template>
  <v-tabs-window-item value="options">
    <v-row class="mb-2">
      <v-col cols="12">
        <v-switch
          :disabled="updateFood"
          hide-details="auto"
          :label="$t('survey-schemes.prompts.other')"
          :model-value="other"
          @update:model-value="update('other', $event)"
        />
      </v-col>
      <v-col cols="12">
        <v-switch
          v-if="showUpdateFoodSwitch"
          :disabled="!validationRequired"
          hide-details="auto"
          :label="$t('survey-schemes.prompts.updateFood')"
          :model-value="updateFood"
          @update:model-value="update('updateFood', $event)"
        />
      </v-col>
    </v-row>
    <language-selector
      border
      :default="[]"
      :label="$t('common.options.title')"
      :model-value="options"
      :required="true"
      @update:model-value="update('options', $event)"
    >
      <template v-for="lang in Object.keys(options)" :key="lang" #[`lang.${lang}`]>
        <options-list
          exclusive
          :options="options[lang]"
          @update:options="updateLanguage('options', lang, $event)"
        />

        <v-row v-if="showUpdateFoodConfig" class="mt-3">
          <v-col cols="12">
            <v-btn
              class="my-4 ignore-item"
              color="primary"
              :disabled="!subsetsStale"
              @click="handleClick"
            >
              {{ $t('survey-schemes.prompts.checkbox-list-prompt.refreshSubsets') }}
            </v-btn>
            <v-alert type="info" variant="tonal">
              {{ $t('survey-schemes.prompts.checkbox-list-prompt.updateFoodOptions') }}
            </v-alert>
          </v-col>
          <v-col cols="12">
            <v-switch
              hide-details="auto"
              :label="$t('survey-schemes.prompts.checkbox-list-prompt.updateFoodDefaultOptionSwitch')"
              :model-value="updateFoodDefaultOption[lang] ?? false"
              @update:model-value="updateLanguage('updateFoodDefaultOption', lang, $event)"
            />
          </v-col>
          <v-col v-if="updateFoodDefaultOption[lang]" cols="12">
            <v-text-field
              hide-details="auto"
              :label="$t('survey-schemes.prompts.checkbox-list-prompt.updateFoodDefaultOptionField')"
              :model-value="updateFoodDefaultOptionValue[lang] ?? ''"
              variant="outlined"
              @update:model-value="updateDefaultOptionValue(lang, $event)"
            />
          </v-col>
          <v-col
            v-for="[key, value] in Object.entries(currentUpdateFoodOptions[lang] ?? {})"
            :key="`${lang}-${key}`"
            cols="12"
          >
            <v-text-field
              :disabled="subsetsStale"
              hide-details="auto"
              :label="`${value.label}`"
              :model-value="value.foodcode"
              variant="outlined"
              @update:model-value="updateCurrentFoodCode(lang, key, $event)"
            />
          </v-col>
        </v-row>
      </template>
    </language-selector>
  </v-tabs-window-item>
</template>

<script lang="ts">
import type { PropType } from 'vue';

import type { Prompts } from '@intake24/common/prompts';
import type { PromptSection } from '@intake24/common/surveys';
import type { ListOption } from '@intake24/common/types';

import { defineComponent } from 'vue';

import { selectListPrompt } from '../partials';

export default defineComponent({
  name: 'CheckboxListPrompt',

  mixins: [selectListPrompt],

  props: {
    updateFood: {
      type: Boolean as PropType<Prompts['checkbox-list-prompt']['updateFood']>,
      required: true,
    },
    updateFoodOptions: {
      type: Object as PropType<Prompts['checkbox-list-prompt']['updateFoodOptions']>,
      required: true,
    },
    updateFoodDefaultOption: {
      type: Object as PropType<Prompts['checkbox-list-prompt']['updateFoodDefaultOption']>,
      required: true,
    },
    updateFoodDefaultOptionValue: {
      type: Object as PropType<Prompts['checkbox-list-prompt']['updateFoodDefaultOptionValue']>,
      required: true,
    },
    validation: {
      type: Object as PropType<Prompts['checkbox-list-prompt']['validation']>,
      required: true,
    },
    promptSection: {
      type: String as PropType<PromptSection | undefined>,
      default: undefined,
    },
  },

  data() {
    return {
      hasInitializedUpdateFoodOptions: false,
      subsetsStale: false,
      currentUpdateFoodOptions: {} as Record<string, Record<string, { label: string; foodcode: string }>>,
    };
  },

  computed: {
    validationRequired(): boolean {
      return this.validation.required;
    },
    showUpdateFoodSwitch(): boolean {
      return this.promptSection === 'foods';
    },
    canUseUpdateFood(): boolean {
      return this.showUpdateFoodSwitch && this.validationRequired;
    },
    showUpdateFoodConfig(): boolean {
      return this.updateFood && this.canUseUpdateFood;
    },
    optionsFingerprint(): string {
      return JSON.stringify(
        Object.entries(this.options ?? {})
          .sort(([langA], [langB]) => langA.localeCompare(langB))
          .map(([lang, options]) => [
            lang,
            (options ?? []).map(option => ({
              id: option.id ?? null,
              value: option.value ?? null,
              label: option.label,
              exclusive: !!option.exclusive,
            })),
          ]),
      );
    },
  },

  watch: {
    canUseUpdateFood: {
      immediate: true,
      handler(value: boolean) {
        if (!value && this.updateFood)
          this.update('updateFood', false);
      },
    },
    updateFood(value: boolean) {
      if (value)
        this.update('other', false);
    },
    optionsFingerprint: {
      immediate: true,
      handler() {
        if (!this.hasInitializedUpdateFoodOptions) {
          this.currentUpdateFoodOptions = this.buildCurrentUpdateFoodOptions();
          this.hasInitializedUpdateFoodOptions = true;
          this.subsetsStale = false;
          return;
        }

        this.subsetsStale = true;
      },
    },
    currentUpdateFoodOptions: {
      deep: true,
      handler() {
        this.update('updateFoodOptions', Object.fromEntries(
          Object.entries(this.currentUpdateFoodOptions).map(([lang, inner]) => [
            lang,
            Object.fromEntries(
              Object.entries(inner).map(([subsetKey, v]) => [subsetKey, this.normalizeFoodCode(v.foodcode)]),
            ),
          ]),
        ));
      },
    },
  },

  methods: {
    normalizeFoodCode(value: unknown): string {
      if (typeof value === 'string')
        return value;

      return '';
    },
    buildCurrentUpdateFoodOptions() {
      const next: Record<string, Record<string, { label: string; foodcode: string }>> = {};

      for (const [lang, options] of Object.entries(this.options ?? {})) {
        const subsets = this.updateSubsets(options);
        const currentByLanguage = this.currentUpdateFoodOptions[lang] ?? {};
        const existingFromProps = this.updateFoodOptions?.[lang] ?? {};

        next[lang] = Object.fromEntries(subsets.map(subset => [
          subset.key,
          {
            label: subset.label,
            foodcode: this.normalizeFoodCode(currentByLanguage[subset.key]?.foodcode)
              || this.normalizeFoodCode(existingFromProps[subset.key]),
          },
        ]));
      }

      return next;
    },
    handleClick() {
      this.currentUpdateFoodOptions = this.buildCurrentUpdateFoodOptions();
      this.subsetsStale = false;
    },
    updateDefaultOptionValue(lang: string, value: unknown) {
      this.updateLanguage('updateFoodDefaultOptionValue', lang, this.normalizeFoodCode(value));
    },
    updateCurrentFoodCode(lang: string, subsetKey: string, foodcode: unknown) {
      const currentByLanguage = this.currentUpdateFoodOptions[lang] ?? {};
      const currentEntry = currentByLanguage[subsetKey] ?? { label: subsetKey, foodcode: '' };

      this.currentUpdateFoodOptions = {
        ...this.currentUpdateFoodOptions,
        [lang]: {
          ...currentByLanguage,
          [subsetKey]: {
            ...currentEntry,
            foodcode: this.normalizeFoodCode(foodcode),
          },
        },
      };
    },
    updateSubsets(options: ListOption[]): { key: string; label: string }[] {
      const subsets: { key: string; label: string }[] = [];

      for (let mask = 1; mask < (2 ** options.length); mask++) {
        const selectedIndexes: number[] = [];
        for (let i = 0; i < options.length; i++) {
          if ((mask & (1 << i)) !== 0)
            selectedIndexes.push(i);
        }

        const selected = selectedIndexes.map(index => options[index]);
        const hasExclusive = selected.some(option => option.exclusive);

        if (hasExclusive && selected.length > 1)
          continue;

        const selectedIds = selectedIndexes
          .map(index => options[index].id ?? index)
          .sort((left, right) => left - right);

        subsets.push({
          key: selectedIds.join(', '),
          label: selected.map(option => option.label).join(', '),
        });
      }

      return subsets;
    },
  },
});
</script>

<style lang="scss" scoped></style>
