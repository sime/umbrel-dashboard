<template>
  <div>
    <div class="d-flex flex-column align-items-center justify-content-center min-vh100 p-2">
      <img alt="Umbrel" src="@/assets/logo.svg" class="mb-2 logo" />
      <h1 class="text-center mb-2">{{ heading }}</h1>
      <p class="text-muted w-75 text-center">{{ text }}</p>

      <div class="form-container mt-3 d-flex flex-column form-container w-100 align-items-center">
        <b-form-input
          v-model="name"
          ref="name"
          placeholder="Your name"
          v-show="currentStep === 1"
          class="card-input w-100"
          autofocus
        ></b-form-input>

        <input-password
          v-model="password"
          ref="password"
          v-show="currentStep === 2"
          placeholder="Your password"
          inputClass="card-input w-100"
        />

        <input-password
          v-model="confirmPassword"
          ref="confirmPassword"
          placeholder="Re-enter your password"
          v-show="currentStep === 3"
          inputClass="card-input w-100"
        />

        <div v-show="currentStep === 5">
          <seed
            :words="seed"
            @complete="finishedSeed"
            @incomplete="incompleteRecoverySeed"
            @input="inputRecoverySeed"
            v-show="seed.length && !isRegistering"
            :recover="recover"
          ></seed>
          <b-spinner v-show="!seed.length || isRegistering"></b-spinner>
        </div>

        <input-copy v-if="currentStep === 6" class="w-100" size="sm" :value="onionAddress"></input-copy>

        <div v-show="currentStep === 7">
          <div class="text-center bg-white p-3 rounded">
            <small
              class="d-block text-muted text-small text-center mb-3"
            >By clicking next, I agree that:</small>
            <span class="d-block text-muted text-small mb-1">
              <b-icon icon="exclamation-circle-fill" variant="warning" class="mr-1"></b-icon>Umbrel is in beta and should not be considered secure
            </span>
            <span class="d-block text-muted text-small mb-1">
              <b-icon icon="exclamation-circle-fill" variant="warning" class="mr-1"></b-icon>I should not put more funds on my Umbrel than I'm prepared to lose
            </span>
          </div>
        </div>

        <div class="text-center" v-show="currentStep === 8">
          <p
            class="text-muted"
          >But you don't have to wait for the sync to complete... You can start using Umbrel right away!</p>
          <a
            href="#"
            v-b-tooltip.hover.bottom
            title="Umbrel uses neutrino while the sync is in progress, and automatically switches to Bitcoin Core once it's synced"
          >
            <small>
              <b-icon icon="exclamation-circle-fill" variant="primary" class="mr-1"></b-icon>How?
            </small>
          </a>
        </div>

        <!-- <p class="text-danger text-left align-self-start mt-1">
          <small>{{ errorMessage }}</small>
        </p>-->

        <b-button
          variant="success"
          size="lg"
          @click="nextStep"
          :disabled="!isStepValid || isRegistering || !isLndOperational"
          class="mt-3 mx-auto d-block px-4"
          :class="{ 'loading-fade-blink': !isLndOperational || (currentStep === 8 && !unlocked), 'invisible': currentStep === 5 && recover && !isStepValid }"
        >{{ !isLndOperational ? "Loading" : nextButtonText }}</b-button>
        <b-button
          variant="link"
          size="sm"
          class="mt-3 mx-auto d-block"
          v-if="currentStep === 4 || (currentStep === 5 && !recover)"
          @click="skipSeed"
          :disabled="isRegistering"
        >Note Down Later</b-button>
        <b-button
          variant="link"
          size="sm"
          @click="recoverFromSeed"
          v-if="currentStep === 4"
          class="mt-2 mx-auto d-block"
        >Recover</b-button>
        <b-button
          variant="link"
          size="sm"
          @click="prevStep"
          v-if="currentStep > 0 && currentStep !== 6 && currentStep !== 8"
          class="mt-2 mx-auto d-block text-dark"
        >Back</b-button>
      </div>
      <b-progress :value="progress" height="1rem" class="onboarding-progress"></b-progress>
    </div>
  </div>
</template>

<script>
import Vue from "vue";
import VueConfetti from "vue-confetti";
import { mapState } from "vuex";

import delay from "@/helpers/delay";

import InputPassword from "@/components/Utility/InputPassword";
import Seed from "@/components/Utility/Seed";
import InputCopy from "@/components/Utility/InputCopy";

Vue.use(VueConfetti);

