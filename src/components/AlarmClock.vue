<script setup>

import {inject, onMounted, ref, watch} from "vue";

const props = defineProps({
  alarmCfg: {
    type: Number,
    required: true
  },
  minAlarm: {
    type: Number,
    required: true
  },
  hourAlarm: {
    type: Number,
    required: true
  },
  dayWeekAlarm: {
    type: Number,
    required: true
  }
})

const BOX_SETTING_CMD_SET_ALARM = 3

let date = new Date();
let year = date.getFullYear();
let month = date.getMonth();
let day = date.getDate();
let week = date.getDay();
let hour = date.getHours();
let minute = date.getMinutes();

const alarmEnableModel = defineModel('alarmEnableModel', {default: true});
const weekModeModel = defineModel('weekModeModel', {default: 'by-week'});
const everyDayModel = defineModel('everyDayModel', {default: false});
const everyWeekModel = defineModel('everyWeekModel', {default: false});
const dateModel = defineModel('dateModel', {default: ""});
const timeModel = defineModel('timeModel', {default: ""});
const checkedWeeksModel = defineModel('checkedWeeksModel', {default: []});
const weekSingleSelectType = ref("checkbox");

const uartTx = inject('uartTx');

onMounted(() => {
  console.log("alarm clock onMounted");

  alarmEnableModel.value = (props.alarmCfg & 0x01) === 1;
  weekModeModel.value = ((props.alarmCfg >> 1) & 0x01) === 1 ? "by-day" : 'by-week';
  weekSingleSelectType.value = ((props.alarmCfg >> 3) & 0x01) === 1 ? "radio" : 'checkbox';

  let loadMin = props.minAlarm & 0b01111111;
  let loadHour = props.hourAlarm & 0b01111111;

  let loadDayWeek = props.dayWeekAlarm & 0b01111111;
  if (weekModeModel.value === "by-day") {
    console.log("alarm mode:by-day");
    everyDayModel.value = (props.dayWeekAlarm & 0b10000000) > 0;
    dateModel.value = year + "-" + zeroPad((month + 1), 10) + "-" + zeroPad(loadDayWeek, 10);
  } else {
    console.log("alarm mode:by-week");
    // week mode
    dateModel.value = year + "-" + zeroPad((month + 1), 10) + "-" + zeroPad(day, 10);
    if ((props.dayWeekAlarm & 0b10000000) > 0) {
      everyWeekModel.value = true;
      checkedWeeksModel.value = ['0', '1', '2', '3', '4', '5', '6'];
    } else {
      let i = 0;
      let arr = [];
      while (loadDayWeek > 0) {
        if ((loadDayWeek & 0x01) > 0) {
          arr.push(i.toString());
        }
        i++;
        loadDayWeek = loadDayWeek >> 1;
      }

      checkedWeeksModel.value = arr;
    }
  }

  timeModel.value = zeroPad(loadHour, 10) + ":" + zeroPad(loadMin, 10);
})

watch(checkedWeeksModel, (newValue, oldValue) => {
  console.log("checked weeks", oldValue, "=>", newValue);

})

function setAlarmSubmit() {
  let weekMode = weekModeModel.value === 'by-week';
  let alarmEnable = alarmEnableModel.value;
  console.log("alarm enable", alarmEnable, "weekMode:", weekMode);

  let resultDayWeek = 0;
  if (weekMode) {
    let checkedWeeks = checkedWeeksModel.value;
    if (checkedWeeks.length === 0 && alarmEnable && !everyWeekModel.value) {
      alert("请选择星期");
      return
    }
    for (let checkedWeek of checkedWeeks) {
      resultDayWeek = resultDayWeek | (0x01 << parseInt(checkedWeek));
    }
    if (everyWeekModel.value) {
      resultDayWeek = 0x80 | resultDayWeek;
    }
  } else {
    if (weekModeModel.value === 'by-day') {
      if (everyDayModel.value) {
        resultDayWeek = 0x80;
      } else {
        let dateValue = dateModel.value;
        if (dateValue === "" && alarmEnable) {
          alert("请选择日期");
          return
        }

        let month = parseInt(dateValue.split("-")[1]);
        let day = parseInt(dateValue.split("-")[2]);
        resultDayWeek = day;
      }
    } else {
      alert("不支持的模式:" + weekModeModel.value);
      return
    }
  }

  if (timeModel.value.length === 0 && alarmEnable) {
    alert("请选择时间:" + weekModeModel.value);
    return
  }

  let hourInput = timeModel.value.split(":")[0];
  let minuteInput = timeModel.value.split(":")[1];

  // bit 0 enable // bit 1 0 week 1 day // bit 2 af // bit 3 week single
  let alarmCfg = (alarmEnableModel.value) ? 0x01 : 0x00;
  if (!weekMode) {
    // day mode
    alarmCfg |= (0x01 << 1)
  }

  console.log("===submit alarm===");
  console.log("alarmCfg", alarmCfg, "alarmMin", minuteInput, "alarmHour", hourInput, "alarmWeekDay", resultDayWeek);
  let data = new Uint8Array([BOX_SETTING_CMD_SET_ALARM, alarmCfg, minuteInput, hourInput, resultDayWeek]);
  uartTx(data, (success, err) => {
    if (success) {
      alert("设置成功");
    } else {
      alert("操作失败" + err);
    }
  });
}


