<template>
  <component
    :is="customPromptLayout"
    v-bind="{ food, meal, prompt, section, isValid }"
    @action="action"
  >
    <template #actions>
      <yes-no-choice v-model="state" />
    </template>
  </component>
</template>

<script lang="ts" setup>
import type { EncodedFood } from '@intake24/common/surveys';

import { computed } from 'vue';

import { YesNoChoice } from '@intake24/survey/components/elements';
import { usePromptUtils } from '@intake24/survey/composables';
import { foodsService } from '@intake24/survey/services';
import { useSurvey } from '@intake24/survey/stores';

import { BaseLayout, CardLayout, PanelLayout } from '../layouts';
import { createBasePromptProps } from '../prompt-props';

defineOptions({
  name: 'YesNoPrompt',
  components: { BaseLayout, CardLayout, PanelLayout },
});

const props = defineProps({
  ...createBasePromptProps<'yes-no-prompt'>(),
  modelValue: {
    type: Boolean,
    default: undefined,
  },
});

const emit = defineEmits(['action', 'update:modelValue']);

const { action, customPromptLayout } = usePromptUtils(props, { emit });
const survey = useSurvey();

const isValid = computed(() => props.modelValue !== undefined);
const state = computed({
  get() {
    return props.modelValue;
  },
  set(value) {
    emit('update:modelValue', value);

    if (typeof value === 'boolean')
      void customAction(value);
  },
});

async function customAction(value: boolean) {
  if (props.prompt.updateFood) {
    const foodId = props.food?.id;
    const foodCode = value ? props.prompt.updateFoodYes?.trim() : props.prompt.updateFoodNo?.trim();

    if (foodId && foodCode && foodCode !== 'NO_UPDATE') {
      try {
        const foodData = await foodsService.getData(survey.localeId, foodCode);
        const newFood: EncodedFood = {
          id: foodId,
          type: 'encoded-food',
          data: foodData,
          searchTerm: props.food?.searchTerm ?? '',
          portionSizeMethodIndex: null,
          portionSize: null,
          customPromptAnswers: props.food?.customPromptAnswers ?? {},
          flags: props.food?.flags ?? [],
          linkedFoods: [],
        };

        survey.replaceFood({ foodId, food: newFood });
      }
      catch (error) {
        console.error('YesNoPrompt failed to replace food:', error);
      }
    }
  }

  action('next');
}

defineExpose({ isValid });
</script>

<style lang="scss" scoped></style>
