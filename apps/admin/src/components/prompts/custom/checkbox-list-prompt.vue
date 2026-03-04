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
            <v-alert type="info" variant="tonal">
              {{ $t('survey-schemes.prompts.checkbox-list-prompt.updateFoodOptions') }}
            </v-alert>
          </v-col>
          <v-col cols="12">
            <v-switch
              hide-details="auto"
              :label="$t('survey-schemes.prompts.checkbox-list-prompt.updateFoodDefaultOptionSwitch')"
              :model-value="localizedUpdateFoodDefaultOption[lang] ?? false"
              @update:model-value="updateDefaultOptionSwitch(lang, !!$event)"
            />
          </v-col>
          <v-col v-if="localizedUpdateFoodDefaultOption[lang]" cols="12">
            <v-text-field
              hide-details="auto"
              :label="$t('survey-schemes.prompts.checkbox-list-prompt.updateFoodDefaultOptionField')"
              :model-value="localizedUpdateFoodDefaultOptionValue[lang] ?? ''"
              variant="outlined"
              @update:model-value="updateDefaultOptionValue(lang, $event)"
            />
          </v-col>
          <v-col
            v-for="subset in optionSubsetsByLanguage[lang] ?? []"
            :key="`${lang}-${subset.key}`"
            cols="12"
          >
            <v-text-field
              hide-details="auto"
              :label="`${subset.label}`"
              :model-value="subsetCodeValue(lang, subset)"
              variant="outlined"
              @update:model-value="updateSubsetCode(lang, subset, $event)"
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
      optionSubsetsByLanguageCache: {} as Record<string, { key: string; label: string; legacyKey: string }[]>,
      optionsRefreshTimeout: null as ReturnType<typeof setTimeout> | null,
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
    localizedUpdateFoodOptions(): Record<string, Record<string, string>> {
      const next: Record<string, Record<string, string>> = {};

      for (const [key, value] of Object.entries(this.updateFoodOptions ?? {})) {
        if (value && typeof value === 'object') {
          next[key] = Object.entries(value).reduce<Record<string, string>>((acc, [subsetKey, subsetValue]) => {
            if (typeof subsetValue === 'string')
              acc[subsetKey] = subsetValue;

            return acc;
          }, {});
        }
      }

      return next;
    },
    localizedUpdateFoodDefaultOption(): Record<string, boolean> {
      return Object.entries(this.updateFoodDefaultOption ?? {}).reduce<Record<string, boolean>>((acc, [lang, enabled]) => {
        if (typeof enabled === 'boolean')
          acc[lang] = enabled;

        return acc;
      }, {});
    },
    localizedUpdateFoodDefaultOptionValue(): Record<string, string> {
      return Object.entries(this.updateFoodDefaultOptionValue ?? {}).reduce<Record<string, string>>((acc, [lang, value]) => {
        if (typeof value === 'string')
          acc[lang] = value;

        return acc;
      }, {});
    },
    optionSubsetsByLanguage(): Record<string, { key: string; label: string; legacyKey: string }[]> {
      return this.optionSubsetsByLanguageCache;
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
    options: {
      immediate: true,
      deep: true,
      handler() {
        this.scheduleOptionsRefresh();
      },
    },
  },

  beforeUnmount() {
    if (this.optionsRefreshTimeout) {
      clearTimeout(this.optionsRefreshTimeout);
      this.optionsRefreshTimeout = null;
    }
  },

  methods: {
    rebuildOptionSubsets() {
      this.optionSubsetsByLanguageCache = Object.entries(this.options).reduce<Record<string, { key: string; label: string; legacyKey: string }[]>>((acc, [lang, options]) => {
        acc[lang] = this.optionSubsetsForLanguage(options);
        return acc;
      }, {});
    },
    syncUpdateFoodSettings() {
      if (!this.showUpdateFoodConfig)
        return;

      const subsetsByLanguage = this.optionSubsetsByLanguage;

      const next = Object.entries(subsetsByLanguage).reduce<Record<string, Record<string, string>>>((acc, [lang, subsets]) => {
        const mappings = this.localizedUpdateFoodOptions[lang] ?? {};
        const normalized = subsets.reduce<Record<string, string>>((subsetAcc, subset) => {
          const value = mappings[subset.key] ?? mappings[subset.legacyKey];
          if (typeof value === 'string')
            subsetAcc[subset.key] = value;

          return subsetAcc;
        }, {});

        acc[lang] = normalized;
        return acc;
      }, {});
      if (!this.isEqual(next, this.localizedUpdateFoodOptions))
        this.update('updateFoodOptions', next);

      const validLanguages = new Set(Object.keys(this.options));
      const nextDefaultOption = Object.entries(this.localizedUpdateFoodDefaultOption).reduce<Record<string, boolean>>((acc, [lang, enabled]) => {
        if (validLanguages.has(lang))
          acc[lang] = enabled;

        return acc;
      }, {});
      const nextDefaultOptionValue = Object.entries(this.localizedUpdateFoodDefaultOptionValue).reduce<Record<string, string>>((acc, [lang, value]) => {
        if (validLanguages.has(lang))
          acc[lang] = value;

        return acc;
      }, {});

      if (!this.isEqual(nextDefaultOption, this.localizedUpdateFoodDefaultOption))
        this.update('updateFoodDefaultOption', nextDefaultOption);

      if (!this.isEqual(nextDefaultOptionValue, this.localizedUpdateFoodDefaultOptionValue))
        this.update('updateFoodDefaultOptionValue', nextDefaultOptionValue);
    },
    scheduleOptionsRefresh() {
      if (this.optionsRefreshTimeout)
        clearTimeout(this.optionsRefreshTimeout);

      this.optionsRefreshTimeout = setTimeout(() => {
        this.rebuildOptionSubsets();
        this.syncUpdateFoodSettings();
        this.optionsRefreshTimeout = null;
      }, 500);
    },
    updateSubsetCode(lang: string, subset: { key: string; legacyKey: string }, value: string) {
      const current = { ...(this.localizedUpdateFoodOptions[lang] ?? {}) };
      delete current[subset.legacyKey];

      this.update('updateFoodOptions', {
        ...this.localizedUpdateFoodOptions,
        [lang]: {
          ...current,
          [subset.key]: value,
        },
      });
    },
    subsetCodeValue(lang: string, subset: { key: string; legacyKey: string }): string {
      return this.localizedUpdateFoodOptions[lang]?.[subset.key]
        ?? this.localizedUpdateFoodOptions[lang]?.[subset.legacyKey]
        ?? '';
    },
    updateDefaultOptionSwitch(lang: string, value: boolean | null) {
      const nextValue = !!value;
      if ((this.localizedUpdateFoodDefaultOption[lang] ?? false) === nextValue)
        return;

      this.update('updateFoodDefaultOption', {
        ...this.localizedUpdateFoodDefaultOption,
        [lang]: nextValue,
      });
    },
    updateDefaultOptionValue(lang: string, value: string) {
      if ((this.localizedUpdateFoodDefaultOptionValue[lang] ?? '') === value)
        return;

      this.update('updateFoodDefaultOptionValue', {
        ...this.localizedUpdateFoodDefaultOptionValue,
        [lang]: value,
      });
    },
    optionSubsetsForLanguage(options: ListOption[]): { key: string; label: string; legacyKey: string }[] {
      const subsets: { key: string; label: string; legacyKey: string }[] = [];

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

        subsets.push({
          key: JSON.stringify(selected.map(option => this.optionSubsetIdentity(option)).sort()),
          legacyKey: selectedIndexes.join('|'),
          label: selected.map(option => option.label).join(', '),
        });
      }

      return subsets;
    },
    optionSubsetIdentity(option: ListOption): string {
      if (option.id !== undefined)
        return `id:${option.id}`;

      return `value:${String(option.value)}`;
    },
    isEqual(left: unknown, right: unknown): boolean {
      return JSON.stringify(left) === JSON.stringify(right);
    },
  },
});
</script>

<style lang="scss" scoped></style>
