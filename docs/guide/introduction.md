# Introduction
Ready to dive🤿 in? Head on over to [Getting Started](/guide/getting-started.html).

---

a few years back I had an epiphany... The *way* we authenticate on the frontend - be it Firebase, Supabase, Laravel, Ruby, Auth0 or Okta - the **way** we authenticate is the exact same.

Take "logging in" as an example...
1. Send a request which returns a promise
2. Show a spinner
3. if successful, go to a different page
4. if failed show an error

*It's almost ALWAYS the same story*

So this raised an interresting question... Could the concept of frontend authentication be wrapped up into contracts? Turns out - at least among the currently supported backends - it **can**!

## The Creation Of VueAuth
VueAuth is not your average auth provider! It's a **set of contracts** that unify the *way* we authenticate on the frontend.

The image below demonstrates that Firebase (the auth provider) is decoupled from our app code through the use of interfaces.
![Authenticating With Firebase](/images/authenticating-vue-with-firebase.png)

This means we can swap out *firebase* with something like *supabase*
![Authenticating With Firebase](/images/authenticating-vue-with-supabase.png)

Cool huh? There's a few reasons we may want to do this (number **6** is the most exciting):
1. Auth providers are easily "swapped out" (e.g. maybe you're moving from firebase to Auth0 and don't want to rewrite the frontend auth code)
2. Beginners can easily play around with different auth providers
3. Devs don't have to relearn how to authenticate when trying a different auth provider
4. Devs don't have to think about the API for their authentication. It's all done for you via the contracts!
5. Tests for auth providers can - to an extent - be reused making it easier to add more auth providers
6. UI frameworks can easily support authentication with **any backend supported by VueAuth**! All they have to do is hook up their UI to VueAuth ([Quasar already has this](https://quasar.vueauth.com))

## How Does It Work?
If you're ready to dive in, checkout the [Getting Started Guide](/guide/getting-started). Otherwise, keep reading!

At its core, VueAuth is just a bunch of contracts! Here's a stripped down example of a VueAuth contract for loggin in with an identity (e.g. email) and password:

```ts
import { ValidationErrors } from '../types/ValidationErrors'
import { RequestErrors } from '../types/RequestErrors'
import { Ref, ComputedRef } from 'vue-demi'

export interface UseIdentityPasswordLoginReturn {
  form: Ref<IdentityPasswordLoginForm>;
  login: () => Promise<void>
  loading: Ref<boolean>
  validationErrors: Ref<ValidationErrors>;
  hasValidationErrors: ComputedRef<boolean>
  hasErrors: ComputedRef<boolean>
  errors: Ref<RequestErrors>;
  resetStandardErrors: () => void
  resetValidationErrors: () => void
  resetErrors: () => void
  isReauthenticating?: Ref<boolean>
  resetForm: () => void
}
```

Now, we implement this contract with something like "firebase". For example:

```ts
import useHandlesErrors from './useHandlesErrors'
import { getAuth, signInWithEmailAndPassword, AuthError, reauthenticateWithCredential, EmailAuthProvider } from 'firebase/auth'
import { ref, watch } from 'vue-demi'
import { UseIdentityPasswordLogin } from '@vueauth/core'

const useIdentityPasswordLogin: UseIdentityPasswordLogin = () => {
  const loading = ref(false)

  const isReauthenticating = ref(false)

  const {
    validationErrors,
    hasValidationErrors,
    hasErrors,
    errors,
    resetErrors,
    fromResponse: setErrorsFromResponse,
    resetStandardErrors,
    resetValidationErrors,
  } = useHandlesErrors()

  const form = ref({
    email: '',
    password: '',
  })
  function resetForm () {
    Object.keys(form.value).forEach(key => { form.value[key] = '' })
  }

  watch(form.value, () => {
    resetErrors()
  })

  const login = async () => {
    loading.value = true
    try {
      const auth = getAuth()
      const user = auth.currentUser

      if (user && isReauthenticating.value) {
        const { email, password } = form.value
        const credentials = EmailAuthProvider.credential(email, password)
        await reauthenticateWithCredential(auth.currentUser, credentials)
      } else {
        await signInWithEmailAndPassword(
          auth,
          form.value.email,
          form.value.password,
        )
      }
    } catch (err) {
      if (typeof err === 'object' && err !== null && err.constructor.name === 'FirebaseError') {
        setErrorsFromResponse(err as AuthError)
      }
    }
    loading.value = false
  }

  return {
    form,
    login,
    loading,
    validationErrors,
    hasValidationErrors,
    hasErrors,
    errors,
    resetErrors,
    resetStandardErrors,
    resetValidationErrors,
    isReauthenticating,
    resetForm,
  }
}

export {
  useIdentityPasswordLogin as default,
  useIdentityPasswordLogin,
}
```

Note that this isn't a lot of code, and it shouldn't be! These contracts usually aren't too difficult to implement, since frontend libraries (in the above example, *firebase/auth*) have usually done all the hard work for us.

Now that the contract is implemented, one might think it can simply be used like below:

```ts
import { useIdentityPasswordLogin } from '@vueauth/firebase'

const { login, loading, form } = useIdentityPasswordLogin()
```

This will work, but it introduces a problem.

::: warning uh-oh!
The import of `useIdentityPasswordLogin` is now coupled to `@vueauth/firebase`
:::

Hmmm, that's not good. It means we can't easily "swap out" our auth provider. Enter **Provide and inject**

## Decoupling Auth Providers With "Provide and Inject"
When installing VueAuth, we assign implementations to our contracts. It looks something like this:

Specifically, notice the `features` property
```ts
import { createApp } from 'vue'
import {
  SupabasePlugin,
  useIdentityPasswordLogin,
  useIdentityPasswordRegister,
  useHandlesErrors,
} from '@vueauth/supabase'
import { AuthPlugin } from '@vueauth/core'
import App from './App.vue'

const app = createApp(App)

const supabaseCredentials = {
  supabaseUrl: 'https://xxxxxxxxxxxx.supabase.co',
  supabaseKey: 'XXXXXXXXXX.XXXXXXXXX.XXXXXXXXXX',
}

app.use(SupabasePlugin, { credentials: supabaseCredentials })

app.use(AuthPlugin, {
  default: 'supabase',
  providers: {
    supabase: {
      features: { // Here, we bind "features" to implementations 🤿
        'identityPassword:register': useIdentityPasswordRegister,
        'identityPassword:login': useIdentityPasswordLogin,
        'errorHandler': useHandlesErrors,
      }
    }
  }
})

app.mount('#app')
```

Now behind the scenes, through the magic of provide and inject, we can import our composables like this:

```ts
import { useIdentityPasswordLogin } from '@vueauth/core'

const { login, loading, form } = useIdentityPasswordLogin()
```

Now we can easily swap out the `firebase` 'features' with, for example, `supabase` features and **everything would still work**!

And it turns out this has other benefits! It allows UI libraries to scaffold out all the authentication for you 😱. This has [already been done for Quasar](https://quasar.vueauth.com)

How. Cool. Is That!

Now if you're ready, head on over to the [Getting Started Guide](/guide/getting-started) and start authenticating with VueAuth!