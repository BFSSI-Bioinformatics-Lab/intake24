<template>
  <v-app :class="{ mobile: $vuetify.display.mobile }">
    <loader :show="isAppLoading" />
    <v-navigation-drawer
      v-model="navDrawer"
      height="auto"
      :location="$vuetify.display.mobile ? 'bottom' : undefined"
      temporary
    >
      <template v-if="loggedIn && surveyId">
        <div class="py-3 d-flex flex-row justify-space-between align-center">
          <v-btn
            :to="{ name: 'survey-profile', params: { surveyId } }"
            variant="plain"
          >
            <v-icon class="me-4" color="info" icon="fas fa-circle-user" size="30" />
            {{ userName ?? $t('profile._') }}
          </v-btn>
          <confirm-dialog
            v-if="$vuetify.display.mobile"
            :label="$t('common.logout._')"
            @confirm="logout"
          >
            <template #activator="{ props }">
              <v-btn v-bind="props" class="me-2" color="grey-darken-2" rounded variant="tonal">
                <v-icon icon="$logout" start />
                <span>{{ $t('common.logout._') }}</span>
              </v-btn>
            </template>
            {{ $t('common.logout.text') }}
          </confirm-dialog>
        </div>
        <v-divider />
      </template>
      <v-list density="compact" nav>
        <v-list-item
          link
          :to="
            loggedIn && surveyId
              ? { name: 'survey-home', params: { surveyId } }
              : { name: 'home' }
          "
        >
          <template #prepend>
            <v-icon icon="$home" />
          </template>
          <v-list-item-title>{{ $t('common.home') }}</v-list-item-title>
        </v-list-item>
        <template v-if="loggedIn && surveyId">
          <v-list-item
            v-if="recallAllowed"
            link
            :to="{ name: 'survey-recall', params: { surveyId } }"
          >
            <template #prepend>
              <v-icon icon="$survey" />
            </template>
            <v-list-item-title>{{ $t('recall._') }}</v-list-item-title>
          </v-list-item>
          <v-list-item
            v-if="feedbackAllowed"
            link
            :to="{ name: 'feedback-home', params: { surveyId } }"
          >
            <template #prepend>
              <v-icon icon="$feedback" />
            </template>
            <v-list-item-title>{{ $t('feedback._') }}</v-list-item-title>
          </v-list-item>
        </template>
      </v-list>
      <template #append>
        <div v-if="loggedIn && !$vuetify.display.mobile" class="pa-2">
          <confirm-dialog
            :label="$t('common.logout._')"
            @confirm="logout"
          >
            <template #activator="{ props }">
              <v-btn v-bind="props" block color="grey-darken-2" rounded variant="tonal">
                <v-icon icon="$logout" start />
                {{ $t('common.logout._') }}
              </v-btn>
            </template>
            {{ $t('common.logout.text') }}
          </confirm-dialog>
        </div>
        <app-nav-footer />
      </template>
    </v-navigation-drawer>
    <gcds-header skip-to-href="#main-content" />
    <v-app-bar class="px-2 px-md-0" color="primary" flat>
      <v-app-bar-nav-icon
        v-if="!$vuetify.display.mobile"
        :title="$t('common.nav._')"
        @click.stop="toggleNavDrawer"
      />
      <template v-if="loggedIn">
        <div v-if="surveyName" class="app-bar-survey-info">
          <i18n-t keypath="recall.survey" tag="span">
            <template #name>
              <span class="font-weight-medium">{{ surveyName }}</span>
            </template>
          </i18n-t>
          <template v-if="recallAllowed">
            <v-divider v-if="$vuetify.display.smAndUp" class="bg-grey mx-4" vertical />
            <i18n-t keypath="recall.submissions.count" tag="span">
              <template #count>
                <span class="font-weight-medium">{{ recallNumber }}</span>
              </template>
            </i18n-t>
          </template>
        </div>
        <template v-if="!$vuetify.display.mobile">
          <v-spacer />
          <router-link v-if="surveyId" :to="{ name: 'survey-profile', params: { surveyId } }">
            <gcds-button
              button-role="primary"
              size="small"
              :title="$t('profile._')"
            >
              <span>{{ $t('profile._') }}</span>
              <v-icon end icon="$profile" />
            </gcds-button>
          </router-link>
          <confirm-dialog
            :label="$t('common.logout._')"
            @confirm="logout"
          >
            <template #activator="{ props }">
              <gcds-button v-bind="props" button-role="danger" size="small">
                {{ $t('common.logout._') }}
                <v-icon end icon="$logout" />
              </gcds-button>
            </template>
            {{ $t('common.logout.text') }}
          </confirm-dialog>
        </template>
      </template>
      <template v-else>
        <v-app-bar-title>{{ $t('common._') }}</v-app-bar-title>
      </template>
    </v-app-bar>
    <v-main id="main-content">
      <router-view />
    </v-main>
    <navigation
      v-if="showNav && surveyId"
      v-model="navDrawer"
      v-bind="{
        surveyId,
        recall: recallAllowed,
        feedback: feedbackAllowed,
      }"
      @toggle-nav="toggleNavDrawer"
    />
    <service-worker />
    <message-box />
    <gcds-footer
      contextual-heading="Intake24"
      contextual-links="{ &quot;Optional Link #1&quot;: &quot;#&quot;, &quot;Optional Link #2&quot;: &quot;#&quot;, &quot;Optional Link #3&quot;: &quot;#&quot; }"
      display="compact"
    />
  </v-app>
