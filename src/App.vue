<script setup>
import {computed, onMounted, provide, ref} from "vue";
import moment from 'moment'
import CPU from "@/components/CPU.vue";
import Mem from "@/components/Mem.vue";
import NetIn from "@/components/NetIn.vue";
import NetOut from "@/components/NetOut.vue";
import axios from "axios";
import { Message } from "@arco-design/web-vue";
import StatsCard from "@/components/StatsCard.vue";
import {formatBytes, formatTimeStamp, formatUptime, calculateRemainingDays} from '@/utils/utils'
import HeaderLocale from "@/components/HeaderLocale.vue";
import {useI18n} from "vue-i18n";

const { t } = useI18n()

const socketURL = ref('')
const apiURL = ref('')

const theme = window.localStorage.getItem('theme') || 'light'
const dark = ref(theme !== 'light')

const handleChangeDark = () => {
  dark.value = !dark.value

  if (dark.value) {
    window.localStorage.setItem('theme','dark')
    document.body.setAttribute('arco-theme', 'dark')
  } else {
    // 恢复亮色主题
    window.localStorage.setItem('theme','light')
    document.body.removeAttribute('arco-theme');
   }
}

const area = ref([])
const selectArea = ref('all')

const type = ref('all')

const data = ref([])

const selectHost = ref('')

const charts = ref({})

const cpuRef = ref(null)
const memRef = ref(null)
const netInRef = ref(null)
const netOutRef = ref(null)

const host = computed(() => {
  if (selectArea.value === 'all') {
    return data.value
  }

  return data.value.filter(item => item.Host.Name.slice(0, 2) === selectArea.value)
})

const hosts = computed(() => {
  if (type.value === 'all') {
    return host.value
  } else if (type.value === 'online') {
    return host.value.filter(item => item.status)
  } else {
    return host.value.filter(item => !item.status)
  }
})

const stats = computed(() => {
  const online = host.value.filter(item => item.status)
  let bandwidth_up = 0
  let bandwidth_down = 0
  let traffic_up = 0
  let traffic_down = 0

  host.value.forEach((item) => {
    bandwidth_up += item.State.NetOutSpeed
    bandwidth_down += item.State.NetInSpeed
    traffic_up += item.State.NetOutTransfer
    traffic_down += item.State.NetInTransfer
  })

  return {
    total: host.value.length,
    online: online.length,
    offline: host.value.length - online.length,
    bandwidth_up: bandwidth_up,
    bandwidth_down: bandwidth_down,
    traffic_up: traffic_up,
    traffic_down: traffic_down
  }
})

let socket = null

let nowtime = (Math.floor(Date.now() / 1000))

const fetchConfig = async () => {
  try {
    const res = await axios.get('/config.json')
    socketURL.value = res.data.socket
    apiURL.value = res.data.apiURL
  } catch (e) {
    Message.error(t('get-config-error'))
  }
}

const initScoket = async () => {
  await fetchConfig()

  socket = new WebSocket(socketURL.value);  // 替换为实际的 WebSocket 服务器 URL

  socket.onmessage = function(event) {
    try {
      const message = event.data;
      const res = JSON.parse(message.replace('data: ', ''))
      if (res != null) {
        area.value = Array.from(new Set(res.map(item => item.Host.Name.slice(0, 2))))
      }
      data.value = res.map((host) => {
        if (!charts.value[host.Host.Name]) {
          charts.value[host.Host.Name] = {
            cpu: [],
            mem: [],
            net_in: [],
            net_out: []
          }
        }

        if (charts.value[host.Host.Name].cpu.length == 2) {
          charts.value[host.Host.Name].cpu = charts.value[host.Host.Name].cpu.slice(1)
          charts.value[host.Host.Name].mem = charts.value[host.Host.Name].mem.slice(1)
          charts.value[host.Host.Name].net_in = charts.value[host.Host.Name].net_in.slice(1)
          charts.value[host.Host.Name].net_out = charts.value[host.Host.Name].net_out.slice(1)
        }

        charts.value[host.Host.Name].cpu.push([host.TimeStamp * 1000, host.State.CPU])
        charts.value[host.Host.Name].mem.push([host.TimeStamp * 1000, host.State.MemUsed])
        charts.value[host.Host.Name].net_in.push([host.TimeStamp * 1000, host.State.NetOutSpeed])
        charts.value[host.Host.Name].net_out.push([host.TimeStamp * 1000, host.State.NetInSpeed])

        return {
          ...host,
          status: (nowtime - host.TimeStamp <= 10) ? 1 : 0
        }
      })

      setTimeout(() => sendPing(), 1000)

    } catch (error) {
      console.error(t('ws-error'), error);
    }
  };

  socket.onopen = function () {
    sendPing()
  }

  socket.onclose = function () {
    Message.warning(t('ws-error-reconnect'))

    initScoket()
  }
}

