<script>
import { defineComponent } from 'vue'
import { IonPage } from '@ionic/vue'

import '@capacitor-community/http'
import { Plugins } from '@capacitor/core'
const { Http } = Plugins

export default defineComponent({
  name: 'Home',

  components: {
    IonPage,
  },

  data: () => ({
    advice: '',
    animationState: '',
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
    await Promise.all([
      this.getCookie('lastSaveDate').then(lastSaveDate => this.lastSaveDate = +lastSaveDate),
      this.getCookie('hourToFetchNewAdvice').then(hourToFetchNewAdvice => this.hourToFetchNewAdvice = +hourToFetchNewAdvice),
      this.getCookie('advice').then(advice => this.advice = advice)
    ])

    // ^Preferable implementation to below because all three requests can run in parallel
    // this.lastSaveDate = +await this.getCookie('lastSaveDate')
    // this.hourToFetchNewAdvice = +await this.getCookie('hourToFetchNewAdvice')
    // this.advice = await this.getCookie('advice')

    // If we haven't stored an hourToFetchNewAdvice before, calculate and store that and hourToEraseCurrentAdvice
    if(!this.hourToFetchNewAdvice) this.hourToFetchNewAdvice = this.currentHour

    // Store it to cookies regardless of status so it doesn't expire
    this.setCookie({
      key: 'hourToFetchNewAdvice',
      value: this.hourToFetchNewAdvice,
    })

    this.updateAdvice()
  },

  methods: {
    async deleteCookie(optionsObject) {
      return await Http.deleteCookie(optionsObject)
    },

    async getCookie(key) {
      const allCookiesWrapper = await Http.getCookies()
      const allCookies = allCookiesWrapper.value

      for (let i = 0; i < allCookies.length; ++i) {  // Cannot be forEach
        const currentCookie = allCookies[i]
        if(currentCookie.key === key) return currentCookie.value
      }

      return null
    },

    async setCookie(optionsObject) {
      const cookie = await Http.setCookie({
        ...optionsObject,
        ageDays: 2,  // Set max number of days to save cookie
      })
    },

    async fetchAdvice() {
      // Fetch a new advice slip when 24 hours have passed
      return await Http.request({
        method: 'GET',
        url: 'https://api.adviceslip.com/advice',
      })

      // Save it to the component's state
      .then(({ data }) => {
        this.animationState = 'fadeIn'
        this.resetAnimationState()

        // Save new advice
        this.advice = JSON.parse(data).slip.advice.toUpperCase()  // All caps to fit within available characters in font
          // In longform:
            // const dataInJs = JSON.parse(data)
            // const slip = dataInJs.slip
            // this.advice = slip.advice
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
        this.animationState = 'fadeOut'
        this.resetAnimationState()

        setTimeout(() => {
          this.advice = ''
          this.deleteCookie({ key: 'advice' })
        }, 10000)
      }

      // Check every 10m if it's time to refresh advice
      setTimeout(this.updateAdvice, 600000)
    },

    resetAnimationState() {
      setTimeout(() => {
        this.animationState = ''
      }, 10000)
    }
  }
})
</script>

<template>
<IonPage>
  <div class="Home">
    <!-- image from https://www.dmc.com/us/new-baby-name-pattern-9005116.html -->
    <img src="../../public/assets/embroidery-top.png" alt="leaf and heart flourish" />
    <p class="embroidery" :class="animationState">{{ advice }}</p>
    <img src="../../public/assets/embroidery-bottom.png" alt="leaf and heart flourish" />
  </div>
</IonPage>
</template>

<style scoped>
.Home {
  background: white;
  height: 100%;
  padding: 1rem;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: space-between;

  font-family: 'HovdenStitch';
  font-size: 5rem;
  color: #002657;
  text-align: center;
}

.embroidery {
  max-width: 686px;
  position: absolute;
  top: 53%;
  transform: translateY(-75%);
}

/* https://medium.com/cloud-native-the-gathering/how-to-use-css-to-fade-in-and-fade-out-html-text-and-pictures-f45c11364f08 */
.fadeIn {
  animation: fadeIn ease 10s;
}

.fadeOut {
  animation: fadeOut ease 10s;
}

@keyframes fadeIn {
  0% {
    opacity: 0;
  }
  100% {
    opacity: 1;
  }
}

@keyframes fadeOut {
  0% {
    opacity: 1;
  }
  100% {
    opacity: 0;
  }
}
</style>