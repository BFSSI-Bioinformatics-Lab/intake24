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
      </template>
    </language-selector>

    <v-row v-if="showUpdateFoodConfig" class="mt-3">
      <v-col cols="12">
        <v-alert type="info" variant="tonal">
          {{ $t('survey-schemes.prompts.checkbox-list-prompt.updateFoodOptions') }}
        </v-alert>
      </v-col>
      <v-col cols="12">
        <v-switch
          hide-details="auto"
          label="Default option"
          :model-value="updateFoodDefaultOption"
          @update:model-value="update('updateFoodDefaultOption', $event)"
        />
      </v-col>
      <v-col v-if="updateFoodDefaultOption" cols="12">
        <v-text-field
          hide-details="auto"
          label="Default food code"
          :model-value="updateFoodDefaultOptionValue"
          variant="outlined"
          @update:model-value="update('updateFoodDefaultOptionValue', $event)"
        />
      </v-col>
      <v-col
        v-for="subset in optionSubsets"
        :key="subset.key"
        cols="12"
      >
        <v-text-field
          hide-details="auto"
          :label="`${subset.label}`"
          :model-value="updateFoodOptions[subset.key] ?? ''"
          variant="outlined"
          @update:model-value="updateSubsetCode(subset.key, $event)"
        />
      </v-col>
    </v-row>
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
      type: Boolean as PropType<Prompts['checkbox-list-prompt']['updateFoodDefaultOption']>,
      required: true,
    },
    updateFoodDefaultOptionValue: {
      type: String as PropType<Prompts['checkbox-list-prompt']['updateFoodDefaultOptionValue']>,
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
    baseOptions(): ListOption[] {
      if (this.options.en?.length)
        return this.options.en;

      const firstLang = Object.keys(this.options)[0];
      return firstLang ? this.options[firstLang] : [];
    },
    optionSubsets(): { key: string; label: string }[] {
      const options = this.baseOptions;
      const subsets: { key: string; label: string }[] = [];

      for (let mask = 1; mask < (2 ** options.length); mask++) {
        const selectedIndexes = [];
        for (let i = 0; i < options.length; i++) {
          if ((mask & (1 << i)) !== 0) {
            selectedIndexes.push(i);
          }
        }
        const selected = selectedIndexes.map(index => options[index]);
        const hasExclusive = selected.some(option => option.exclusive);

        if (hasExclusive && selected.length > 1)
          continue;

        const key = selectedIndexes.join('|');
        const label = selected.map(option => option.label).join(', ');
        subsets.push({ key, label });
      }

      return subsets;
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
    optionSubsets() {
      const valid = new Set(this.optionSubsets.map(subset => subset.key));
      const next = Object.entries(this.updateFoodOptions).reduce<Record<string, string>>((acc, [key, value]) => {
        if (valid.has(key))
          acc[key] = value;

        return acc;
      }, {});

      const currentKeys = Object.keys(this.updateFoodOptions);
      const nextKeys = Object.keys(next);
      const hasSameKeys = currentKeys.length === nextKeys.length
        && currentKeys.every(key => valid.has(key));

      if (!hasSameKeys)
        this.update('updateFoodOptions', next);
    },
  },

  methods: {
    updateSubsetCode(key: string, value: string) {
      this.update('updateFoodOptions', {
        ...this.updateFoodOptions,
        [key]: value,
      });
    },
  },
});
</script>

<style lang="scss" scoped></style>
