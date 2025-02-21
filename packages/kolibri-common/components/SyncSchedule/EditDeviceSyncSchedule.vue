<template>

  <ImmersivePage
    :appBarTitle="$tr('editSyncScheduleTitle')"
    :route="backRoute"
    :icon="icon"
  >
    <KPageContainer v-if="device" :style="pageHeight">
      <KGrid gutter="48" class="edit-sync-schedule">

        <KGridItem>
          <h1>{{ $tr('editSyncScheduleTitle') }}</h1>
        </KGridItem>

        <KGridItem>
          <p>{{ device.device_name }} </p>
        </KGridItem>

        <KGrid class="align-kselects">
          <KGrid>
            <KGridItem>
              <KSelect
                v-model="selectedItem"
                class="selector"
                :style="selectorStyle"
                :options="selectArray"
                label="Repeat"
              />
            </KGridItem>

          </KGrid>
          <KGrid
            v-if="selectedItem.value !== 0
              && selectedItem.value !== 1"
            class=""
          >
            <KGridItem>
              <KSelect
                v-model="selectedDay"
                class="selector"
                :style="selectorStyle"
                :options="getDays"
                label="On"
              />

            </KGridItem>

          </KGrid>
          <KGrid
            v-if="selectedItem.value !== 0"
            class=""
          >
            <KGridItem>
              <KSelect
                v-model="selectedTime"
                class="selector"
                :style="selectorStyle"
                :options="SyncTime"
                label="At"
              />
            </KGridItem>
          </KGrid>

        </KGrid>
        <KGridItem>

          <p class="spacing">
            {{ $tr('serverTime') }}
            {{ now }}
          </p>

          <p class="spacing">
            <KCheckbox>
              {{ $tr('checkboxLabel') }}
            </KCheckbox>
          </p>
          <p>
            <KButton
              v-if="removeBtn"
              appearance="basic-link"
              class="spacing"
              @click="removeDeviceModal = true"
            >
              {{ $tr('removeDeviceLabel') }}
            </KButton>

          </p>

        </KGridItem>

      </KGrid>

    </KPageContainer>

    <BottomAppBar>
      <KButtonGroup>
        <KButton
          :text="coreString('cancelAction')"
          appearance="flat-button"
          @click="cancelBtn"
        />
        <KButton
          :text="coreString('saveAction')"
          :primary="true"
          @click="handleSaveSchedule"
        />
      </KButtonGroup>
    </BottomAppBar>


    <KModal
      v-if="removeDeviceModal"
      :title="$tr('removeDevice')"
      size="medium"
      :submitText="coreString('removeAction')"
      :cancelText="coreString('cancelAction')"
      @cancel="closeModal"
      @submit="handleDeleteDevice"
    >
      <KGrid>
        <KGridItem
          :layout8="{ span: 6 }"
          :layout12="{ span: 10 }"
        >
          <p>{{ deviceName }}</p>
        </KGridItem>

        <KGridItem
          :layout8="{ span: 8 }"
          :layout12="{ span: 12 }"
        >
          <p>{{ $tr('removeDeviceWarning') }}</p>
          <p v-if="device.available">

          </p>
          <p v-else>
            {{ $tr('deviceNotConnected') }}
          </p>
        </KGridItem>
      </KGrid>
    </KModal>
  </ImmersivePage>

</template>


