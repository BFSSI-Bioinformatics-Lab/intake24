<template>
  <v-container :class="{ 'pa-0': $vuetify.display.mobile }">
    <v-row class="justify-center" :no-gutters="$vuetify.display.mobile">
      <v-col cols="12" md="8" sm="9">
        <v-card v-if="profile" :tile="$vuetify.display.mobile">
          <v-card-title class="text-h5 font-weight-medium mb-2 pt-4" tag="h1">
            {{ $t('profile._') }}
          </v-card-title>
          <v-divider />
          <div class="px-4 py-2">
            <h2 class="text-caption text-strong-emphasis font-weight-medium pt-2 pb-1">
              {{ $t('profile.info') }}
            </h2>
            <div class="d-flex align-center py-3">
              <v-avatar class="me-6" color="secondary" icon="fas fa-user" />
              <div>
                <div class="text-subtitle-2">
                  {{ $t('common.name') }}
                </div>
                <div class="text-body-2 text-medium-emphasis">
                  {{ user?.name || $t('common.not.provided') }}
                </div>
              </div>
            </div>
            <div class="d-flex align-center py-3">
              <v-avatar class="me-6" color="secondary" icon="fas fa-id-badge" />
              <div>
                <div class="text-subtitle-2">
                  {{ $t('profile.provider') }}
                </div>
                <div class="text-body-2 text-medium-emphasis">
                  {{ profile.subject.provider || $t('common.not.provided') }}
                </div>
              </div>
            </div>
            <div class="d-flex align-center py-3">
              <v-avatar class="me-6" color="secondary" icon="fas fa-key" />
              <div>
                <div class="text-subtitle-2">
                  {{ $t('profile.providerId') }}
                </div>
                <div class="text-body-2 text-medium-emphasis">
                  {{ profile.subject.providerKey || $t('common.not.provided') }}
                </div>
              </div>
            </div>
          </div>
          <v-divider />
          <div class="px-4 py-2">
            <h2 class="text-caption text-strong-emphasis font-weight-medium pt-2 pb-1">
              {{ $t('profile.settings') }}
            </h2>
            <div class="d-flex align-center py-3">
              <v-avatar class="me-6" color="secondary" icon="fas fa-language" />
              <v-select
                class="py-1"
                hide-details="auto"
                item-title="englishName"
                item-value="code"
                :items="languages"
                :label="$t('profile.languages._')"
                :model-value="language"
                variant="outlined"
                @update:model-value="updateLanguage"
              >
                <template #item="{ props, item }">
                  <v-list-item v-bind="props" :title="item.englishName">
                    <template #prepend>
                      <span :class="`fi fi-${item.countryFlagCode} me-3`" />
                    </template>
                  </v-list-item>
                </template>
                <template #selection="{ item }">
                  <span :class="`fi fi-${item.countryFlagCode} me-3`" />
                  {{ item.englishName }}
                </template>
              </v-select>
            </div>
          </div>
          <v-divider />
          <app-info />
          <v-card-text class="mt-6 d-flex justify-center">
            <confirm-dialog :label="$t('common.logout._')" @confirm="logout">
              <template #activator="{ props }">
                <v-btn
                  v-bind="props"
                  :block="$vuetify.display.mobile"
                  class="px-10"
                  color="primary"
                  rounded
                  size="x-large"
                  variant="outlined"
                >
                  <span class="me-2">{{ $t('common.logout._') }}</span>
                  <v-icon end>
                    $logout
                  </v-icon>
                </v-btn>
              </template>
              {{ $t('common.logout.text') }}
            </confirm-dialog>
          </v-card-text>
        </v-card>
      </v-col>
    </v-row>
  </v-container>
</template>

<script lang="ts">
import { mapState } from 'pinia';
import { computed, defineComponent } from 'vue';
import { useRouter } from 'vue-router';

import { useApp, useAuth, useSurvey, useUser } from '@intake24/survey/stores';
import { AppInfo, ConfirmDialog } from '@intake24/ui';

export default defineComponent({
  name: 'SurveyUserProfile',

  components: { AppInfo, ConfirmDialog },

  props: {
    surveyId: {
      type: String,
      required: true,
    },
  },

  setup(props) {
    const app = useApp();
    const router = useRouter();

    const language = computed(() => app.lang);
    const languages = computed(() => app.availableLanguages);

    const updateLanguage = async (languageId: string) => {
      useApp().setLanguage(languageId);
    };

    const logout = async () => {
      await useAuth().logout(true);
      const { surveyId } = props;
      await router.push(
        surveyId ? { name: 'survey-login', params: { surveyId } } : { name: 'home' },
      );
    };

    return {
      language,
      languages,
      logout,
      updateLanguage,
    };
  },

  computed: {
    ...mapState(useUser, ['profile']),
    ...mapState(useSurvey, ['user']),
  },
});
</script>
