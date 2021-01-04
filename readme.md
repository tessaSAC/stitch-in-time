# Bypassing CORS with the Capacitor Community HTTP Plugin

[CORS](https://www.codecademy.com/articles/what-is-cors), or Cross-Origin Resource Sharing, is not a popular word among developers working with front-end tech. A browser-enforced restriction mainly to protect users from a type of attack known as [cross-site request forgery](https://owasp.org/www-community/attacks/csrf), CORS is more well known for the headaches it causes web developers, and that's before we even think to mention mobile applications! But what if there were a better way?

Thanks to the [Capacitor Community](https://github.com/capacitor-community/welcome), there is! The Capacitor Community is an open-source working group that builds and maintains useful tools and plugins for Ionic's [Capacitor runtime](https://github.com/ionic-team/capacitor), and its HTTP plugin empowers developers to sidestep CORS and smoothly make successful [HTTP requests](https://www.tutorialspoint.com/http/http_requests.htm) across desktop and mobile devices.

Let's get to know the plugin by using it to build an aesthetic daily advice app!


![Screenshot of cross-stitched advice](./public/assets/readme/app_preview.jpg?raw=true "Being kind is more rewarding than being right.")

## What we'll build

*A Vue.js app that loads and displays one cross-stitched piece of advice per day from the [Advice Slip JSON API](https://api.adviceslip.com) and which can be deployed cross platform.*

&nbsp;

## What we'll learn

- How to make a GET request with the HTTP plugin
- How to get, set, and delete cookies

&nbsp;

## Prior knowledge

In order to follow along, it might be helpful to have some understanding of the following technologies:

- [Ionic](https://ionicframework.com/docs/vue/overview)
- [Vue.js](https://v3.vuejs.org)
- [JavaScript](https://developer.mozilla.org/en-US/docs/Web/JavaScript)
- [TypeScript](https://www.typescriptlang.org/docs/handbook/typescript-in-5-minutes.html)
- [Async/Await](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function)
- [Git](https://www.udacity.com/course/version-control-with-git--ud123)
- [Yarn](https://yarnpkg.com/getting-started) or [NPM](https://flaviocopes.com/npm)
- [Bash](https://www.educative.io/blog/bash-shell-command-cheat-sheet)

&nbsp;

## Getting started

#### Note: If at any point you get lost, you can always refer back to the example code in the [main repo](https://github.com/tessaSAC/stitch-in-time).

You can either [clone the `startingPoint` branch](https://github.com/tessaSAC/stitch-in-time/tree/startingPoint) or start your own project from scratch with the following steps:

1. Create a new Ionic Vue project as per [Creating a project with the Ionic CLI](https://ionicframework.com/docs/vue/quickstart#creating-a-project-with-the-ionic-cli)
2. `cd` into your project and [install the HTTP plugin](https://github.com/capacitor-community/http) with the following commands:

```bash
npm install @capacitor-community/http
npx cap sync
```

> ### 😬 Tip
>
> If, after running both commands, you see the error `Capacitor could not find the web assets directory "pathToYourRepo/dist"`, try running `yarn build` or `npm build` before running `npx cap sync` a second time.

&nbsp;

3. Open your project in your favorite code editor; I like to use [VSCode](https://code.visualstudio.com).

4. _Optional:_ Download [HovdenStitch](https://www.fontspace.com/hovden-stitch-font-f43700), a free cross-stitch font, and move the `otf` file to your `public/assets` folder as `HovdenStitchRegular.otf`. Then in `src/App.vue`, copy and paste the below style tag at the bottom of the file to allow this custom font to be used across your app.

```vue
<style>
@font-face {
  font-family: "HovdenStitch";
  src: url("../public/assets/HovdenStitchRegular.otf");
}
</style>
```

> ### 💡 Tip:
> For the purposes of this tutorial, CSS styles will be contained within Vue's [Single File Components](https://v3.vuejs.org/guide/single-file-component.html) so that all changes are explicitly visible. However when it comes to your own projects, consider putting global styles in your [`src/theme` folder](https://ionicframework.com/docs/theming/themes) when necessary..

&nbsp;

## Designing the order of operations
The basic idea for this app's functionality is that it will fetch and display a maximum of one new piece of advice per 24 hours, regardless of whether the user refreshes the page or leaves the app, and erase the current advice approximately one hour before new advice is fetched.

While this concept may seem simple on its face, it requires quite a bit of state tracking in order to ensure the expected behavior. You might, like me, find it helpful to draw or write out a rough plan for how you expect things to go as I've done here:

![Screenshot of handwritten flowchart roughly diagramming above behavior](./public/assets/readme/planning_notes.jpg?raw=true "notes")

&nbsp;

## Preparing HTTP helper functions


In `src/views/Home.vue`, replace your `script` tag with the below starter code:
```vue
<script>
import { defineComponent } from 'vue'
import { IonPage } from '@ionic/vue'

export default defineComponent({
  name: 'Home',
  components: { IonPage },

  data: () => ({
    advice: '',  // Advice currently being displayed
    animationState: '',  // Dynamic class that determines whether to fade advice in or out
    hourToFetchNewAdvice: null,
    lastSaveDate: null,  // The last time we fetched new Advice
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

  // When we'll check if advice data needs to be changed
  async ionViewWillEnter() {
    // For more on`ionViewWillEnter`, see: https://ionicframework.com/docs/vue/lifecycle#guidance-for-each-lifecycle-method
  },

  methods: {},
})
</script>
```
#### _**Optional:** Move the `script` tag to be at the top of the file, above the `template` tag as per the [Vue Style Guide](https://vuejs.org/v2/style-guide/#Single-file-component-top-level-element-order-recommended). Once you get used to this pattern it can speed up your development process by reducing the scrolling between `script` and `template` and between `template` and `style`._

&nbsp;

In order to use the HTTP Plugin, first import it by adding the following code to the top of the `script` tag as per the [README](https://github.com/capacitor-community/http/blob/455dc0cb0add1b872dbc914077b9754df4d8c0f3/src/definitions.ts#L106):

```js
import '@capacitor-community/http'
import { Plugins } from '@capacitor/core'
const { Http } = Plugins
```

Now that we have access to the HTTP plugin and its helper methods inside this component, add the following functions to the `Home` component's `methods`:

```js
async fetchAdvice() {
  return await Http.request({
    method: 'GET',
    url: 'https://api.adviceslip.com/advice',
  })
  .then(({ data }) => {
    // Set dynamic class to fade in advice
    this.animationState = 'fadeIn'
    this.resetAnimationState()

    // Save new advice
    this.advice = JSON.parse(data).slip.advice.toUpperCase()
      // In other words:
        // const dataInJs = JSON.parse(data)
        // const slip = dataInJs.slip
        // this.advice = slip.advice
        // this.advice = this.advice.toUpperCase() — // font supports upper case only

    // Update lastSaveDate
    this.lastSaveDate = this.currentDate
  })
},

updateAdvice() {

  // If 24h have passed, fetch new advice
  if(this.currentHour === this.hourToFetchNewAdvice && this.currentDate != this.lastSaveDate) this.fetchAdvice()

  // If 23 hours have passed, start fading out current advice
  else if (this.currentHour === this.hourToEraseCurrentAdvice && this.advice) {

    // Set dynamic class to fade out text
    this.animationState = 'fadeOut'
    this.resetAnimationState()

    // Clear advice from state after the fade out animation ends
    setTimeout(() => {
      this.advice = ''
    }, 10000)
  }

  // Check every 10m if it's time to fetch/erase advice
  setTimeout(this.updateAdvice, 600000)
},

// Clear animation from advice after one playthrough
// A safer approach might be to listen for `transistionend`
resetAnimationState() {
  setTimeout(() => {
    this.animationState = ''
  }, 10000)
},
```
#### The `fetchAdvice` method is almost identical to the `GET` example in the HTTP Plugin's README, but its syntax has been reordered to fit within [Vue's `methods` style](https://v3.vuejs.org/guide/data-methods.html#methods). The `HTTP.request` method enables us to make [HTTP calls](https://developer.mozilla.org/en-US/docs/Web/HTTP/Methods) across different deploy targets without worrying about CORS issues. Here we're using it to request advice from the [Advice Slip JSON API's random advice GET endpoint](https://api.adviceslip.com/#endpoint-random).


&nbsp;

Next, we'll want to add the following code to `ionViewWillEnter` to fetch advice when the page loads:
```js
async ionViewWillEnter() {
  // If we haven't stored an hourToFetchNewAdvice before, calculate and store that and hourToEraseCurrentAdvice
  if(!this.hourToFetchNewAdvice) this.hourToFetchNewAdvice = this.currentHour

  this.updateAdvice()
},
```

&nbsp;

## Displaying our cross-stitched advice

Open `views/Home.vue` and replace the existing `template` with the below code:
```vue
<template>
<IonPage>
  <div class="Home">
    <img src="yourImageHere" alt="Don't forget to add alt text!" />
    <p class="embroidery" :class="animationState">{{ advice }}</p>
    <img src="yourImageHere" alt="Don't forget to add alt text!" />
  </div>
</IonPage>
</template>
```

&nbsp;

Let's revew what's going on in the template:
- This *view*, or page, is wrapped in the `IonPage` component to [enable us to leverage component lifecycle hooks](https://ionicframework.com/docs/vue/lifecycle).
- The `Home` class will be used to visually center and style advice in the viewport.
- **Action required:** The `img` tags are placeholders for you to add personalized decorative flourishes to surround your advice; feel free to also [copy the images from the base repo](https://github.com/tessaSAC/stitch-in-time/tree/main/public/assets) or remove them altogether.
- The `p` tag is where we'll render advice from the Advice Slip JSON API.
- The [v-bound](https://v3.vuejs.org/api/directives.html#v-bind) `animationState` class enables us to dynamically fade advice in and out as necessary.

&nbsp;

Next, replace the `style` tag with the following:
```vue
<style scoped>
/* Center and style the content */
.Home {
  background: white;
  height: 100%;
  padding: 1rem;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: space-between;

  /* Apply the custom font and style the text */
  font-family: 'HovdenStitch';
  font-size: 5rem;
  color: #002657;
  text-align: center;
}

/* Allow the flourishes to visually curve more closely around the text */
.embroidery {
  max-width: 686px;
  position: absolute;
  top: 53%;
  transform: translateY(-75%);
}

/* Animate new advice being added and old advice being removed */
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
```

Now if you run `yarn serve` or `npm serve`, you should be able to see some advice in the local preview in your browser!

&nbsp;

## Persisting state

While it's great that our advice is rendering and all, you may have noticed a small catch: we get a new piece of advice on every page load, even though the `updateAdvice` method is supposed to wait 24 hours before fetching new advice. This is because the state is stored only within the component, which means when the component disappears, so does the data.

To get around this, we'll store some of our state in [cookies](https://en.wikipedia.org/wiki/HTTP_cookie), which can outlive the component lifecycle.

### To get started, add the following helpers to the component's `methods`:

```js
async setCookie(optionsObject) {
  const cookie = await Http.setCookie({
    ...optionsObject,
    ageDays: 2,  // Sets max number of days to save cookie
  })
},
```
#### This method is very similar to the `setCookie` example in the HTTP Plugin's README, but its syntax has been reordered to fit within Vue's `methods` style. It has also been augmented to expire any cookies after two days.

#### One potential point of confusion here is that when looking at the [Http.setCookie source code](https://github.com/capacitor-community/http/blob/455dc0cb0add1b872dbc914077b9754df4d8c0f3/src/web.ts#L112), it may seem like this method is behaving essentially identically to the [browser's HTTP Set-Cookie approach](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Set-Cookie); however, if you try to pass in a value for `name` or `SameSite`, you will quickly discover this is *not* what's happening. The [method's interface](https://github.com/capacitor-community/http/blob/455dc0cb0add1b872dbc914077b9754df4d8c0f3/src/definitions.ts#L99) reveals that this method will only take four potential pieces of data: `url`, `key`, `value`, and `ageDays`, where `key` becomes the cookie's `name` and `ageDays` its `Expires` value.

&nbsp;

```js
async deleteCookie(optionsObject) {
  return await Http.deleteCookie(optionsObject)
},

```
#### This method is similar to the `deleteCookie` example in the HTTP Plugin's README, with the addition of an optional `optionsObject` argument which can be passed to the HTTP Plugin's `deleteCookie` method. If we check out [the interface for this method in the source code](https://github.com/capacitor-community/http/blob/455dc0cb0add1b872dbc914077b9754df4d8c0f3/src/definitions.ts#L112), we can see that it will accept either a `key` or a `url` to specify which cookie should be deleted.

&nbsp;

```js
async getCookie(key) {
  const allCookiesWrapper = await Http.getCookies()
  const allCookies = allCookiesWrapper.value

  for (let i = 0; i < allCookies.length; ++i) {
    const currentCookie = allCookies[i]
    if(currentCookie.key === key) return currentCookie.value
  }

  return null
},
```
#### This method is similar to the `getCookies` example in the HTTP Plugin's README. However, that method will return an Array of all cookies, so if we want to get a specific one we'll have to filter the list ourselves.

#### Note that a `forEach` will not work here as that type of loop cannot be short-circuited by a `return` statement.

&nbsp;

### Now that we have access to some cookie helper methods, let's use them.

First, add them to `ionViewWillEnter`:
```js
async ionViewWillEnter() {
  // Check if there are a stored lastSaveDate, hourToFetchNewAdvice, and advice in cookies, i.e. outside component/session state
  await Promise.all([
    this.getCookie('lastSaveDate').then(lastSaveDate => this.lastSaveDate = +lastSaveDate),
    this.getCookie('hourToFetchNewAdvice').then(hourToFetchNewAdvice => this.hourToFetchNewAdvice = +hourToFetchNewAdvice),
    this.getCookie('advice').then(advice => this.advice = advice)
  ])

  if(!this.hourToFetchNewAdvice) this.hourToFetchNewAdvice = this.currentHour

  // Store the hourToFetchNewAdvice in cookies regardless so it doesn't expire
  this.setCookie({
    key: 'hourToFetchNewAdvice',
    value: this.hourToFetchNewAdvice,
  })

  this.updateAdvice()
},
```

Next, update `fetchAdvice`:

```js
async fetchAdvice() {
  return await Http.request({
    method: 'GET',
    url: 'https://api.adviceslip.com/advice',
  })
  .then(({ data }) => {
    this.animationState = 'fadeIn'
    this.resetAnimationState()

    this.advice = JSON.parse(data).slip.advice.toUpperCase()

    // Save the advice to a cookie too
    this.setCookie({
      key: 'advice',
      value: this.advice,
    })

    this.lastSaveDate = this.currentDate

    // Update lastSaveDate in cookies also
    this.setCookie({
      key: 'lastSaveDate',
      value: this.currentDate,
    })
  })
},
```

And `updateAdvice`:

```js
updateAdvice() {
  if(this.currentHour === this.hourToFetchNewAdvice && this.currentDate != this.lastSaveDate) this.fetchAdvice()

  else if (this.currentHour === this.hourToEraseCurrentAdvice && this.advice) {
    this.animationState = 'fadeOut'
    this.resetAnimationState()

    setTimeout(() => {
      this.advice = ''

      // Erase the advice from cookies as well
      this.deleteCookie({ key: 'advice' })
    }, 10000)
  }

  setTimeout(this.updateAdvice, 600000)
},
```

&nbsp;

## Celebrate!

We did it! Now the app will fetch and erase advice once every 24 hours as designed. 🎉

&nbsp;

## Review

We covered a lot of APIs and concepts from the Capacitor Community HTTP Plugin today including:
- How to make an HTTP `GET` request
- How to save a cookie
  - Which properties of a cookie can be set with the plugin
- How to get a *specific* cookie
- How to delete a *specific* cookie
- How to use the source code to answer questions not covered by the README

&nbsp;

## Next steps

What's next in the exciting world of cross-platform Vue apps? If you're short on ideas, here are a few suggestions for next steps:

- [Offline first](https://ionicframework.com/blog/best-practices-for-building-offline-apps): Try persisting state using another method, such as service-worker caching
- PWA/mobile: Try deploying the app to your phone as a [Progressive Web App](https://ionicframework.com/docs/vue/your-first-app/6-deploying-mobile) or a native [Android/iOS one](https://ionicframework.com/docs/vue/your-first-app/6-deploying-mobile)
  - Note: If you opted to generate your own project for this tutorial instead of cloning from the repo, you may need to follow some [additional installation steps](https://github.com/capacitor-community/http#installation) to get the HTTP Plugin working on Android
- Styling: Cross stitch not your thing? What theme would you give this app?
- Loading state: How might you save the state of the app such that if the user exits while advice is fading in or out, the transition will resume at the same spot the next time they reopen the app?
- [Another API](https://dev.to/biplov/15-fun-apis-for-your-next-project-5053): There are lots of free APIs out there; how would you apply the HTTP Plugin to a new project?

&nbsp;

### Thanks for reading and happy coding! 🎉