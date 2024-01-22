<script setup>
import {onMounted, ref} from "vue";

let currentTime = ref("-");

function displayCurrentTime() {
  let date = new Date();
  let year = date.getFullYear();
  let month = date.getMonth();
  let day = date.getDate();
  let week = date.getDay();
  let hour = date.getHours();
  let minute = date.getMinutes();
  let second = date.getSeconds();

  const dayWeek = [
    "天",
    "一",
    "二",
    "三",
    "四",
    "五",
    "六",
  ];

  currentTime.value = year + "-" + zeroPad((month + 1), 10) + "-" + zeroPad(day) + " " + zeroPad(hour) + ":" + zeroPad(minute) + ":" + zeroPad(second) + " " + dayWeek[week];
}

function zeroPad(nr, base = 10) {
  let len = (String(base).length - String(nr).length) + 1;
  return len > 0 ? new Array(len).join('0') + nr : nr;
}

onMounted(() => {
  console.log("CurrentTimeClock onMounted");
  setInterval(displayCurrentTime, 1000);
})

</script>

<template>
  <label style="font-weight: bold">当前时间：</label>
  <div id="current-time">{{ currentTime }}</div>
</template>

<style scoped>

</style>