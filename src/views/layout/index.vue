<template>
  <div
    class="app-wrapper"
    :class="className"
  >
    <transition name="el-fade-in-linear">
      <Sidebar
        v-show="!fullScreen"
        class="sidebar-container"
      />
    </transition>
    <div
      class="main-container"
      :style="fullScreen ? 'margin-left: 0' : ''"
    >
      <transition name="el-fade-in-linear">
        <div style="position: relative; z-index: 99">
          <navbar v-show="!fullScreen" @handleFullScreen="handleFullScreen" />
          <tags-view v-show="!fullScreen" />
        </div>
      </transition>
      <app-main />
    </div>
  </div>
</template>

<script>
import mqtt from 'mqtt'
import { mapGetters } from 'vuex'
import { AppMain, Sidebar, Navbar, TagsView } from './components'

export default {
  name: 'Layout',
  components: {
    AppMain,
    Sidebar,
    Navbar,
    TagsView
  },
  data () {
    return {
      tip: null // 弹窗
    }
  },
  computed: {
    className () {
      return {
        hideSidebar: !this.sidebar.opened,
        openSidebar: this.sidebar.opened
      }
    },
    ...mapGetters([
      'sidebar',
      'fullScreen',
      'userInfo'
    ])
  },
  mounted () {
    this.initMqtt()
    // 添加全局的键盘事件
    document.body.addEventListener('keyup', this.handleFullScreen)
  },
  methods: {
    // 全屏显示
    handleFullScreen (event, start) {
      const e = event || window.event
      // 按下 shift + F11 组合 全屏显示
      if (e.keyCode === 122 && e.shiftKey || start) {
        // this.fullScreen = !this.fullScreen
        this.$store.commit('app/TOGGLE_FULLSCREEN', !this.fullScreen)
        if (this.fullScreen) {
          this.tip = this.$message({
            message: '按 Esc 或 shift + F11组合键 退出全屏模式',
            center: true
          })
        } else {
          this.tip.close()
        }
      }
      if (this.fullScreen && e.keyCode === 27) {
        this.$store.commit('app/TOGGLE_FULLSCREEN', !this.fullScreen)
      }
    },
    initMqtt () {
      const URL = process.env.VUE_APP_TYPE === 'localhost' ? '127.0.0.1:1212' : 'www.lyh.red/mqttwss'
      // 在不同协议下，使用不同的连接方式
      const protocol = location.protocol === 'http:' ? 'mqtt:' : 'mqtts:'
      const client = mqtt.connect(`${protocol}//${URL}`)
      const TopicList = [
        `/chat/user/${this.userInfo.id}`,
        `/chat/group/#`,
        `/message/user/${this.userInfo.id}`,
        `/message/audit/#`
      ]
      // 连接
      client.on('connect', () => {
        console.log('连接' + new Date())
        TopicList.forEach(topic => {
          client.subscribe(topic, function (err) {
            if (!err) {
              console.log('订阅成功: ' + topic)
            }
          })
        })
      })
      // 获取到消息
      client.on('message', (topic, message) => {
        // message is Buffer
        console.log(topic, message.toString())
      })
      // 断开自动重连
      client.on('close', () => {
        console.log('close重新连接' + new Date())
      })
    }
  }
}
</script>

<style scoped lang="scss">
  .app-wrapper{
    height: 100%;
    .sidebar-container{
      position: fixed;
      top: 0;
      bottom: 0;
      left: 0;
      width: 220px;
      // transition: width 10s;
      overflow-y: auto;
      &::-webkit-scrollbar {
        display: none;
      }
    }
    .main-container{
      transition: all .3s linear;
    }
  }
  .hideSidebar{
    .main-container{
      margin-left: 60px;
    }
  }
  .openSidebar{
    .main-container{
      margin-left: 220px;
    }
  }
</style>
