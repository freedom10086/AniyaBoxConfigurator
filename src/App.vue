<script setup>
import {computed, onMounted, provide, ref, watch} from "vue";

import ImageEditor from './components/ImageEditor.vue';
import AlarmClock from "@/components/AlarmClock.vue";
import Manual from "@/components/Manual.vue";
import DeviceTimeClock from "@/components/DeviceTimeClock.vue";
import DeviceBaseInfo from "@/components/DeviceBaseInfo.vue";

let connectedDevice = ref(null);
let deviceTime = ref("-");
let bluetoothServer = null;
let bluetoothServerRef = ref(null);

let deviceConnected = computed(() => {
  return connectedDevice.value != null && connectedDevice.value.gatt.connected
})

provide('bluetoothServer', bluetoothServerRef);
provide('deviceConnected', deviceConnected);
provide('uartRx', uartRx);
provide('uartTx', uartTx);
provide('deviceTime', deviceTime);

let alarmLoaded = ref(false);

let alarmCfg = 0;
let alarmMin = 0;
let alarmHour = 0;
let alarmDayWeek = 0;

const BOX_SETTING_CMD_PING = 0
const BOX_SETTING_CMD_LOAD_CONFIG = 1
const BOX_SETTING_CMD_SET_TIME = 2
const BOX_SETTING_CMD_SET_ALARM = 3

function connectBluetooth() {
  let options = {
    filters: [
      {services: ['battery_service', 0x180A, 'environmental_sensing']},
      {name: 'aniya-box'},
      {namePrefix: 'aniya-box'}
    ],
    optionalServices: ['6e400001-b5a3-f393-e0a9-e50e24dcca9e']
  }

  // https://developer.chrome.com/docs/capabilities/bluetooth?hl=zh-cn
  navigator.bluetooth.requestDevice(options).then(function (device) {
    device.addEventListener('gattserverdisconnected', onDeviceDisconnected);
    console.log('名称：' + device.name, device);

    device.gatt.connect().then(server => {
      console.log("连接成功： ", server);

      bluetoothServer = server;
      bluetoothServerRef.value = server;
      connectedDevice.value = device;
    }).catch(err => {
      console.log("连接出现错误： " + err);
    })
  }).catch(function (error) {
    console.log("出现错误： " + error);
  });
}

function disConnectBluetooth() {
  if (!connectedDevice.value) {
    return;
  }
  if (connectedDevice.value.gatt.connected) {
    connectedDevice.value.gatt.disconnect();
    console.log('Disconnecting from Bluetooth Device...');
  }
}

function onDeviceDisconnected() {
  connectedDevice.value = null;
  console.log('> Bluetooth Device disconnected');
}

let autoDisconnectTimeoutId = null;
onMounted(() => {
  document.addEventListener("visibilitychange", () => {
    if (document.hidden) {
      console.log("start counting for disconnect");
      if (autoDisconnectTimeoutId == null) {
        // document.visibilityState !== 'visible'
        autoDisconnectTimeoutId = setTimeout(disConnectBluetooth, 20000);
      }
    } else {
      console.log("clear counting for disconnect");
      if (autoDisconnectTimeoutId != null) {
        window.clearTimeout(autoDisconnectTimeoutId);
        autoDisconnectTimeoutId = null;
      }
    }
  });
})

let updateDeviceTimeIntervalId;
watch(connectedDevice, async (newDev, oldDev) => {
  if (newDev != null && newDev.gatt.connected) {
    setTimeout(loadDeviceSetting, 1000);
    updateDeviceTimeIntervalId = setInterval(loadDeviceSetting, 12000);
  } else {
    clearInterval(updateDeviceTimeIntervalId);
  }
})

function loadDeviceSetting() {
  if (deviceConnected) {
    uartRx();
  }
}

function zeroPad(nr, base = 10) {
  let len = (String(base).length - String(nr).length) + 1;
  return len > 0 ? new Array(len).join('0') + nr : nr;
}

