<template>
  <v-container :class="{ 'pa-0': $vuetify.display.mobile }">
    <v-row justify="center" :no-gutters="$vuetify.display.mobile">
      <v-col cols="12" lg="9">
        <v-card :tile="$vuetify.display.mobile">
          <!-- Survey info -->
          <v-card-title class="text-h5 font-weight-medium mb-2 pt-4" tag="h1">
            {{ $t('survey.welcome._') }}
          </v-card-title>
          <v-card-subtitle class="pb-4">
            {{ $t('survey.welcome.subtitle') }}
          </v-card-subtitle>
          <v-divider />
          <div class="px-4 py-2">
            <h2 class="text-h6 font-weight-medium mb-2 pt-4">
              {{ $t('survey.info') }}
            </h2>
            <div class="py-2">
              <h3 class="text-subtitle-2">
                {{ $t('survey._') }}
              </h3>
              <div class="text-body-2 text-medium-emphasis">
                {{ parameters?.name }}
              </div>
            </div>
            <div class="py-2">
              <h3 class="text-subtitle-2">
                {{ $t('survey.states._') }}
              </h3>
              <div class="text-body-2 text-medium-emphasis">
                {{ $t(`survey.states.${parameters?.state}`) }}
              </div>
            </div>
          </div>
          <v-divider />
          <!-- Recall info -->
          <template v-if="recallAllowed">
            <div class="px-4 py-2">
              <h2 class="text-h6 font-weight-medium mb-2 pt-4">
                {{ $t('recall.info') }}
              </h2>
              <template v-if="limitReached">
                <div class="d-flex align-center py-3">
                  <v-icon class="me-6" color="error" size="large">
                    fas fa-ban
                  </v-icon>
                  <div class="text-subtitle-2">
                    {{ $t(`recall.limitReached.${dailyLimitReached ? 'daily' : 'total'}`) }}
                  </div>
                </div>
              </template>
              <template v-else-if="hasFinished">
                <div class="d-flex align-center py-3">
                  <v-icon class="me-6" color="info" size="large">
                    $check
                  </v-icon>
                  <div class="flex-grow-1 text-subtitle-2">
                    {{ $t('recall.finishedAt', { finishedAt: endTime?.toLocaleString() }) }}
                  </div>
                  <div v-if="$vuetify.display.mdAndUp">
                    <v-btn
                      block
                      color="primary"
                      rounded
                      variant="outlined"
                      @click.stop="startRecall"
                    >
                      <v-icon start>
                        fas fa-beat fa-play
                      </v-icon>
                      {{ $t('recall.start.another') }}
                    </v-btn>
                  </div>
                </div>
                <div v-if="$vuetify.display.smAndDown" class="d-flex justify-end pb-3">
                  <v-btn
                    block
                    color="primary"
                    rounded
                    size="large"
                    variant="outlined"
                    @click.stop="startRecall"
                  >
                    <v-icon start>
                      fas fa-beat fa-play
                    </v-icon>
                    {{ $t('recall.start.another') }}
                  </v-btn>
                </div>
              </template>
              <template v-else-if="hasStarted">
                <div class="d-flex align-center py-3">
                  <v-icon class="me-6" color="info" size="large">
                    $pause
                  </v-icon>
                  <div class="flex-grow-1 text-subtitle-2">
                    {{ $t('recall.startedAt', { startedAt: startTime?.toLocaleString() }) }}
                  </div>
                  <div v-if="$vuetify.display.mdAndUp" class="d-flex align-stretch">
                    <v-btn
                      class="me-2"
                      color="info"
                      rounded
                      :to="{ name: 'survey-recall', params: { surveyId } }"
                      variant="outlined"
                    >
                      <v-icon class="fa-beat" start>
                        $pause
                      </v-icon>
                      {{ $t('recall.continue') }}
                    </v-btn>
                    <confirm-dialog
                      :label="$t('recall.abort.label')"
                      @confirm="cancelRecall"
                    >
                      <template #activator="{ props }">
                        <v-btn
                          v-bind="props"
                          color="error"
                          rounded
                          :title="$t('recall.abort.label')"
                          variant="outlined"
                        >
                          <v-icon start>
                            $cancel
                          </v-icon>
                          {{ $t('recall.abort._') }}
                        </v-btn>
                      </template>
                      {{ $t('recall.abort.confirm') }}
                    </confirm-dialog>
                  </div>
                </div>
                <div v-if="$vuetify.display.smAndDown" class="d-flex justify-sm-end pb-3">
                  <confirm-dialog
                    :label="$t('recall.abort.label')"
                    @confirm="cancelRecall"
                  >
                    <template #activator="{ props }">
                      <v-btn
                        v-bind="props"
                        class="flex-grow-1 flex-sm-grow-0 me-2"
                        color="error"
                        rounded
                        size="large"
                        :title="$t('recall.abort.label')"
                        variant="outlined"
                      >
                        <v-icon start>
                          $cancel
                        </v-icon>
                        {{ $t('recall.abort._') }}
                      </v-btn>
                    </template>
                    {{ $t('recall.abort.confirm') }}
                  </confirm-dialog>
                  <v-btn
                    class="flex-grow-1 flex-sm-grow-0"
                    color="info"
                    rounded
                    size="large"
                    :to="{ name: 'survey-recall', params: { surveyId } }"
                    variant="outlined"
                  >
                    <v-icon class="fa-beat" start>
                      $pause
                    </v-icon>
                    {{ $t('recall.continue') }}
                  </v-btn>
                </div>
              </template>
              <template v-else>
                <div class="d-flex align-center py-3">
                  <v-icon class="me-6" color="info" size="large">
                    $start
                  </v-icon>
                  <div class="flex-grow-1 text-subtitle-2">
                    {{ $t('recall.none') }}
                  </div>
                  <div v-if="$vuetify.display.mdAndUp">
                    <v-btn
                      color="primary"
                      rounded
                      :to="{ name: 'survey-recall', params: { surveyId } }"
                      variant="outlined"
                    >
                      <v-icon aria-hidden="true" class="fa-beat" start>
                        $start
                      </v-icon>
                      {{ $t('recall.start._') }}
                    </v-btn>
                  </div>
                </div>
                <div v-if="$vuetify.display.smAndDown" class="d-flex justify-end pb-3">
                  <v-btn
                    color="primary"
                    rounded
                    size="large"
                    :to="{ name: 'survey-recall', params: { surveyId } }"
                    variant="outlined"
                  >
                    <v-icon class="fa-beat" start>
                      $start
                    </v-icon>
                    {{ $t('recall.start._') }}
                  </v-btn>
                </div>
              </template>
            </div>
            <v-divider />
          </template>
          <!-- Feedback info -->
          <template v-if="feedbackAllowed">
            <div class="px-4 py-2">
              <h2 class="text-h6 font-weight-medium mb-2 pt-4">
                {{ $t('feedback.info') }}
              </h2>
              <template v-if="feedbackAvailable">
                <div class="d-flex align-center py-3">
                  <v-icon class="me-6" color="info" size="large">
                    $feedback
                  </v-icon>
                  <div class="flex-grow-1 text-subtitle-2">
                    {{ $t('feedback.status.available') }}
                  </div>
                  <div v-if="$vuetify.display.mdAndUp">
                    <v-btn
                      color="info"
                      rounded
                      :to="{ name: 'feedback-home', params: { surveyId } }"
                      variant="outlined"
                    >
                      <v-icon start>
                        $feedback
                      </v-icon>
                      {{ $t(`feedback._`) }}
                    </v-btn>
                  </div>
                </div>
                <div v-if="$vuetify.display.smAndDown" class="d-flex justify-end pb-3">
                  <v-btn
                    color="info"
                    rounded
                    size="large"
                    :to="{ name: 'feedback-home', params: { surveyId } }"
                    variant="outlined"
                  >
                    <v-icon start>
                      $feedback
                    </v-icon>
                    {{ $t(`feedback._`) }}
                  </v-btn>
                </div>
              </template>
              <div v-else class="d-flex align-center py-3">
                <v-icon class="me-6" color="info" size="large">
                  $info
                </v-icon>
                <div class="text-subtitle-2">
                  {{
                    $t('feedback.status.lowRecalls', {
                      minRecalls: parameters?.numberOfSubmissionsForFeedback,
                    })
                  }}
                </div>
              </div>
            </div>
            <v-divider />
          </template>
          <!-- Past recalls -->
          <v-card-subtitle class="my-4" tag="h2">
            {{ $t('recall.submissions.past') }}
          </v-card-subtitle>
          <v-card-text class="py-0">
            <template v-if="submissions.length">
              <v-timeline density="compact">
                <v-timeline-item
                  v-for="(submission, idx) in submissions"
                  :key="submission.id"
                  :dot-color="idx % 2 ? 'info' : 'primary'"
                  size="small"
                >
                  <v-row class="pt-1">
                    <v-col>
                      <strong>{{ `${$t('recall.submissions._')} ${idx + 1}` }}</strong>
                      <div class="text-caption">
                        {{ `${new Date(submission.endTime).toLocaleDateString()}` }}
                      </div>
                    </v-col>
                  </v-row>
                </v-timeline-item>
              </v-timeline>
            </template>
            <div v-else class="d-flex align-center px-4 py-3">
              <v-icon class="me-6" size="large">
                $survey
              </v-icon>
              <div class="text-subtitle-2">
                {{ $t('recall.submissions.none') }}
              </div>
            </div>
          </v-card-text>
        </v-card>
      </v-col>
    </v-row>
  </v-container>
