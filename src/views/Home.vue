<template>
<div class="Home">
    <ion-label position="stacked">username</ion-label>
    <ion-input></ion-input>

    {{ advice }}

    <ion-button @click="getAdvice">get</ion-button>
</div>
</template>

<script>
// import { defineComponent } from 'vue'
import '@capacitor-community/http';
import { Plugins } from '@capacitor/core';

// Example of a GET request
const doGet = async () => {
  // Destructure as close to usage as possible for web plugin to work correctly
  // when running in the browser
  const { Http } = Plugins;

  return await Http.request({
    method: 'GET',
    url: 'https://api.adviceslip.com/advice',
  });
};

export default {
  name: 'Home',

  data: () => ({
    advice: '',
  }),

  methods: {
    getAdvice(query) {
      if(!query) {
        return doGet().then(({ data }) => {
          // const dataInJs = JSON.parse(data)
          // const slip = dataInJs.slip
          // this.advice = slip.advice

          this.advice = JSON.parse(data).slip.advice
        })
      }
    }
  }
}
</script>

<style scoped>
.Home {
  background: white;
}

/* #container {
  text-align: center;

  position: absolute;
  left: 0;
  right: 0;
  top: 50%;
  transform: translateY(-50%);
}

#container strong {
  font-size: 20px;
  line-height: 26px;
}

#container p {
  font-size: 16px;
  line-height: 22px;

  color: #8c8c8c;

  margin: 0;
}

#container a {
  text-decoration: none;
} */
</style>