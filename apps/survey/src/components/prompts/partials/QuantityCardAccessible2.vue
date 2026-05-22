<template>
  <v-row justify="center">
    <v-col cols="12" md="8" sm="10">
      <div class="d-flex flex-column ga-3">
        <div v-if="showAll" class="pa-2">
          <v-btn block @click.stop="setAll">
            {{ $t('prompts.linkedAmount.all') }}
          </v-btn>
        </div>

        <v-card class="pa-4" variant="outlined">
          <div v-if="whole" class="d-flex flex-column ga-2">
            <label
              :id="wholeInputLabelId"
              class="mb-0 text-body-1"
              :for="wholeInputId"
            >
              {{ $t('prompts.quantity.accessible2.whole', { food: foodText }) }}
            </label>
            <v-number-input
              :id="wholeInputId"
              :aria-labelledby="wholeInputLabelId"
              :max="maxWhole"
              :min="minWhole"
              :model-value="currentWhole"
              variant="outlined"
              @update:model-value="updateWhole"
            />
          </div>

          <div v-if="fraction" class="d-flex flex-column ga-2" :class="{ 'mt-4': whole }">
            <label
              :id="fractionInputLabelId"
              class="mb-0 text-body-1"
              :for="fractionInputId"
            >
              {{ $t('prompts.quantity.accessible2.fraction', { food: foodText }) }}
            </label>
            <v-select
              :id="fractionInputId"
              :aria-labelledby="fractionInputLabelId"
              density="comfortable"
              hide-details="auto"
              :items="fractionOptions"
              :model-value="fractionInput"
              variant="outlined"
              @update:model-value="updateFraction"
            />
          </div>
        </v-card>

        <p aria-atomic="true" aria-live="polite" class="text-body-2" role="status">
          {{ $t('prompts.quantity.accessible2.currentSelection', { selection: quantitySummary }) }}
        </p>

        <div v-if="confirm" class="pt-1">
          <v-btn block color="primary" @click="updateConfirmed(true)">
            {{ $t('prompts.quantity.confirm') }}
          </v-btn>
        </div>
      </div>
    </v-col>
  </v-row>
</template>

<script lang="ts" setup>
import { computed, useId, watch } from 'vue';

import { useI18n } from '@intake24/ui';

defineOptions({ name: 'QuantityCardAccessible2' });

const props = defineProps({
  confirm: {
    type: Boolean,
    default: true,
  },
  min: {
    type: Number,
    default: 0.25,
  },
  max: {
    type: Number,
    default: 30,
  },
  showAll: {
    type: Boolean,
    default: false,
  },
  modelValue: {
    type: Number,
    default: 1,
  },
  confirmed: {
    type: Boolean,
    default: true,
  },
  fraction: {
    type: Boolean,
    default: true,
  },
  whole: {
    type: Boolean,
    default: true,
  },
  foodLabel: {
    type: String,
    default: '',
  },
});

const emit = defineEmits(['update:modelValue', 'update:confirmed']);

const { i18n: { t } } = useI18n();

const fractions = [1 / 4, 1 / 3, 1 / 2, 2 / 3, 3 / 4];
const currentValue = defineModel('modelValue', { type: Number, default: 1 });

const wholeInputId = useId();
const wholeInputLabelId = `${wholeInputId}-label`;
const fractionInputId = useId();
const fractionInputLabelId = `${fractionInputId}-label`;

const currentWhole = computed(() => Math.floor(currentValue.value));
const currentFraction = computed(() => currentValue.value - currentWhole.value);

const foodText = computed(() => props.foodLabel.trim() || t('prompts.quantity.accessible2.food'));

const minWhole = computed(() => Math.floor(props.min));
const maxWhole = computed(() => Math.floor(props.max));

const fractionOptions = computed(() => [
  { title: t('prompts.quantity.accessible2.none'), value: 0 },
  { title: '¼', value: 1 / 4 },
  { title: '⅓', value: 1 / 3 },
  { title: '½', value: 1 / 2 },
  { title: '⅔', value: 2 / 3 },
  { title: '¾', value: 3 / 4 },
] as const);

const fractionLabels = {
  0.25: '¼',
  0.33: '⅓',
  '0.50': '½',
  0.67: '⅔',
  0.75: '¾',
} as const;

const fractionInput = computed(() => normalizeFraction(currentFraction.value));
const quantitySummary = computed(() => {
  const wholeValue = currentWhole.value;
  const fractionValue = fractionInput.value;

  if (!fractionValue)
    return wholeValue.toString();

  const fractionLabel = fractionToLabel(fractionValue);

  if (!wholeValue)
    return fractionLabel;

  return `${wholeValue} ${t('prompts.quantity.and')} ${fractionLabel}`;
});

function fractionToLabel(value: number) {
  return fractionLabels[value.toFixed(2) as keyof typeof fractionLabels] ?? value.toFixed(2);
}

function normalizeFraction(value: number) {
  const normalized = value.toFixed(2);
  return fractions.find(fraction => fraction.toFixed(2) === normalized) ?? 0;
}

function setAll() {
  update(props.max);
  updateConfirmed(true);
};

function clamp(value: number) {
  return Math.min(props.max, Math.max(props.min, value));
};

function update(value: number) {
  currentValue.value = clamp(currentValue.value + value);
};

function updateWhole(value: string | number | null) {
  const sanitized = `${value ?? ''}`.replaceAll(/\D/g, '');
  const parsedWhole = sanitized ? Number.parseInt(sanitized, 10) : 0;

  currentValue.value = clamp(parsedWhole + fractionInput.value);
}

function updateFraction(value: number | string | null) {
  const parsed = Number(value ?? 0);
  const selectedFraction = normalizeFraction(Number.isNaN(parsed) ? 0 : parsed);

  currentValue.value = clamp(currentWhole.value + selectedFraction);
}

function updateConfirmed(value: boolean) {
  emit('update:confirmed', value);
};

watch(currentValue, () => {
  if (props.confirmed)
    updateConfirmed(false);
});
</script>

<style scoped></style>