export default {
  data() {
    return {
      name: "",
      password: "",
      confirmPassword: "",
      currentStep: 0,
      steps: [
        {
          heading: "welcome to umbrel",
          text: "Your journey to become bitcoin starts now."
        },
        {
          heading: "what is your name?",
          text:
            "Your name stays on your Umbrel and is never shared with a 3rd party."
        },
        {
          heading: "set your password",
          text: "You'll need this password to login to your Umbrel."
        },
        {
          heading: "confirm your password",
          text: "You'll need this password to login to your Umbrel."
        },
        {
          heading: "note down your secret words",
          text:
            "On the next screen you will be shown 24 words. It's recommended that you write them down on a piece of paper and store it in a safe place."
        },
        {
          heading: "note down your secret words",
          text:
            'Remember, there is no "forgot password" button. You will need these 24 words to recover your Umbrel.'
        },
        {
          heading: "access from anywhere",
          text:
            "Even when you're not on your home network, you can access your Umbrel using Tor Browser on the following URL"
        },
        {
          heading: "one last thing",
          text: "Don't be too #reckless."
        },
        {
          heading: "🎉 that's it!",
          text:
            "Congratulations! Your Umbrel is now set up and synchronizing the Bitcoin blockchain."
        }
      ],
      notedSeed: false,
      isRegistering: false,
      recover: false,
      recoverySeed: []
    };
  },
  computed: {
    ...mapState({
      isLndOperational: state => state.lightning.operational,
      registered: state => state.user.registered,
      seed: state => state.user.seed,
      unlocked: state => state.lightning.unlocked,
      onionAddress: state => state.system.onionAddress
    }),
    heading() {
      if (this.currentStep === 5 && this.recover) {
        return "recover your umbrel";
      }
      return this.steps[this.currentStep]["heading"];
    },
    text() {
      if (this.currentStep === 5 && this.recover) {
        return "Enter your 24 secret words in the exact order to recover your Umbrel.";
      }
      return this.steps[this.currentStep]["text"];
    },
    nextButtonText() {
      if (this.currentStep === 0) {
        return "Start";
      }
      if (this.currentStep === 8) {
        return "Go to dashboard";
      }
      return "Next";
    },
    isStepValid() {
      if (this.currentStep === 1) {
        // if (!/^[A-Za-z ]+$/.test(this.name)) {
        //   return false;
        // }
        // if (this.name.length < 3) {
        //   return false;
        // }
        return this.name.length;
      }

      if (this.currentStep === 2) {
        return this.password.length > 11;
      }

      if (this.currentStep === 3) {
        if (this.confirmPassword !== this.password) {
          return false;
        }
      }

      if (this.currentStep === 5) {
        return this.notedSeed;
      }

      if (this.currentStep === 8) {
        return this.unlocked;
      }

      return true;
    },
    progress() {
      return this.currentStep === 0
        ? 0
        : Math.round((this.currentStep * 100) / (this.steps.length - 1));
    }
  },
  methods: {
    skipSeed() {
      if (this.currentStep === 4) {
        this.currentStep = 5;
      }
      return this.nextStep();
    },
    recoverFromSeed() {
      this.recover = true;
      this.currentStep++;
    },
    inputRecoverySeed(seed) {
      this.recoverySeed = seed;
    },
    async nextStep() {
      if (this.currentStep === 4) {
        this.recover = false;
      }

      //Register user and initialize wallet at the end
      if (this.currentStep === 5) {
        this.isRegistering = true;
        const seed = this.recover ? this.recoverySeed : this.seed;
        try {
          await this.$store.dispatch("user/register", {
            name: this.name,
            password: this.password,
            seed
          });
        } catch (error) {
          this.isRegistering = false;
          window.eerr = error;
          if (error.response && error.response.data) {
            this.$bvToast.toast(`${error.response.data}`, {
              title: "Error",
              autoHideDelay: 3000,
              variant: "danger",
              solid: true,
              toaster: "b-toaster-top-center"
            });
          }
          console.error("Error registering user", error);
          return;
        }

        this.isRegistering = false;

        // fetch onion address for the next step
        this.$store.dispatch("system/getOnionAddress");
      }

      if (this.currentStep === 7) {
        //Wohoo! Time to celebrate!
        this.$confetti.start({
          particles: [
            {
              type: "rect"
            }
          ]
        });

        this.lndUnlockInterval = window.setInterval(async () => {
          await this.$store.dispatch("lightning/getStatus");
          if (this.unlocked) {
            return window.clearInterval(this.lndUnlockInterval);
          }
        }, 1000);

        // TODO: Find the root problem and fix it
        // Refresh page after 60s if LND still hasn't unlocked
        window.setTimeout(() => {
          if (!this.unlocked) {
            window.location.reload(true);
          }
        }, 60 * 1000);

        //Ok. 3s is more than enough to celebrate.
        window.setTimeout(() => {
          this.$confetti.stop();
        }, 3000);
      }

      if (this.currentStep === 8) {
        return this.$router.push("/dashboard");
      }

      return (this.currentStep = this.currentStep + 1);
    },
    prevStep() {
      if (this.currentStep === 5) {
        this.notedSeed = false;
      }

      this.currentStep = this.currentStep - 1;
    },
    finishedSeed() {
      this.notedSeed = true;
    },
    incompleteRecoverySeed() {
      this.notedSeed = false;
    }
  },
  async created() {
    //redirect to home if the user is already registered
    if (this.registered) {
      return this.$router.push("/");
    }

    // Wait for LND
    while (!this.isLndOperational) {
      await this.$store.dispatch("lightning/getStatus");
      await delay(1000);
    }

    //generate a new seed on load
    this.$store.dispatch("user/getSeed", { password: "", otpToken: "" });
  },
  beforeDestroy() {
    window.clearInterval(this.lndUnlockInterval);
  },
  components: {
    InputPassword,
    Seed,
    InputCopy
  }
};
</script>

<style lang="scss" scoped>
.logo {
  height: 20vh;
  max-height: 200px;
  width: auto;
}
.form-container {
  max-width: 500px;
}
.onboarding-progress {
  position: absolute;
  width: 100%;
  top: 0;
  left: 0;
  border-radius: 0;
  background: transparent;
}
.loading-fade-blink {
  animation: loadingFadeBlink 1s infinite linear;
}

@keyframes loadingFadeBlink {
  0%,
  100% {
    opacity: 0.6;
  }
  50% {
    opacity: 0.3;
  }
}
</style>
