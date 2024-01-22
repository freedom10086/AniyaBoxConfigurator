<script setup>
import {inject, onMounted} from "vue";
import CurrentTimeClock from "@/components/CurrentTimeClock.vue";

const BOX_SETTING_CMD_SET_TIME = 2
const uartTx = inject('uartTx');
const deviceTime = inject('deviceTime');

function syncTime() {
  let date = new Date();
  let year = date.getFullYear();
  let month = date.getMonth();
  let day = date.getDate();
  let week = date.getDay();
  let hour = date.getHours();
  let minute = date.getMinutes();
  let second = date.getSeconds();

  let data = new Uint8Array([BOX_SETTING_CMD_SET_TIME, year % 100, month + 1, day, week, hour, minute, second]);
  uartTx(data, (success, err) => {
    if (!success) {
      alert("同步时间失败" + err);
    }
  });
}

onMounted(() => {
  let deviceTimeStr = deviceTime.value;

  // compare with current time if diff auto sync time
  let date = new Date();
  let realYear = date.getFullYear();
  let realMonth = date.getMonth() + 1;
  let realDay = date.getDate();
  let realHour = date.getHours();
  let realMinute = date.getMinutes();

  // 2024-01-02 18:18:18
  let deviceYear = parseInt(deviceTimeStr.split(" ")[0].split("-")[0]);
  let deviceMonth = parseInt(deviceTimeStr.split(" ")[0].split("-")[1]);
  let deviceDay = parseInt(deviceTimeStr.split(" ")[0].split("-")[2]);
  let deviceHour = parseInt(deviceTimeStr.split(" ")[1].split(":")[0]);
  let deviceMinute = parseInt(deviceTimeStr.split(" ")[1].split(":")[1]);

  if (realYear !== deviceYear || realMonth !== deviceMonth || realDay !== deviceDay
      || realHour !== deviceHour || realMinute !== deviceMinute) {
    console.log(deviceTimeStr, deviceYear + "-" + deviceMonth + "-" + deviceDay + " " + deviceHour + ":" + deviceMinute);
    console.log("=====auto sync device time=====");
    syncTime();
  }
})
</script>

<template>
  <label style="font-weight: bold">设备时间：</label>
  <div id="device-time" style="margin-right: 10px">{{ deviceTime }}</div>
  <CurrentTimeClock/>
  <button style="margin-left: 10px" @click="syncTime">同步时间</button>
</template>

<style scoped>

</style>