<template>
<IonPage>
  <IonContent class="Home">
      {{ advice }}

      <IonButton @click="updateAdvice">get</IonButton>
  </IonContent>
</IonPage>
</template>

<script>
import { defineComponent } from 'vue'
import { IonButton, IonContent, IonPage } from '@ionic/vue';

import '@capacitor-community/http';
import { Plugins } from '@capacitor/core';
const { Http } = Plugins;

export default defineComponent({
  name: 'Home',

  components: {
    IonButton,
    IonContent,
    IonPage,
  },

  data: () => ({
    advice: '',
    hourToFetchNewAdvice: null,
    lastSaveDate: null,
    today: new Date(),
  }),

  computed: {
    currentDate() {
      return this.today.getDate()
    },

    currentHour() {
      return this.today.getHours()
    },

    hourToEraseCurrentAdvice() {
      let oneHourPrior = this.hourToFetchNewAdvice - 1
      if(oneHourPrior < 0) oneHourPrior = 23
      return oneHourPrior
    }
  },

  async ionViewWillEnter() {
    // Check if there are a stored date, hour, and advice
    this.lastSaveDate = await this.getCookie('lastSaveDate')
    this.hourToFetchNewAdvice = await this.getCookie('hourToFetchNewAdvice')
    this.advice = await this.getCookie('advice')

    console.log('ADVICE', this.advice)

    // If we haven't stored an hourToFetchNewAdvice before, calculate and store that and hourToEraseCurrentAdvice
    if(!this.hourToFetchNewAdvice) {
      console.log('setHourToFetchNewAdvice')
      this.hourToFetchNewAdvice = this.currentHour

      this.setCookie({
        key: 'hourToFetchNewAdvice',
        value: this.hourToFetchNewAdvice,
      })
    }

    // If there's no saved advice fetch and save advice
    if(!this.advice) this.updateAdvice()
  },

  methods: {
    async deleteCookie(optionsObject) {
      return await Http.deleteCookie(optionsObject);
    },

    async getCookie(key) {
      console.log({ key })
      const allCookiesWrapper = await Http.getCookies()
      const allCookies = allCookiesWrapper.value
      console.log('Got cookies', allCookies);

      for (let i = 0; i < allCookies.length; ++i) {
        const currentCookie = allCookies[i]

        if(currentCookie.key === key) {
          console.log('I\'m true', currentCookie.key, currentCookie.value)
          return currentCookie.value
        }
      }

      console.log('no match')
      return null
    },

    async setCookie(optionsObject) {
      console.log(optionsObject)
      const cookie = await Http.setCookie({
        ...optionsObject,
        ageDays: 2,  // Set max number of days to save cookie
      })
      console.log('set', cookie)
    },

    async fetchAdvice() {
      // Fetch a new advice slip when 24 hours have passed
      return await Http.request({
        method: 'GET',
        url: 'https://api.adviceslip.com/advice',
      }).

      // Save it to the component's state
      then(({ data }) => {
        // const dataInJs = JSON.parse(data)
        // const slip = dataInJs.slip
        // this.advice = slip.advice

        console.log({ data })

        // Save new advice
        this.advice = JSON.parse(data).slip.advice.toUpperCase()  // All caps to fit within available characters in font
        this.setCookie({
          key: 'advice',
          value: this.advice,
        })

        // Update lastSaveDate
        this.lastSaveDate = this.currentDate
        this.setCookie({
          key: 'lastSaveDate',
          value: this.currentDate,
        })
      })
    },

    updateAdvice() {
      if(this.currentHour === this.hourToFetchNewAdvice && this.currentDate != this.lastSaveDate) this.fetchAdvice()

      // Erase current advice when 23 hours have passed
      else if (this.currentHour === this.hourToEraseCurrentAdvice && this.advice) {
        this.advice = ''
        this.deleteCookie({ key: 'advice' })
      }

      // Check every 10m if it's time to refresh advice
      setTimeout(this.updateAdvice, 600000);
    },
  }
})
</script>

<style scoped>
.Home {
  background: white;
  font-family: 'HovdenStitch';
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