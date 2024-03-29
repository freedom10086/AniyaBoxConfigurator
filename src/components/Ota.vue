<script setup>
import {inject, onMounted, ref} from "vue";

const deviceConnected = inject("deviceConnected");
const bluetoothServer = inject('bluetoothServer');

const COMMAND_ID_START = 0x0001
const COMMAND_ID_END = 0x0002
const COMMAND_ID_ACK = 0x0003

const COMMAND_ACK_ACCEPT = 0x0000
const COMMAND_ACK_REFUSE = 0x0001

const BIN_ACK_SUCCESS = 0x0000
const BIN_ACK_CRC_ERROR = 0x0001
const BIN_ACK_SECTOR_INDEX_ERROR = 0x0002
const BIN_ACK_PAYLOAD_LENGTH_ERROR = 0x0003

const MTU_SIZE = 517
const MTU_STATUS_FAILED = 20000
const EXPECT_PACKET_SIZE = 463

const SERVICE_UUID = "00008018-0000-1000-8000-00805f9b34fb";
const CHAR_RECV_FW_UUID = "00008020-0000-1000-8000-00805f9b34fb";
const CHAR_PROGRESS_UUID = "00008021-0000-1000-8000-00805f9b34fb";
const CHAR_COMMAND_UUID = "00008022-0000-1000-8000-00805f9b34fb";
const CHAR_CUSTOMER_UUID = "00008023-0000-1000-8000-00805f9b34fb";

function crc(data, offset, length) {
  let crc16 = 0;
  for (let cur = offset; cur < length; ++cur) {
    let aInt = data[cur] & 0xff;
    crc16 ^= aInt << 8;
    for (let i = 0; i < 8; ++i) {
      if ((crc16 & 0x8000) != 0) {
        crc16 = ((crc16 << 1) ^ 0x1021) & 0xffff;
      } else {
        crc16 = (crc16 << 1) & 0xffff;
      }
    }
  }

  return crc16;
}

const updateInit = ref(false);

let charRecvFw, charProgress, charCmd, charCust;

async function findUpdateService() {
  try {
    let service = await bluetoothServer.value.getPrimaryService(SERVICE_UUID);
    let characteristics = service.getCharacteristics([CHAR_RECV_FW_UUID, CHAR_PROGRESS_UUID, CHAR_COMMAND_UUID, CHAR_CUSTOMER_UUID]);
    // 对获取到的特征进行遍历或操作
    characteristics.forEach(characteristic => {
      console.log('特征 UUID:', characteristic.uuid);
      // 这里可以执行与特征相关的操作
      if (characteristic.uuid === CHAR_RECV_FW_UUID) {
        charRecvFw = characteristic;
      } else if (characteristic.uuid === CHAR_PROGRESS_UUID) {
        charProgress = characteristic;
      }
      if (characteristic.uuid === CHAR_COMMAND_UUID) {
        charCmd = characteristic;
      }
      if (characteristic.uuid === CHAR_CUSTOMER_UUID) {
        charCust = characteristic;
      }

      updateInit.value = true;
    });

  } catch (err) {
    console.error('获取升级service failed:', err);
    return
  }

  bluetoothServer.value.getPrimaryService(SERVICE_UUID).then(service => {

  }).catch(error => {
    console.error('无升级service:', error);
  });
}

onMounted(() => {
  findUpdateService();


})

function startOta() {
  
}

</script>

<template>
  <button v-if="updateInit" @click="startOta">升级</button>
</template>

<style scoped>

</style>