const sendPing = () => {
  nowtime = (Math.floor(Date.now() / 1000))
  socket.send('ping')
}

onMounted(async() => {
  if (dark.value) {
    document.body.setAttribute('arco-theme', 'dark')
  }

  await initScoket()
  handleFetchHostInfo()
})

const progressStatus = (value) => {
  if (value < 80) {
    return 'success';
  } else if (value < 90) {
    return 'warning';
  } else {
    return 'danger';
  }
}

const handleSelectArea = (area) => {
  selectArea.value = area
}

const handleSelectHost = (host) => {
  if (selectHost.value === host) {
    selectHost.value = ''
    return
  }

  selectHost.value = host
}

// 删除主机
const deleteVisible = ref(false)
const authSecret = ref('')
const deleteHostName = ref('')

const handleShowDelete = (name) => {
  authSecret.value =  window.localStorage.getItem('auth_secret') || ''
  deleteHostName.value = name
  deleteVisible.value = true
}

const handleDeleteHost = async () => {
  try {
    await axios.post(apiURL.value + '/delete', {
      "auth_secret": authSecret.value,
      "name": deleteHostName.value
    })

    window.localStorage.setItem('auth_secret', authSecret.value)

    Message.success(t('remove-success'))

    deleteVisible.value = false
  } catch (e) {
    Message.error(t('remove-fail'))
  }
}

const handleClose = () => {
  deleteVisible.value = false
}

const hostInfo = ref({})

const handleFetchHostInfo = async () => {
  try {
    const res = await axios.get(apiURL.value + '/info')
    res.data.forEach((item) => {
      hostInfo.value[item.name] = item
    })
  } catch (e) {
    // Message.error('删除失败，管理密钥错误')
  }
}

const handleChangeType = (value) => {
  type.value = value
}

const editHostName = ref('')
const duetime = ref(0)
const buy_url = ref('')
const seller = ref('')
const price = ref('')

const editVisible = ref(false)

const handleShowEdit = (name) => {
  if (!hostInfo.value[name]) {
    editHostName.value = name
    duetime.value = 0
    buy_url.value = ''
    seller.value = ''
    price.value = ''
    editVisible.value = true
    authSecret.value = window.localStorage.getItem('auth_secret') || ''
    return
  }

  editHostName.value = name
  duetime.value = hostInfo.value[name].due_time
  buy_url.value = hostInfo.value[name].buy_url
  seller.value = hostInfo.value[name].seller
  price.value = hostInfo.value[name].price
  editVisible.value = true
  authSecret.value = window.localStorage.getItem('auth_secret') || ''

}

const handleEditHost = async () => {
  try {
    await axios.post(apiURL.value + '/info', {
      "name": editHostName.value,
      "due_time": new Date(duetime.value).getTime(),
      "buy_url": buy_url.value,
      "seller": seller.value,
      "auth_secret": authSecret.value,
      "price": price.value
    })

    window.localStorage.setItem('auth_secret', authSecret.value)

    Message.success(t('edit-success'))

    handleFetchHostInfo()

    editVisible.value = false
  } catch (e) {
    Message.error(t('edit-fail'))
  }

}