<script>

  import ImmersivePage from 'kolibri.coreVue.components.ImmersivePage';
  import BottomAppBar from 'kolibri.coreVue.components.BottomAppBar';
  import { NetworkLocationResource, FacilityResource, TaskResource } from 'kolibri.resources';
  import { now } from 'kolibri.utils.serverClock';
  import commonCoreStrings from 'kolibri.coreVue.mixins.commonCoreStrings';
  import { TaskTypes } from 'kolibri.utils.syncTaskUtils';
  import { PageNames } from '../../../../kolibri/plugins/facility/assets/src/constants';

  export default {
    name: 'EditDeviceSyncSchedule',
    components: {
      ImmersivePage,
      BottomAppBar,
    },
    mixins: [commonCoreStrings],
    props: {
      icon: {
        type: String,
        default: 'back',
      },
    },
    data() {
      return {
        removeDeviceModal: false,
        deviceName: null,
        device: [],
        now: now(),
        selectedItem: '',
        tasks: [],
        selectedDay: null,
        selectedTime: null,
        removeBtn: false,
      };
    },
    computed: {
      backRoute() {
        return { name: PageNames.MANAGE_SYNC_SCHEDULE };
      },
      pageHeight() {
        return {
          height: '80%',
          zIndex: -1,
        };
      },
      selectorStyle() {
        return {
          color: this.$themeTokens.text,
          backgroundColor: this.$themePalette.grey.v_200,
          borderRadius: '5px 5px 0px 0px',
          paddingTop: '5px',
          paddingLeft: '5px',
          width: '300px',
          marginLeft: '16px',
        };
      },
      selectArray() {
        return [
          { label: this.$tr('everyHour'), value: 0 },
          { label: this.$tr('everyDay'), value: 1 },
          { label: this.$tr('everyWeek'), value: 2 },
          { label: this.$tr('everyTwoWeeks'), value: 3 },
          { label: this.$tr('everyMonth'), value: 4 },
        ];
      },
      getDays() {
        const today = new Date();
        const daysOfWeek = [];
        const date = new Date(
          today.getFullYear(),
          today.getMonth(),
          today.getDate() + (7 - today.getDay())
        );
        for (let i = 0; i < 7; i++) {
          daysOfWeek.push({ label: this.$formatDate(date, { weekday: 'long' }), value: i });
          date.setDate(date.getDate() + 1);
        }
        return daysOfWeek;
      },

      SyncTime() {
        const endTime = new Date();
        endTime.setHours(24, 0, 0, 0);
        const interval = 30;

        const times = [];
        const time = new Date();
        time.setHours(0, 0, 0, 0);

        while (time < endTime) {
          times.push(this.$formatTime(time));

          time.setMinutes(time.getMinutes() + interval);
        }

        return times;
      },
    },
    beforeMount() {
      this.fetchDevice();
    },
    mounted() {
      this.timer = setInterval(() => {
        this.now = now();
      }, 10000);
    },
    beforeDestroy() {
      clearInterval(this.timer);
    },
    methods: {
      closeModal() {
        this.removeDeviceModal = false;
      },
      handleDeleteDevice() {
        this.removeDeviceModal = false;
        NetworkLocationResource.deleteModel({ id: this.$route.query.id })
          .then(() => {
            this.showSnackbarNotification('deviceRemove');
            history.back();
          })
          .catch(() => {
            this.showSnackbarNotification('deviceNotRemove');
          });
      },
      handleSaveSchedule() {
        FacilityResource.fetchModel({ id: this.$store.getters.activeFacilityId, force: true }).then(
          facility => {
            this.facility = { ...facility };
            TaskResource.startTask({
              type: TaskTypes.SYNCPEERFULL,
              facility: this.facility.id,
              device_id: this.device.id,
              baseurl: this.baseurl,
              enqueue_args: { enqueue_at: this.serverTime, repeat_interval: 2, repeat: 2 },
            })
              .then(() => {
                history.back();
                this.showSnackbarNotification('syncAdded');
              })
              .catch(() => {
                this.createTaskFailedSnackbar();
              });
          }
        );
      },

      cancelBtn() {
        this.$router.push({ name: PageNames.ManageSyncSchedule });
      },
      fetchDevice() {
        NetworkLocationResource.fetchModel({ id: this.$route.params.deviceId }).then(device => {
          this.device = device;
          TaskResource.list({ queue: 'facility_task' }).then(tasks => {
            this.tasks = tasks.filter(
              task =>
                task.extra_metadata.device_id === device.id &&
                task.facility_id === this.$store.getters.activeFacilityId
            );
            if (this.tasks.length != 0) {
              this.removeBtn = true;
            }
          });
        });
      },
    },
    $trs: {
      editSyncScheduleTitle: {
        message: 'Edit device sync schedule',
        context: 'Subtitle for the edit sync schedule page',
      },
      serverTime: {
        message: 'Server time:',
        context: 'Server time label',
      },
      checkboxLabel: {
        message: 'If scheduled sync fails, keep trying',
        context: 'Label for checkbox',
      },
      removeDevice: {
        message: 'Remove device',
        context: 'Title for the remove device modal',
      },
      removeDeviceWarning: {
        message: 'You are about to remove this device from the sync schedule.',
        context: 'Label to warn the user before removing the device',
      },
      deviceNotConnected: {
        message: 'This device is not currently connected to your network.',
        context: 'Message showing that the device is no longer on the network',
      },
      everyHour: {
        message: 'Every hour',
        context: 'Period for scheduling the sync between devices every hour',
      },
      everyDay: {
        message: 'Every day',
        context: 'Period for scheduling the sync between devices every day',
      },
      everyWeek: {
        message: 'Every week',
        context: 'Period for scheduling the sync between devices every week',
      },
      everyMonth: {
        message: 'Every month',
        context: 'Period for scheduling the sync between devices every month',
      },
      everyTwoWeeks: {
        message: 'Every two weeks',
        context: 'Period for scheduling the sync between devices every two weeks',
      },
      removeDeviceLabel: {
        message: 'Remove device from sync schedule',
        context: 'Button label for removing the device from the sync schedule',
      },
    },
  };

</script>


<style scoped>
  .spacing{
    margin-top:10px;
  }
  .loader{
    margin-top:5px;
  }
  .edit-sync-schedule{
    margin-left:20px;
  }
  .align-kselects{
    margin-left:16px;
  }


</style>
