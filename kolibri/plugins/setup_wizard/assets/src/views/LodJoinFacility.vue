<template>

  <UserCredentialsForm
    :header="$tr('header')"
    :description="getCommonSyncString('superAdminPermissionDescription')"
    :footerMessageType="footerMessageType"
    :disabled="loading"
    :step="1"
    :steps="2"
    :doNotContinue="true"
    :uniqueUsernameValidator="() => true"
    :adminUserLabels="false"
    :noBackAction="false"
    @submit="handleClickNext"
  />

</template>


<script>

  import commonCoreStrings from 'kolibri.coreVue.mixins.commonCoreStrings';
  import { TaskResource } from 'kolibri.resources';
  import { ERROR_CONSTANTS } from 'kolibri.coreVue.vuex.constants';
  import commonSyncElements from 'kolibri.coreVue.mixins.commonSyncElements';
  import { SetupWizardResource } from '../api';
  import { FooterMessageTypes } from '../constants';
  import UserCredentialsForm from './onboarding-forms/UserCredentialsForm';

  export default {
    name: 'LodJoinFacility',
    components: {
      UserCredentialsForm,
    },
    mixins: [commonCoreStrings, commonSyncElements],
    inject: ['wizardService'],
    data() {
      const footerMessageType = FooterMessageTypes.JOIN_FACILITY;
      return {
        loading: false,
        footerMessageType,
        shouldValidate: false,
        usernameIsUniqueFn: () => true,
      };
    },
    computed: {
      facility() {
        return this.wizardService._state.context.selectedFacility;
      },
    },
    methods: {
      handleClickNext() {
        this.shouldValidate = true;
        const { baseurl, id } = this.wizardService.state.context.importDevice;

        const user = {
          username: this.$store.state.onboardingData.user.username,
          password: this.$store.state.onboardingData.user.password,
        };

        this.loading = true;
        SetupWizardResource.createuseronremote({
          facility_id: this.facility.id,
          baseurl: baseurl.slice(0, -1),
          ...user,
        }).then(response => {
          const { status, data } = response.data;

          if (status == 201) {
            const task_name = 'kolibri.core.auth.tasks.peeruserimport';
            const params = {
              type: task_name,
              ...user,
              facility: this.facility.id,
              facility_name: this.facility.name,
              device_id: id,
              using_admin: false,
            };

            TaskResource.startTask(params)
              .then(() => this.wizardService.send('CONTINUE'))
              .catch(err => console.error(err));
          } else {
            const errorData = JSON.parse(data);
            if (errorData.find(error => error.id === ERROR_CONSTANTS.USERNAME_ALREADY_EXISTS)) {
              this.usernameIsUniqueFn = () => false;
            }
            this.loading = false;
          }
        });
      },
    },
    $trs: {
      header: {
        message: 'Select super admin',
        context: 'Page title,',
      },
    },
  };

</script>


<style lang="scss" scoped>

  .select {
    max-width: 400px;
    margin: 24px 0;
  }

</style>