const handleEditClose = () => {
  editVisible.value = false
}

provide('handleChangeType', handleChangeType)

</script>

<template>
  <div class="max-container">
    <div class="header">
      <a class="logo" href="#">
        <svg class="arco-icon" viewBox="0 0 256 256" fill="none" xmlns="http://www.w3.org/2000/svg">
          <path d="M61.9999 74.6666C61.9999 74.6666 57.1585 81.3609 51.3655 92.4445M51.3655 92.4445C42.7381 108.951 32 135.193 32 163.555C62 163.555 79.9999 181.333 79.9999 181.333C79.9999 181.333 91.9999 234.667 128 234.667C164 234.667 176 181.333 176 181.333C176 181.333 194 163.555 224 163.555C224 135.193 213.262 108.951 204.635 92.4445M51.3655 92.4445C51.3655 92.4445 20 68.7398 51.3655 21.3333C61.9999 27.2593 92 50.963 92 50.963C92 50.963 110 39.1111 128 39.1111C146 39.1111 164 50.963 164 50.963C164 50.963 194 27.2593 206 21.3333C236 68.742 204.635 92.4445 204.635 92.4445M194 74.6666C194 74.6666 198.842 81.3609 204.635 92.4445" stroke="url(#paint0_linear_55_29)" stroke-width="20" stroke-linecap="round" stroke-linejoin="round"/>
          <path d="M117.333 192H128M128 192H138.667M128 192V202.667" stroke="url(#paint1_linear_55_29)" stroke-width="20" stroke-linecap="round" stroke-linejoin="round"/>
          <path d="M90.6666 133.333L106.667 149.333" stroke="url(#paint2_linear_55_29)" stroke-width="20" stroke-linecap="round" stroke-linejoin="round"/>
          <path d="M165.333 133.333L149.333 149.333" stroke="url(#paint3_linear_55_29)" stroke-width="20" stroke-linecap="round" stroke-linejoin="round"/>
          <defs>
            <linearGradient id="paint0_linear_55_29" x1="128" y1="21.3333" x2="128" y2="234.667" gradientUnits="userSpaceOnUse">
              <stop stop-color="#ED6D68"/>
              <stop offset="1" stop-color="#AC1B1B"/>
            </linearGradient>
            <linearGradient id="paint1_linear_55_29" x1="128" y1="192" x2="128" y2="202.667" gradientUnits="userSpaceOnUse">
              <stop stop-color="#ED6D68"/>
              <stop offset="1" stop-color="#AC1B1B"/>
            </linearGradient>
            <linearGradient id="paint2_linear_55_29" x1="98.6666" y1="133.333" x2="98.6666" y2="149.333" gradientUnits="userSpaceOnUse">
              <stop stop-color="#ED6D68"/>
              <stop offset="1" stop-color="#AC1B1B"/>
            </linearGradient>
            <linearGradient id="paint3_linear_55_29" x1="157.333" y1="133.333" x2="157.333" y2="149.333" gradientUnits="userSpaceOnUse">
              <stop stop-color="#ED6D68"/>
              <stop offset="1" stop-color="#AC1B1B"/>
            </linearGradient>
          </defs>
        </svg>
        <span>Acck Monitor</span>
        <small style="font-weight: 400;opacity: .8"> ｜ {{ $t('title') }}</small>
      </a>
      <a-space>
        <HeaderLocale />
        <a-button class="theme-btn" :shape="'round'" @click="handleChangeDark">
          <template #icon>
            <icon-sun-fill v-if="!dark" />
            <icon-moon-fill v-else />
          </template>
        </a-button>
      </a-space>
    </div>
    <div class="area-tabs">
      <div class="area-tab-item" :class="selectArea === 'all' ? 'is-active' : ''" @click="handleSelectArea('all')">
        {{$t('all-area')}}
      </div>
      <div class="area-tab-item" :class="selectArea === item ? 'is-active' : ''" v-for="(item, index) in area" :key="item" @click="handleSelectArea(item)">
        <span :class="`flag-icon flag-icon-${item.replace('UK', 'GB').toLowerCase()}`" style="margin-right: 3px;"></span> {{item}}
      </div>
    </div>
    <StatsCard :type="type" :stats="stats" @handleChangeType="handleChangeType" />
    <div class="monitor-card">
      <div class="monitor-item" :class="selectHost === item.Host.Name ? 'is-active' : ''" v-for="(item, index) in hosts" @click="handleSelectHost(item.Host.Name)" :key="index">
        <div class="name">
          <div class="title">
            <span :class="`flag-icon flag-icon-${item.Host.Name.slice(0, 2).replace('UK', 'GB').toLowerCase()}`"></span>
            {{item.Host.Name}}
          </div>
          <div class="status" :class="item.status ? 'online' : 'offline'">
            <span>{{item.status  ? $t('online') : $t('offline')}}</span>
            <span style="margin-left: 6px;">{{formatUptime(item.State.Uptime)}}</span>
          </div>
        </div>
        <div class="platform">
          <div class="monitor-item-title">{{ $t('system') }}</div>
          <div class="monitor-item-value">{{item.Host.Platform}} {{item.Host.PlatformVersion}}</div>
        </div>
        <div class="cpu">
          <div class="monitor-item-title">CPU</div>
          <div class="monitor-item-value">{{item.State.CPU.toFixed(2) + '%'}}</div>
          <a-progress class="monitor-item-progress" :status="progressStatus(item.State.CPU)" :percent="item.State.CPU/100" :show-text="false" style="width: 60px" />
        </div>
        <div class="mem">
          <div class="monitor-item-title">{{ $t('memory') }}</div>
          <div class="monitor-item-value">{{(item.State.MemUsed / item.Host.MemTotal * 100).toFixed(2) + '%'}}</div>
          <a-progress class="monitor-item-progress" :status="progressStatus(item.State.MemUsed / item.Host.MemTotal * 100)" :percent="item.State.MemUsed / item.Host.MemTotal" :show-text="false" style="width: 60px" />
        </div>
        <div class="network">
          <div class="monitor-item-title">{{ $t('network') }} (IN|OUT)</div>
          <div class="monitor-item-value">{{`${formatBytes(item.State.NetInSpeed)}/s | ${formatBytes(item.State.NetOutSpeed)}/s`}}</div>
        </div>
        <div class="average">
          <div class="monitor-item-title">{{ $t('load') }} (1|5|15)</div>
          <div class="monitor-item-value">{{`${item.State.Load1} | ${item.State.Load5} | ${item.State.Load15}`}}</div>
        </div>
        <div class="uptime">
          <div class="monitor-item-title">{{ $t('report-time') }}</div>
          <div class="monitor-item-value">{{formatTimeStamp(item.TimeStamp)}}</div>
        </div>
        <div class="detail" v-if="selectHost === item.Host.Name">
          <a-row>
            <a-col :span="10" :xs="24" :sm="24" :md="10" :lg="10" :sl="10">
              <div class="detail-item-list">
                <div class="detail-item">
                  <div class="name">{{ $t('hostname') }}</div>
                  <div class="value">{{item.Host.Name}}</div>
                </div>
                <div class="detail-item">
                  <div class="name">{{ $t('area') }}</div>
                  <div class="value">
                    <span :class="`flag-icon flag-icon-${item.Host.Name.slice(0, 2).replace('UK', 'GB').toLowerCase()}`"></span>
                    {{item.Host.Name.slice(0, 2).toUpperCase()}}
                  </div>
                </div>
                <div class="detail-item">
                  <div class="name">{{ $t('system') }}</div>
                  <div class="value">{{item.Host.Platform}} {{item.Host.PlatformVersion}}</div>
                </div>
                <div class="detail-item">
                  <div class="name">{{ $t('arch') }}</div>
                  <div class="value">{{item.Host.Arch}}</div>
                </div>
                <div class="detail-item">
                  <div class="name">{{ $t('virtualization') }}</div>
                  <div class="value">{{item.Host.Virtualization || '-'}}</div>
                </div>
                <div class="detail-item">
                  <div class="name">CPU</div>
                  <div class="value">{{item.Host.CPU.join(',')}}</div>
                </div>
                <div class="detail-item">
                  <div class="name">CPU{{ $t('use') }}</div>
                  <div class="value">{{item.State.CPU.toFixed(2) + '%'}}</div>
                </div>
                <div class="detail-item">
                  <div class="name">{{ $t('memory') }}</div>
                  <div class="value">{{(item.State.MemUsed / item.Host.MemTotal * 100).toFixed(2) + '%'}} ({{formatBytes(item.State.MemUsed)}} / {{formatBytes(item.Host.MemTotal)}})</div>
                </div>
                <div class="detail-item">
                  <div class="name">{{ $t('swap') }}</div>
                  <div class="value">{{formatBytes(item.State.SwapUsed)}} / {{formatBytes(item.Host.SwapTotal)}}</div>
                </div>
                <div class="detail-item">
                  <div class="name">{{ $t('network') }}（IN|OUT）</div>
                  <div class="value">{{`${formatBytes(item.State.NetInSpeed)}/s | ${formatBytes(item.State.NetOutSpeed)}/s`}}</div>
                </div>
                <div class="detail-item">
                  <div class="name">{{ $t('load') }}(1|5|15)</div>
                  <div class="value">{{`${item.State.Load1} | ${item.State.Load5} | ${item.State.Load15}`}}</div>
                </div>
                <div class="detail-item">
                  <div class="name">{{ $t('bandwidth') }}↑|↓</div>
                  <div class="value">{{formatBytes(item.State.NetOutTransfer)}} | {{formatBytes(item.State.NetInTransfer)}}</div>
                </div>
                <div class="detail-item">
                  <div class="name">{{ $t('uptime') }}</div>
                  <div class="value">{{formatTimeStamp(item.Host.BootTime)}}</div>
                </div>
                <div class="detail-item">
                  <div class="name">{{ $t('report-time') }}</div>
                  <div class="value">{{formatTimeStamp(item.TimeStamp)}}</div>
                </div>
                <div class="detail-item" v-if="hostInfo[item.Host.Name] && hostInfo[item.Host.Name].seller">
                  <div class="name">{{ $t('isp-name') }}</div>
                  <div class="value">{{hostInfo[item.Host.Name].seller}}</div>
                </div>
                <div class="detail-item" v-if="hostInfo[item.Host.Name] && hostInfo[item.Host.Name].price">
                  <div class="name">{{ $t('host-price') }}</div>
                  <div class="value">{{hostInfo[item.Host.Name].price}}</div>
                </div>
                <div class="detail-item" v-if="hostInfo[item.Host.Name] && hostInfo[item.Host.Name].due_time">
                  <div class="name">{{ $t('due-time') }}</div>
                  <div class="value">{{moment(hostInfo[item.Host.Name].due_time).format('YYYY-MM-DD')}}</div>
                </div>
                <div class="detail-item" v-if="hostInfo[item.Host.Name] && hostInfo[item.Host.Name].buy_url">
                  <div class="name">{{ $t('buy-url') }}</div>
                  <div class="value">
                    <a style="color: #0077ff" :href="hostInfo[item.Host.Name].buy_url" target="_blank" @click.stop="() => {}">{{hostInfo[item.Host.Name].buy_url}}</a>
                  </div>
                </div>
              </div>
            </a-col>
            <a-col :span="14" :xs="24" :sm="24" :md="14" :lg="14" :sl="14">
              <a-row :gutter="20">
                <a-col :span="12" :xs="24" :sm="24" :md="12" :lg="12" :sl="12">
                  <CPU ref="cpuRef" style="margin-bottom: 20px;" :data="charts[item.Host.Name].cpu" />
                </a-col>
                <a-col :span="12" :xs="24" :sm="24" :md="12" :lg="12" :sl="12">
                  <Mem ref="memRef" :max="item.Host.MemTotal" style="margin-bottom: 20px;" :data="charts[item.Host.Name].mem" />
                </a-col>
                <a-col :span="12" :xs="24" :sm="24" :md="12" :lg="12" :sl="12">
                  <NetIn ref="netInRef" :data="charts[item.Host.Name].net_in" />
                </a-col>
                <a-col :span="12" :xs="24" :sm="24" :md="12" :lg="12" :sl="12">
                  <NetOut ref="netOutRef" :data="charts[item.Host.Name].net_out" />
                </a-col>
              </a-row>
            </a-col>
          </a-row>
        </div>
        <div class="delete-btn" @click.stop="handleShowDelete(item.Host.Name)">
          <icon-delete />
        </div>
      </div>
    </div>
    <a-modal v-model:visible="deleteVisible" :footer="false" :hide-title="true" width="360px">
      <div class="akile-modal-title">
        <span>{{$t('remove-host-title')}}</span>
        <a-button @click="handleClose">
          <template #icon>
            <icon-close/>
          </template>
        </a-button>
      </div>
      <div class="akile-modal-content">
        <a-input-password v-model="authSecret" :placeholder="$t('auth-placeholder')"></a-input-password>
        <div class="tips">{{$t('remove-host-tip')}}</div>
      </div>
      <div class="akile-modal-action">
        <a-button type="primary" status="danger" :long="true" @click="handleDeleteHost">{{$t('remove-host-btn')}}</a-button>
      </div>
    </a-modal>
    <a-modal v-model:visible="editVisible" :footer="false" :hide-title="true" width="360px">
      <div class="akile-modal-title">
        <span>{{$t('edit-host-title')}}</span>
        <a-button @click="handleEditClose">
          <template #icon>
            <icon-close/>
          </template>
        </a-button>
      </div>
      <div class="akile-modal-content">
        <a-date-picker v-model="duetime" :placeholder="$t('due-time-placeholder')" style="margin-bottom: 10px;width: 100%;"></a-date-picker>
        <a-input v-model="seller" :placeholder="$t('isp-placeholder')" style="margin-bottom: 10px;"></a-input>
        <a-input v-model="price" :placeholder="$t('price-placeholder')" style="margin-bottom: 10px;"></a-input>
        <a-input v-model="buy_url" :placeholder="$t('buy-url-placeholder')" style="margin-bottom: 10px;"></a-input>
        <a-input-password v-model="authSecret" :placeholder="$t('auth-placeholder')"></a-input-password>
      </div>
      <div class="akile-modal-action">
        <a-button type="primary" :long="true" @click="handleEditHost">{{$t('edit-host-btn')}}</a-button>
      </div>
    </a-modal>
    <div class="footer" style="margin-top: 30px">Acck Cloud 节点监控</div>
    <div class="footer" style="margin-bottom: 30px"><a href="https://github.com/akile-network/akile_monitor">Powered by Akile LTD.</a></div>
  </div>
