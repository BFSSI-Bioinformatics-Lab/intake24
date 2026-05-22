<template>
  <v-row justify="center">
    <v-col cols="12" md="8" sm="10">
      <div class="d-flex flex-column ga-3">
        <div v-if="showAll" class="pa-2">
          <v-btn block @click.stop="setAll">
            {{ $t('prompts.linkedAmount.all') }}
          </v-btn>
        </div>

        <v-card v-if="whole" class="pa-4" variant="outlined">
          <p class="mb-3 font-weight-medium text-subtitle-1">
            {{ $t('prompts.quantity.whole') }}
          </p>
          <div class="d-flex flex-row align-center justify-space-between ga-2">
            <v-btn
              :aria-label="`${$t('prompts.quantity.less')} ${$t('prompts.quantity.whole')}`"
              color="secondary"
              :disabled="minDisabled"
              @click="update(-1)"
            >
              {{ $t('prompts.quantity.less') }}
            </v-btn>
            <span aria-atomic="true" aria-live="polite" class="font-weight-medium text-h4" role="status">
              {{ wholeLabel }}
            </span>
            <v-btn
              :aria-label="`${$t('prompts.quantity.more')} ${$t('prompts.quantity.whole')}`"
              color="secondary"
              :disabled="maxDisabled"
              @click="update(1)"
            >
              {{ $t('prompts.quantity.more') }}
            </v-btn>
          </div>
        </v-card>

        <v-card v-if="fraction" class="pa-4" variant="outlined">
          <p class="mb-3 font-weight-medium text-subtitle-1">
            {{ $t('prompts.quantity.fraction') }}
          </p>
          <div class="d-flex flex-row align-center justify-space-between ga-2">
            <v-btn
              :aria-label="`${$t('prompts.quantity.less')} ${$t('prompts.quantity.fraction')}`"
              color="secondary"
              :disabled="minDisabled"
              @click="decrementFraction"
            >
              {{ $t('prompts.quantity.less') }}
            </v-btn>
            <span aria-atomic="true" aria-live="polite" class="font-weight-medium text-h4" role="status">
              {{ fractionLabel }}
            </span>
            <v-btn
              :aria-label="`${$t('prompts.quantity.more')} ${$t('prompts.quantity.fraction')}`"
              color="secondary"
              :disabled="maxDisabled"
              @click="incrementFraction"
            >
              {{ $t('prompts.quantity.more') }}
            </v-btn>
          </div>
        </v-card>

        <p aria-atomic="true" aria-live="polite" class="text-body-2" role="status">
          {{ $t('prompts.quantity._') }}: {{ quantitySummary }}
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
import { computed, watch } from 'vue';

defineOptions({ name: 'QuantityCardAccessible' });

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
});

const emit = defineEmits(['update:modelValue', 'update:confirmed']);

const fractions = [1 / 4, 1 / 3, 1 / 2, 2 / 3, 3 / 4];
const reversedFractions = computed(() => fractions.toReversed());

const currentValue = defineModel('modelValue', { type: Number, default: 1 });

const currentWhole = computed(() => Math.floor(currentValue.value));
const currentFraction = computed(() => currentValue.value - currentWhole.value);

const maxDisabled = computed(() => currentValue.value === props.max);
const minDisabled = computed(() => currentValue.value === props.min);

const wholeLabel = computed(() => currentWhole.value.toString());
const fractionLabel = computed(() => {
  if (!currentFraction.value)
    return currentFraction.value;

  const fraction = currentFraction.value.toFixed(2);

  switch (fraction) {
    case '0.25':
      return '¼';
    case '0.33':
      return '⅓';
    case '0.50':
      return '½';
    case '0.67':
      return '⅔';
    case '0.75':
      return '¾';
    default:
      return fraction.toString();
  }
});

const quantitySummary = computed(() => Number.parseFloat(currentValue.value.toFixed(2)).toString());

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

function incrementFraction() {
  const nextFraction = fractions.find(fraction => fraction.toFixed(2) > currentFraction.value.toFixed(2)) ?? 1;
  currentValue.value = clamp(currentWhole.value + nextFraction);
};
function decrementFraction() {
  if (!currentFraction.value) {
    currentValue.value = clamp(currentWhole.value - 1 + (fractions.at(-1) ?? 0));
    return;
  }

  const prevFraction = reversedFractions.value.find(fraction => fraction.toFixed(2) < currentFraction.value.toFixed(2)) ?? 0;
  currentValue.value = clamp(currentWhole.value + prevFraction);
};

function updateConfirmed(value: boolean) {
  emit('update:confirmed', value);
};

watch(currentValue, () => {
  if (props.confirmed)
    updateConfirmed(false);
});
</script>

<style scoped></style>