</template>

<script lang="ts" setup>
import { storeToRefs } from 'pinia';
import { computed, ref, watch } from 'vue';
import { useRoute, useRouter } from 'vue-router';
import { useLocale } from 'vuetify';

import { Navigation } from '@intake24/survey/components/layouts';
import { sendGtmEvent } from '@intake24/survey/util';
import { AppNavFooter, ConfirmDialog, Loader, MessageBox, ServiceWorker, useHttp, useI18n, useLanguage } from '@intake24/ui';

import { useAuth, useSurvey } from './stores';

const http = useHttp();
const route = useRoute();
const router = useRouter();
const vI18n = useLocale();
const { i18n: { t } } = useI18n();
useLanguage('survey', http, vI18n);
const auth = useAuth();
const survey = useSurvey();

const { loggedIn } = storeToRefs(auth);
const { feedbackAllowed, recallAllowed, recallNumber } = storeToRefs(survey);

const surveyName = computed(() => survey.parameters?.name);
const userName = computed(() => survey.user?.name);

const surveyId = computed<string | undefined>(() => route.params.surveyId?.toString());
const showNav = computed(() => loggedIn.value && !!surveyId.value);
const navDrawer = ref(false);
const title = computed(() => t(route.meta?.title ?? 'common._'));

function toggleNavDrawer() {
  navDrawer.value = !navDrawer.value;
};

async function logout() {
  await useAuth().logout(true);
  sendGtmEvent({
    event: 'surveyLogout',
  });
  await router.push(
    surveyId.value ? { name: 'survey-login', params: { surveyId: surveyId.value } } : { name: 'home' },
  );
};

watch(title, () => {
  document.title = title.value;
});
</script>

<style lang="scss">
@use '@intake24/survey/scss/app.scss';

.v-navigation-drawer.v-navigation-drawer--active {
  height: auto !important;
}

.v-app-bar {
  position: relative !important;
}

.app-bar-survey-info {
  background-color: #ffffff;
  border-radius: 4px;
  box-shadow:
    inset 0px 3px 1px -2px rgba(0, 0, 0, 0.2),
    0px 2px 2px 0px rgba(0, 0, 0, 0.14),
    0px 1px 5px 0px rgba(0, 0, 0, 0.12);
  color: rgba(0, 0, 0, 0.87);

  display: flex;
  flex-direction: row;
  align-items: center;

  padding: 8px 16px 8px 16px;

  overflow: hidden;

  span {
    font-size: 0.9rem;
    letter-spacing: 0.167em;
    text-transform: uppercase;

    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
  }

  @media screen and (max-width: 600px) {
    flex-direction: column;
    flex-grow: 1;
    align-items: stretch;
    justify-content: space-around;

    padding: 0px 16px 0px 16px;
  }
}
</style>