</template>

<script lang="ts">
import type { SurveySubmissionAttributes } from '@intake24/common/types/http/admin';

import { mapState } from 'pinia';
import { computed, defineComponent, onMounted, ref } from 'vue';
import { useRouter } from 'vue-router';

import { userService } from '@intake24/survey/services';
import { useSurvey } from '@intake24/survey/stores';
import { sendGtmEvent } from '@intake24/survey/util';
import { ConfirmDialog } from '@intake24/ui';

export default defineComponent({
  name: 'SurveyHome',

  components: { ConfirmDialog },

  props: {
    surveyId: {
      type: String,
      required: true,
    },
  },

  setup(props) {
    const survey = useSurvey();
    const router = useRouter();

    const submissions = ref<SurveySubmissionAttributes[]>([]);
    const startTime = computed(() => survey.data.startTime);
    const endTime = computed(() => survey.data.endTime);

    async function startRecall() {
      await survey.startRecall(true);
      await router.push({ name: 'survey-recall', params: { surveyId: props.surveyId } });
    };

    async function cancelRecall() {
      await survey.cancelRecall();
    };

    onMounted(async () => {
      sendGtmEvent({
        event: 'surveyHome',
        action: 'login',
        scheme_prompts: 'preMeals',
      });
      submissions.value = await userService.submissions(props.surveyId);
    });

    return {
      submissions,
      startTime,
      endTime,
      cancelRecall,
      startRecall,
    };
  },

  computed: {
    ...mapState(useSurvey, [
      'feedbackAllowed',
      'recallAllowed',
      'feedbackAvailable',
      'hasStarted',
      'hasFinished',
      'dailyLimitReached',
      'totalLimitReached',
      'limitReached',
      'parameters',
    ]),
  },
});
</script>

<style lang="scss" scoped></style>
