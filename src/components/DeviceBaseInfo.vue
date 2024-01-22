<script setup>
import {inject, onMounted, ref} from "vue";

let batteryLevel = ref(0);
let temperature = ref(0.0);
let humility = ref(0.0);

const bluetoothServer = inject('bluetoothServer');

onMounted(() => {
  setTimeout(loadBatteryLevel, 400);
  setTimeout(loadTemp, 800);
  setTimeout(loadHumility, 1200);
})

function loadBatteryLevel() {
  bluetoothServer.value.getPrimaryService('battery_service').then(service => {
    // 获取主要服务成功
    console.log('获取主要服务成功', service);
    service.getCharacteristic('battery_level').then(characteristic => {
      characteristic.readValue().then(value => {
        let level = value.getUint8(0);
        console.log('> Battery Level is ' + level + '%');
        batteryLevel.value = level;
      }).catch(error => {
        console.error('readValue 错误:', error);
      })
    }).catch(error => {
      console.error('getCharacteristic 错误:', error);
    })
  }).catch(error => {
    // 获取主要服务失败
    console.error('获取主要服务错误:', error);
  });
}

function loadTemp() {
  bluetoothServer.value.getPrimaryService('environmental_sensing')
      .then(service => {
        return service.getCharacteristic('temperature');
      })
      .then(characteristic => {
        return characteristic.readValue()
      })
      .then(value => {
        console.log("read temperature", value);
        let temp = value.getInt16(0) / 100.0;
        console.log('> Temperature is ' + temp + '℃');
        temperature.value = temp;
      })
      .catch(error => {
        // 获取主要服务失败
        console.error('read Temperature error:', error);
      });
}

function loadHumility() {
  bluetoothServer.value.getPrimaryService('environmental_sensing')
      .then(service => {
        return service.getCharacteristic('humidity');
      })
      .then(characteristic => {
        return characteristic.readValue()
      })
      .then(value => {
        console.log("read humility", value);
        let hum = value.getUint16(0) / 100.0;
        console.log('> Humidity is ' + hum + '%');
        humility.value = hum;
      })
      .catch(error => {
        // 获取主要服务失败
        console.error('read Temperature error:', error);
      });
}

</script>

<template>
  <label>电池:{{ batteryLevel }}</label>
  <label>温度:{{ temperature }}</label>
  <label>湿度:{{ humility }}</label>
</template>

<style scoped>

</style>