function uartRx() {
  bluetoothServer.getPrimaryService('6e400001-b5a3-f393-e0a9-e50e24dcca9e').then(service => {
    //console.log('获取主要服务成功uart', service);
    service.getCharacteristic('6e400003-b5a3-f393-e0a9-e50e24dcca9e').then(characteristic => {
      characteristic.readValue().then(value => {
        console.log("read uart ", value);
        let year = value.getUint8(0);
        let month = value.getUint8(1);
        let day = value.getUint8(2);
        let week = value.getUint8(3);
        let hour = value.getUint8(4);
        let minute = value.getUint8(5);
        let second = value.getUint8(6);

        const dayWeek = [
          "天",
          "一",
          "二",
          "三",
          "四",
          "五",
          "六",
        ];

        deviceTime.value = "20" + year + "-" + zeroPad(month) + "-" + zeroPad(day) + " " + zeroPad(hour) + ":" + zeroPad(minute) + ":" + zeroPad(second) + " 星期" + dayWeek[week];

        // bit 0 enable // bit 1 WADA mode 0 week 1 day // bit 2 af flag
        alarmCfg = value.getUint8(7);
        alarmMin = value.getUint8(8);
        alarmHour = value.getUint8(9);
        alarmDayWeek = value.getUint8(10);
        console.log("load device alarm alarmCfg:", alarmCfg, "alarmMin", alarmMin, "alarmHour", alarmHour, "alarmDayWeek", alarmDayWeek);

        alarmLoaded.value = true;

      }).catch(error => {
        console.error('read uart 错误:', error);
      })
    }).catch(error => {
      console.error('uart getCharacteristic 错误:', error);
    })
  }).catch(error => {
    console.error('获取主要服务错误uart:', error);
  });
}

function uartTx(data, callback) {
  bluetoothServer.getPrimaryService('6e400001-b5a3-f393-e0a9-e50e24dcca9e').then(service => {
    //console.log('获取主要服务成功uart', service);
    service.getCharacteristic('6e400002-b5a3-f393-e0a9-e50e24dcca9e').then(characteristic => {
      characteristic.writeValue(data).then(_ => {
        console.log("write uart success", data);
        if (callback !== undefined) {
          callback(true);
        }
      }).catch(error => {
        console.error('write uart 错误:', error);
        if (callback !== undefined) {
          callback(false, error);
        }
      })
    }).catch(error => {
      console.error('uart getCharacteristic 错误:', error);
      if (callback !== undefined) {
        callback(false, error);
      }
    })
  }).catch(error => {
    console.error('获取主要服务错误uart:', error);
    if (callback !== undefined) {
      callback(false, error);
    }
  });
}
</script>

<template>
  <main>
    <div v-if="!deviceConnected">
      <button @click="connectBluetooth">连接蓝牙</button>
      <p>长按第三个按键开启蓝牙</p>
    </div>
    <div v-else>
      <button @click="disConnectBluetooth" style="margin-right: 5px;">断开蓝牙</button>
      <label>已连接：{{ connectedDevice.name }}</label>

      <div class="row">
        <DeviceBaseInfo />
      </div>

      <div class="row" style="margin-top: 6px;" v-if="deviceTime !== '-'">
        <DeviceTimeClock/>
      </div>
    </div>

    <ImageEditor msg="hello" style="margin-top: 10px;" @upload-img-success="()=>{}"/>

    <div v-if="deviceConnected">
      <AlarmClock v-if="alarmLoaded" :alarm-cfg="alarmCfg" :min-alarm="alarmMin" :hour-alarm="alarmHour"
                  :day-week-alarm="alarmDayWeek"/>
    </div>

    <div style="margin-top: 12px;">
      <Manual/>
    </div>
  </main>
</template>

<style scoped>
.row {
  display: flex;
  flex-direction: row;
}
</style>
