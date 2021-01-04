<script lang="ts">
import { IonApp, IonRouterOutlet } from '@ionic/vue'
import { defineComponent } from 'vue'

export default defineComponent({
  name: 'App',

  components: {
    IonApp,
    IonRouterOutlet
  }
})
</script>

<template>
  <ion-app>
    <ion-router-outlet />
  </ion-app>
</template>

<style>
/* Makes cross-stitch font available across the app */
  /* Font source: from https://www.fontspace.com/hovden-stitch-font-f43700 */
@font-face {
  font-family: 'HovdenStitch';
  src: url('../public/assets/HovdenStitchRegular.otf');
}
</style>