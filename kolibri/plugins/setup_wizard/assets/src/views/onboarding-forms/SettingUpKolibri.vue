<template>

  <div class="full-page">
    <main class="content">
      <KolibriLoadingSnippet />
      <h1 class="page-title">
        {{ $tr('pageTitle') }}
      </h1>
      <p class="message">
        {{ $tr('pleaseWaitMessage') }}
      </p>
    </main>
    <div
      v-if="devMode"
      style="
        z-index: 9999;
        position: absolute;
        left: 1em;
        right: 1em;
        bottom: 1em;
        top: 1em;
        background-color: rgba(0, 0, 0, 0.86);
        padding: 2em;
        color: white;
        font-weight: bold;
        overflow: auto;
      "
    >
      <h2>Setup Wizard Debugger 3000</h2>
      <h3>Device Provisioning Data</h3>
      <pre>{{ JSON.stringify(deviceProvisioningData, null, 2) }}</pre>
      <KButtonGroup
        style="
          position: fixed;
          top: 2em;
          right: 3em;
        "
      >
        <KButton
          icon="back"
          text="START OVER"
          @click="wizardService.send('START_OVER')"
        />
        <KButton
          primary
          iconAfter="forward"
          text="Continue and Finish"
          @click="provisionDevice()"
        />
      </KButtonGroup>
    </div>
  </div>

</template>