</template>

<style lang="scss">
body {
  margin: 0;
  background-color: #fafafa;
  font-family: Inter,-apple-system,BlinkMacSystemFont,Roboto,PingFang SC,Noto Sans CJK,WenQuanYi Micro Hei,Microsoft YaHei;
}

a {
  text-decoration: none;
}

.max-container {
  margin: 0 auto;
  width: 100%;
  max-width: 1440px;
}

.header {
  margin: 10px;
  display: flex;
  align-items: center;
  justify-content: space-between;

  .theme-btn {
    border: 1px solid #eeeeee!important;
    background-color: #ffffff!important;
    color: #333333!important;
  }
}

.arco-dropdown {
  padding: 4px!important;
  border-radius: 8px!important;
  .arco-dropdown-option {
    border-radius: 4px !important;
    padding: 8px;
    line-height: 13px;
    font-size: 13px;
  }
}

.area-tabs {
  margin: 20px 10px;

  .area-tab-item {
    margin-bottom: 10px;
    margin-right: 10px;
    padding: 6px 12px;
    border-radius: 6px;
    cursor: pointer;
    border: 1px solid #e5e5e5;
    background: #ffffff;
    box-shadow: 0 2px 4px 0 rgba(133, 138, 180, 0.14);
    display: inline-block;

    .flag-icon {
      border-radius: 3px;
      margin-top: -3px;
    }

    &.is-active {
      background: #005fe710;
      color: #054db4;
      border: 1px solid #005fe7;
    }
  }

}

