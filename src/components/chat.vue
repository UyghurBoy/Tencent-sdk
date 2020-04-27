<template>
  <div>
聊天
<button @click='login'>登录</button>
<button @click='logOut'>退出</button>
<input type="text" @input='selectUserId' placeholder="你的userid">
<input type="text" @input='toUserId' placeholder="别人的userid">
<button @click='createTextMessage()'>发送消息</button>
  </div>
</template>

<script>
import TIM from 'tim-js-sdk'
import COS from 'cos-js-sdk-v5'
import {TEST_ACCOUNT} from '../const/account_dev'
import LibGenerateTestUserSig from '../const/lib-generate-test-usersig.min.js'
export default {
  name: 'chat',
  components: {
  },
  data() {
    return {
      sdkAppId: TEST_ACCOUNT.sdkAppId,
      userId: TEST_ACCOUNT.userId + (Math.random() * 100).toFixed(),
      userToken: TEST_ACCOUNT.userToken,
      tim: ''
    }
  },
  created() {
  },
  mounted() {
    // this.createChat();
  },
  methods: {
    selectUserId(e) {
      console.log(e)
      this.userId = e.data
      this.createChat()
    },
    toUserId(e) {
      this.createTextMessage(e.data)
    },
    createChat() {
      this.tim = TIM.create({SDKAppID: this.sdkAppId})
      // 设置 SDK 日志输出级别，详细分级请参见 setLogLevel 接口的说明
      this.tim.setLogLevel(0); // 普通级别，日志量较多，接入时建议使用
      // tim.setLogLevel(1); // release 级别，SDK 输出关键信息，生产环境时建议使用

      // 注册 COS SDK 插件
      this.tim.registerPlugin({'cos-js-sdk': COS});

      this.listenEvent()
    },

    login() {
      //获取签名
      const config = this.genTestUserSig()
      const sdkAppId = config.sdkAppId
      const userSig = config.userSig
      // 开始登录
      this.tim.login({userID: this.userId, userSig: userSig}).then(function(imResponse) {
        console.log(imResponse.data); // 登录成功
        if (imResponse.data.repeatLogin === true) {
          // 标识账号已登录，本次登录操作为重复登录。v2.5.1 起支持
          console.log(imResponse.data.errorInfo);
        }
      }).catch(function(imError) {
        console.warn('login error:', imError); // 登录失败的相关信息
      });
    },

    listenEvent() {
      // 监听事件，如：
      this.tim.on(TIM.EVENT.SDK_READY, (event) => {
        // 收到离线消息和会话列表同步完毕通知，接入侧可以调用 sendMessage 等需要鉴权的接口
        // event.name - TIM.EVENT.SDK_READY
        this.onMessageReceived(event)
      });

      this.tim.on(TIM.EVENT.MESSAGE_RECEIVED, (event) => {
        // 收到推送的单聊、群聊、群提示、群系统通知的新消息，可通过遍历 event.data 获取消息列表数据并渲染到页面
        // event.name - TIM.EVENT.MESSAGE_RECEIVED
        // event.data - 存储 Message 对象的数组 - [Message]
        this.onMessageReceived(event)
      });

      this.tim.on(TIM.EVENT.MESSAGE_REVOKED, (event) => {
        // 收到消息被撤回的通知。使用前需要将SDK版本升级至v2.4.0或以上。
        // event.name - TIM.EVENT.MESSAGE_REVOKED
        // event.data - 存储 Message 对象的数组 - [Message] - 每个 Message 对象的 isRevoked 属性值为 true
        this.onMessageReceived(event)
      });

      this.tim.on(TIM.EVENT.CONVERSATION_LIST_UPDATED, (event) => {
        // 收到会话列表更新通知，可通过遍历 event.data 获取会话列表数据并渲染到页面
        // event.name - TIM.EVENT.CONVERSATION_LIST_UPDATED
        // event.data - 存储 Conversation 对象的数组 - [Conversation]
        this.onMessageReceived(event)
      });

      this.tim.on(TIM.EVENT.GROUP_LIST_UPDATED, (event) => {
        // 收到群组列表更新通知，可通过遍历 event.data 获取群组列表数据并渲染到页面
        // event.name - TIM.EVENT.GROUP_LIST_UPDATED
        // event.data - 存储 Group 对象的数组 - [Group]
        this.onMessageReceived(event)
      });

      this.tim.on(TIM.EVENT.PROFILE_UPDATED, (event) => {
        // 收到自己或好友的资料变更通知
        // event.name - TIM.EVENT.PROFILE_UPDATED
        // event.data - 存储 Profile 对象的数组 - [Profile]
        this.onMessageReceived(event)
      });

      this.tim.on(TIM.EVENT.BLACKLIST_UPDATED, (event) => {
        // 收到黑名单列表更新通知
        // event.name - TIM.EVENT.BLACKLIST_UPDATED
        // event.data - 存储 userID 的数组 - [userID]
        this.onMessageReceived(event)
      });

      this.tim.on(TIM.EVENT.ERROR, (event) => {
        // 收到 SDK 发生错误通知，可以获取错误码和错误信息
        // event.name - TIM.EVENT.ERROR
        // event.data.code - 错误码
        // event.data.message - 错误信息
        this.onMessageReceived(event)
      });

      this.tim.on(TIM.EVENT.SDK_NOT_READY, (event) => {
        // 收到 SDK 进入 not ready 状态通知，此时 SDK 无法正常工作
        // event.name - TIM.EVENT.SDK_NOT_READY
        this.onMessageReceived(event)
      });

      this.tim.on(TIM.EVENT.KICKED_OUT, (event) => {
        // 收到被踢下线通知
        // event.name - TIM.EVENT.KICKED_OUT
        // event.data.type - 被踢下线的原因，例如 :
        //   - TIM.TYPES.KICKED_OUT_MULT_ACCOUNT 多实例登录被踢
        //   - TIM.TYPES.KICKED_OUT_MULT_DEVICE 多终端登录被踢
        //   - TIM.TYPES.KICKED_OUT_USERSIG_EXPIRED 签名过期被踢（v2.4.0起支持）。
        this.onMessageReceived(event)
      });

      this.tim.on(TIM.EVENT.NET_STATE_CHANGE, (event) => {
        // 网络状态发生改变（v2.5.0 起支持）。
        // event.name - TIM.EVENT.NET_STATE_CHANGE
        // event.data.state 当前网络状态，枚举值及说明如下：
        //   - TIM.TYPES.NET_STATE_CONNECTED - 已接入网络
        //   - TIM.TYPES.NET_STATE_CONNECTING - 连接中。很可能遇到网络抖动，SDK 在重试。接入侧可根据此状态提示“当前网络不稳定”或“连接中”
        //   - TIM.TYPES.NET_STATE_DISCONNECTED - 未接入网络。接入侧可根据此状态提示“当前网络不可用”。SDK 仍会继续重试，若用户网络恢复，SDK 会自动同步消息
        this.onMessageReceived(event)
      });
    },

    createTextMessage(userId) {
      // 发送文本消息，Web 端与小程序端相同
      // 1. 创建消息实例，接口返回的实例可以上屏
      let message = this.tim.createTextMessage({
        to: userId,
        conversationType: TIM.TYPES.CONV_C2C,
        // 消息优先级，用于群聊（v2.4.2起支持）。如果某个群的消息超过了频率限制，后台会优先下发高优先级的消息，详细请参考：https://cloud.tencent.com/document/product/269/3663#.E6.B6.88.E6.81.AF.E4.BC.98.E5.85.88.E7.BA.A7.E4.B8.8E.E9.A2.91.E7.8E.87.E6.8E.A7.E5.88.B6)
        // 支持的枚举值：TIM.TYPES.MSG_PRIORITY_HIGH, TIM.TYPES.MSG_PRIORITY_NORMAL（默认）, TIM.TYPES.MSG_PRIORITY_LOW, TIM.TYPES.MSG_PRIORITY_LOWEST
        // priority: TIM.TYPES.MSG_PRIORITY_NORMAL,
        payload: {
          text: 'Hello world!'
        }
      });
      // 2. 发送消息
      let promise = this.tim.sendMessage(message);
      promise.then(function(imResponse) {
        // 发送成功
        console.log('发送消息成功: ',imResponse);
      }).catch(function(imError) {
        // 发送失败
        console.warn('发送消息失败:', imError);
      });
    },

    onMessageReceived(event) {
      console.log("来消息啦",event)
    },

    logOut() {
      this.tim.logout()
      .then(function(imResponse) {
        console.log('登出成功',imResponse.data); // 登出成功
      }).catch(function(imError) {
        console.warn('登出错误', imError);
      });
    },

    genTestUserSig() {
      const USERID = this.userId
      const SDKAPPID = this.sdkAppId;
      const EXPIRETIME = 604800;
      const SECRETKEY = this.userToken;
      if (SDKAPPID === '' || SECRETKEY === '') {
        alert(
          '请先配置好您的账号信息： SDKAPPID 及 SECRETKEY ' +
            '\r\n\r\nPlease configure your SDKAPPID/SECRETKEY in js/debug/GenerateTestUserSig.js'
        );
      }
      const generator = new LibGenerateTestUserSig(SDKAPPID, SECRETKEY, EXPIRETIME);
      const userSig = generator.genTestUserSig(USERID);
      return {
        sdkAppId: SDKAPPID,
        userSig: userSig
      };
    }
  }

}
</script>

<style lang='less'>

</style>