<script>

  import { v4 } from 'uuid';
  import omitBy from 'lodash/omitBy';
  import get from 'lodash/get';
  import { currentLanguage } from 'kolibri.utils.i18n';
  import { checkCapability } from 'kolibri.utils.appCapabilities';
  import KolibriLoadingSnippet from 'kolibri.coreVue.components.KolibriLoadingSnippet';
  import urls from 'kolibri.urls';
  import client from 'kolibri.client';
  import Lockr from 'lockr';
  import { DeviceTypePresets, UsePresets } from '../../constants';

  export default {
    name: 'SettingUpKolibri',
    components: { KolibriLoadingSnippet },
    inject: ['wizardService'],
    data() {
      return {
        devMode: process.NODE_ENV !== 'production',
      };
    },
    computed: {
      facilityData() {
        const usersName = get(this.wizardContext('superuser'), 'full_name', '');
        const facilityName =
          this.wizardContext('facilityName') ||
          this.$tr('onMyOwnFacilityName', { name: usersName });
        const selectedFacility = this.wizardContext('selectedFacility');
        if (selectedFacility) {
          if (selectedFacility.id) {
            // Imported a facility already otherwise we wouldn't have an ID yet,
            // so we'll just be sending off the `facility_id`
            return { facility_id: selectedFacility.id };
          } else {
            // Otherwise we'll pass the facility data we have (including settings set by user)
            return { facility: selectedFacility };
          }
        } else {
          return { facility: { name: facilityName } };
        }
      },
      learnerCanLoginWithNoPassword() {
        // The user answers the question "Enable passwords?" -- so the `requirePassword` value
        // is the boolean opposite of whatever the value we need to assign to
        // `learner_can_login_with_no_password` in the API call.
        // If there is already a facility imported, we will use its value
        // If it is `null`, then it was never set by the user and we set to require passwords
        const { facility, facility_id } = this.facilityData;
        // If we have a facility_id then we imported the facility
        if (facility_id) {
          return null;
        }
        const facilitySetting = get(facility, 'learner_can_login_with_no_password', null);
        if (facilitySetting !== null) {
          return facilitySetting;
        } else {
          return this.wizardContext('requirePassword') === null
            ? true
            : !this.wizardContext('requirePassword');
        }
      },
      learnerCanEditPassword() {
        // Note that we don't ask this question of a user during onboarding -- however,
        // the nonformal facility will set this to `true` by default -- which does not jive
        // with the possibility that a user can login with no password
        if (
          // Learner cannot edit a password they cannot set
          this.learnerCanLoginWithNoPassword ||
          // OS on my own users don't use password to sign in
          (this.isOnMyOwnSetup && checkCapability('get_os_user'))
        ) {
          return false; // Learner cannot edit a password they cannot set
        } else {
          return null; // We'll set this to a key and null values are removed from the API call
        }
      },
      /** The data we will use to initialize the device during provisioning */
      deviceProvisioningData() {
        let superuser = null;
        // We only need a superuser if we cannot get the OS user; null valued keys will be omitted
        // in the eventual API call
        if (!checkCapability('get_os_user')) {
          // Here we see if we've set a firstImportedLodUser -- if they exist, they must be the
          // superuser as they were the first imported user.
          if (this.wizardContext('firstImportedLodUser')) {
            superuser = this.wizardContext('firstImportedLodUser');
          }
          if (!superuser) {
            // If we are creating a user, their data is in the Vuex store because UserCredentials is
            // tightly coupled to it (for now).
            superuser = this.wizardContext('superuser') || this.$store.state.onboardingData.user;
          }
        }

        const settings = {
          learner_can_login_with_no_password: this.learnerCanLoginWithNoPassword,
          learner_can_edit_password: this.learnerCanEditPassword,
          on_my_own_setup: this.isOnMyOwnSetup,
          learner_can_sign_up: this.wizardContext('learnerCanCreateAccount'),
        };

        const payload = {
          ...this.facilityData,
          superuser,
          settings: omitBy(settings, v => v === null),
          preset: this.wizardContext('formalOrNonformal') || 'nonformal',
          language_id: currentLanguage,
          device_name:
            this.wizardContext('deviceName') ||
            this.$tr('onMyOwnDeviceName', { name: get(superuser, 'full_name', '') }),
          allow_guest_access: Boolean(this.wizardContext('guestAccess')),
          is_provisioned: true,
          os_user: checkCapability('get_os_user'),
          is_soud: this.wizardService.state.context.fullOrLOD === DeviceTypePresets.LOD,
          auth_token: v4(),
        };

        // Remove anything that is `null` value
        return omitBy(payload, v => v === null);
      },

      /** Introspecting the machine via it's `state.context` properties */
      isOnMyOwnSetup() {
        return this.wizardContext('onMyOwnOrGroup') == UsePresets.ON_MY_OWN;
      },
    },
    mounted() {
      if (this.devMode) {
        return null; // debugger activated, don't do anything
      } else {
        this.provisionDevice();
      }
    },
    methods: {
      // A helper for readability
      wizardContext(key) {
        return this.wizardService.state.context[key];
      },
      provisionDevice() {
        client({
          url: urls['kolibri:core:deviceprovision'](),
          method: 'POST',
          data: this.deviceProvisioningData,
        })
          .then(response => {
            const appKey = response.data.app_key;

            const path = appKey
              ? urls['kolibri:kolibri.plugins.app:initialize'](appKey) + '?auth_token=' + v4()
              : urls['kolibri:kolibri.plugins.user_auth:user_auth']();

            Lockr.set('savedState', null); // Clear out saved state machine
            window.location.href = path;
          })
          .catch(e => console.error(e));
      },
    },
    $trs: {
      pageTitle: {
        message: 'Setting up Kolibri',
        context: 'The title of the page',
      },
      pleaseWaitMessage: {
        message: 'This may take several minutes',
        context: 'Kolibri is working in the background and the user may need to wait',
      },
      onMyOwnDeviceName: {
        message: 'Personal device for {name}',
        context:
          "The default device name for a user installing Kolibri using the personal 'on my own' (formerly Quick Start) flow",
      },
      onMyOwnFacilityName: {
        message: 'Home Facility for {name}',
        context:
          "Default facility name when Kolibri is installed with the 'Quick start' setup option for at home learning, outside any type of structure or institution like a school or a library. '{name}' will display the full name of the super admin user for their Kolibri server. Note that users can change this default name after the setup, and put whatever name they want to use for their home facility.",
      },
    },
  };

</script>


<style scoped lang="scss">

  .full-page {
    /* Fill the screen, no scroll bars */
    position: relative;
    width: 100vw;
    height: 100vh;
    overflow: hidden;
  }

  .content {
    /* Vertically centered */
    position: relative;
    top: 50%;
    transform: translateY(-50%);
  }

  .page-title,
  .message {
    padding: 0 1em;
    text-align: center;
  }

  .page-title {
    font-size: 1.5em;
  }

</style>