.monitor-card {
  position: relative;
  margin: 0 auto;
  padding: 10px;

  .monitor-item {
    position: relative;
    margin-bottom: 12px;
    padding: 12px 24px;
    border-radius: 6px;
    border: 1px solid #e5e5e5;
    display: block;
    background: #ffffff;
    box-shadow: 0 2px 4px 0 rgba(133, 138, 180, 0.14);
    cursor: pointer;

    &.is-active {
      background: #e7e7e730;

      .delete-btn,
      .edit-btn {
        display: none!important;
      }

      &>.detail {
        margin-top: 15px;
        border-top: 1px solid #eeeeee;
        padding-top: 15px;
        display: block;
      }
    }

    &:hover {
      background: #e7e7e730;

      .delete-btn,
      .edit-btn {
        display: flex;
      }
    }

    .edit-btn {
      right: 60px!important;
      background: rgba(22, 131, 255, 0.13)!important;

      &:hover {
        background: rgba(22, 131, 255, 0.19)!important;
      }

      .arco-icon {
        color: #1673ff!important;
      }
    }

    .delete-btn,
    .edit-btn {
      position: absolute;
      right: 20px;
      top: calc(50% - 16px);
      cursor: pointer;
      background: #ff161620;
      border-radius: 100px;
      width: 32px;
      height: 32px;
      display: none;
      align-items: center;
      justify-content: center;
      transition: .15s background-color ease-in-out;

      &:hover {
        background: #ff161630;
      }

      .arco-icon {
        color: #ff1616;
      }
    }

    .flag-icon {
      margin-right: 5px;
      border-radius: 3px;
    }

    .monitor-item-title {
      margin-bottom: 3px;
      font-size: 11px;
      opacity: .6;
    }

    .monitor-item-value {
      font-weight: 500;
    }

    .monitor-item-progress {
      margin-top: 4px;
      height: 4px;
      display: block;
    }

    .detail {
      width: 100%;
    }

    .name {
      display: inline-block;
      vertical-align: middle;
      width: 250px;

      .title {
        margin-bottom: 5px;
        display: flex;
        align-items: center;
        font-weight: 600;
        font-size: 16px;
      }

      .status {
        display: flex;
        align-items: center;
        &::before {
          margin-right: 10px;
          position: relative;
          display: block;
          content: '';
          width: 8px;
          height: 8px;
          border-radius: 12px;
          background-color: #fb2c36;
        }

        &.online {
          &::before {
            background-color: #00c951;
          }
        }

        span {
          font-size: 13px;
          opacity: .6;
        }
      }
    }

    .platform {
      display: inline-block;
      vertical-align: top;
      width: 120px;
    }

    .cpu {
      display: inline-block;
      vertical-align: top;
      width: 120px;
    }

    .mem {
      display: inline-block;
      vertical-align: top;
      width: 120px;
    }

    .average {
      display: inline-block;
      vertical-align: top;
      width: 200px;
    }

    .network {
      display: inline-block;
      vertical-align: top;
      width: 200px;
    }

    .detail {
      display: none;

      .detail-item-list {
        margin-bottom: 20px;
      }

      .detail-item {
        .name {
          width: 30%;
          font-size: 12px;
          color: #666;
          margin-bottom: 5px;
          display: inline-block;
          vertical-align: top;
        }

        .value {
          width: 70%;
          font-size: 12px;
          font-weight: 500;
          display: inline-block;
          vertical-align: top;
        }
      }
    }
  }
}

