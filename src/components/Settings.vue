<template>
  <v-card>
    <v-card-title class="d-flex flex-row align-center">
      Settings
      <v-spacer />
      <v-btn
        variant="text"
        density="compact"
        icon="mdi-close"
        @click="$emit('close')"
      />
    </v-card-title>
    <v-card-text>
      <v-switch label="Dark Theme" v-model="dark"></v-switch>

      <v-divider class="mt-2 mb-6"></v-divider>
      <dicom-web-settings />
    </v-card-text>
  </v-card>
</template>

<script lang="ts">
import { defineComponent, ref, watch } from 'vue';
import { useTheme } from 'vuetify';
import { useLocalStorage } from '@vueuse/core';

import DicomWebSettings from './dicom-web/DicomWebSettings.vue';
import { DarkTheme, LightTheme, ThemeStorageKey } from '../constants';

export default defineComponent({
  setup() {
    const theme = useTheme();
    const store = useLocalStorage(ThemeStorageKey, theme.global.name.value);
    const dark = ref(theme.global.name.value === DarkTheme);

    watch(dark, (isDark) => {
      theme.global.name.value = isDark ? DarkTheme : LightTheme;
      store.value = theme.global.name.value;
    });

    return {
      dark,
    };
  },
  components: {
    DicomWebSettings,
  },
});
</script>