function zeroPad(nr, base) {
  let len = (String(base).length - String(nr).length) + 1;
  return len > 0 ? new Array(len).join('0') + nr : nr;
}

// bit 0 1 2 3 4  5  6  7
//     1 2 4 8 10 20 40 x
function hex2bcd(x) {
  let y;
  y = (Math.floor(x / 10)) << 4;
  y = y | (x % 10);
  return (y);
}

function bcd2hex(bcd) {
  let dec = 0;
  let mult;
  for (mult = 1; bcd; bcd = bcd >> 4, mult *= 10) {
    dec += (bcd & 0x0f) * mult;
  }
  return dec;
}

</script>

<template>
  <div style="font-weight: bold; margin-top: 12px;">
    闹钟
  </div>
  <div>
    <label>开启闹钟
      <input type="checkbox" v-model="alarmEnableModel"></label>
  </div>
  <div v-if="alarmEnableModel">
    <div>
      <label>重复</label>
      <label class="ml1">按周
        <input type="radio" name="mode" value="by-week" v-model="weekModeModel"/></label>
      <label class="ml1">按天
        <input type="radio" name="mode" value="by-day" v-model="weekModeModel"/></label>
    </div>
    <div v-if="weekModeModel==='by-week'">
      <label>星期</label>
      <label class="ml1">每周<input type="checkbox" v-model="everyWeekModel"></label>
      <label class="ml1">日
        <input :type="weekSingleSelectType" v-model="checkedWeeksModel" value="0" :disabled="everyWeekModel"></label>
      <label class="ml1">一
        <input :type="weekSingleSelectType" v-model="checkedWeeksModel" value="1" :disabled="everyWeekModel"></label>
      <label class="ml1">二
        <input :type="weekSingleSelectType" v-model="checkedWeeksModel" value="2" :disabled="everyWeekModel"></label>
      <label class="ml1">三
        <input :type="weekSingleSelectType" v-model="checkedWeeksModel" value="3" :disabled="everyWeekModel"></label>
      <label class="ml1">四
        <input :type="weekSingleSelectType" v-model="checkedWeeksModel" value="4" :disabled="everyWeekModel">
      </label>
      <label class="ml1">五
        <input :type="weekSingleSelectType" v-model="checkedWeeksModel" value="5" :disabled="everyWeekModel"></label>
      <label class="ml1">六
        <input :type="weekSingleSelectType" v-model="checkedWeeksModel" value="6" :disabled="everyWeekModel"></label>
    </div>
    <div v-if="weekModeModel==='by-day'">
      <label>每天
        <input type="checkbox" v-model="everyDayModel"></label>
      <label v-if="!everyDayModel">日期：
        <input type="date" v-model="dateModel" :min="year + '-'+zeroPad((month+1), 10)+'-'+ zeroPad(day, 10) "
               :max="year + (month === 12 ? 1 : 0) + '-'+zeroPad((month + 2) % 12, 10)+'-'+ zeroPad(Math.max(1,day - 1), 10)"></label>
    </div>
    <div>
      <label>时间：
        <input type="time" v-model="timeModel"></label>
    </div>
  </div>
  <div style="margin-top: 8px;">
    <button @click="setAlarmSubmit">保存闹钟</button>
  </div>
</template>

<style scoped>
.ml1 {
  margin-left: 8px;
}
</style>