.logo {
  margin-top: 20px;
  margin-bottom: 20px;
  position: relative;
  cursor: pointer;
  line-height: 54px;
  height: 54px;
  font-size: 16px;
  font-weight: 900;
  color: #333;
  display: flex;
  align-items: center;

  .arco-icon {
    margin-right: 5px;
    height: 28px;
    width: 28px;
    color: rgb(22,93,255)!important;
  }
}

.monitor-action-bar {
  margin: 0 10px;
  display: inline-block;
  border: 1px solid #e5e5e5;
  background: #ffffff;
  box-shadow: 0 2px 4px 0 rgba(133, 138, 180, 0.14);
  border-radius: 4px;

  :deep(.arco-tabs-content) {
    display: none;
  }
}

.footer {
  line-height: 22px;
  text-align: center;
  color: #565656;

  a {
    color: rgba(var(--primary-6));
  }
}

.akile-modal-title {
  display: flex;
  align-items: center;
  justify-content: space-between;
  font-size: 16px;
  font-weight: 600;
  margin-bottom: 20px;
}

.akile-modal-content {
  margin-bottom: 20px;
  .tips {
    font-size: 12px;
    color: #333333;
    margin-top: 10px;
  }
}

body[arco-theme='dark'] {
  background-color: #111111;

  .arco-dropdown {
    background-color: #000000!important;
    border: 1px solid rgb(46 46 46)!important;
  }

  .arco-modal {
    background-color: #0e0e0e;
    border: 1px solid rgba(255,255,255,0.05);
  }

  .header {
    .logo {
      span,
      small {
        color: #ffffff;
      }
    }

    .theme-btn {
      border: 1px solid #333333!important;
      background-color: #000000!important;
      color: #ffffff!important;
    }
  }

  .area-tabs {
    .area-tab-item {
      border-color: #333333;
      background: #000000;
      color: #ffffff;
      box-shadow: none;

      &.is-active {
        background: #005fe705;
        color: #005fe7;
        border: 1px solid #005fe7;
      }
    }
  }

  .footer {
    color: #ffffffAA;
  }

  .monitor-card {
    .monitor-item {
      border: 1px solid rgb(255 255 255 / 16%);
      box-shadow: none;
      background-color: #000000;
      color: #ffffff;

      &:hover {
        background-color: #101010;
      }

      .detail {
        border-color: #333333AA;
        .detail-item-list {
          .detail-item {
            .name {
              color: #999999;
            }
          }
        }
      }
    }
  }

}

@media screen and (max-width: 768px) {
  .monitor-item {
    &>div {
      margin-bottom: 10px;
    }
  }

  .detail {
    .detail-item {
      .name {
        width: 25%!important;
      }
      .value {
        width: 75%!important;
      }
    }
  }
}

@media screen and (max-width: 1920px) {
  .max-container {
    max-width: 1440px;
  }
}

@media screen and (max-width: 1280px) {
  .max-container {
    max-width: 1140px;
  }
}

@media screen and (max-width: 992px) {
  .max-container {
    max-width: 960px;
  }
}

@media screen and (max-width: 768px) {
  .max-container {
    max-width: 720px;
  }
}

@media screen and (max-width: 576px) {
  .max-container {
    max-width: 540px;
  }
}
</style>
