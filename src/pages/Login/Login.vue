<template>
  <section class="loginContainer">
    <div class="loginInner">
      <div class="login_header">
        <h2 class="login_logo">HAN</h2>
        <div class="login_header_title">
          <a href="javascript:;" :class="{on: loginWay===1}" @click="loginWay = 1">短信登录</a>
          <a href="javascript:;" :class="{on: loginWay===2}" @click="loginWay = 2">密码登录</a>
        </div>
      </div>
      <div class="login_content">
        <form @submit.prevent="login">
          <div :class="{on: loginWay===1}">
            <section class="login_message">
              <input type="tel" maxlength="11" placeholder="手机号" v-model="phone">
              <button :disabled="!rightPhone" class="get_verification" :class="{right_phone: rightPhone}"
                      @click.prevent="getCode()">
                {{cdTime ? `已发送(${cdTime}s)` : '获取验证码'}}
              </button>
            </section>
            <section class="login_verification">
              <input type="text" maxlength="8" placeholder="验证码" v-model="code">
            </section>
            <section class="login_hint">
              温馨提示：未注册硅谷外卖帐号的手机号，登录时将自动注册，且代表已同意
              <a href="javascript:;">《用户服务协议》</a>
            </section>
          </div>
          <div :class="{on: loginWay===2}">
            <section>
              <section class="login_message">
                <input type="text" maxlength="11" placeholder="手机/邮箱/用户名" v-model="name">
              </section>
              <section class="login_verification">
                <input type="text" maxlength="8" placeholder="密码" v-model="pwd" v-if="showPwd">
                <input type="password" maxlength="8" placeholder="密码" v-model="pwd" v-else>
                <div class="switch_button" :class="showPwd ? 'on' : 'off'" @click="showPwd=!showPwd">
                  <div class="switch_circle" :class="{right: showPwd}"></div>
                  <span class="switch_text">{{showPwd ? 'abc' : '...'}}</span>
                </div>
              </section>
              <section class="login_message">
                <input type="text" maxlength="11" placeholder="验证码" v-model="captcha">
                <img class="get_verification" src="http://localhost:4000/captcha" alt="captcha" @click="getCaptcha"
                     ref="captcha">
              </section>
            </section>
          </div>
          <button class="login_submit">登录</button>
        </form>
        <a href="javascript:;" class="about_us">关于我们</a>
      </div>
      <span href="javascript:" class="go_back" @click="$router.back()">
        <i class="iconfont icon-jiantou2"></i>
      </span>
    </div>

    <AlertTip :alert-text="alertTip" v-if="showAlert" @closeTip="closeTip"></AlertTip>
  </section>
</template>

<script>
  import AlertTip from '../../components/AlertTip/AlertTip'
  import {reqSendCode, reqSmsLogin, reqPwdLogin, reqUserInfo} from '../../api/index'

  export default {
    components: {
      AlertTip
    },
    data () {
      return {
        loginWay: 2, // 1短信登录，2表示密码登录
        cdTime: 0, // 倒计时的时间
        showPwd: false, //是否显示密码
        phone: '', // 手机号
        code: '', // 短信验证码
        name: '', // 用户名
        pwd: '', // 密码
        captcha: '', // 图形验证码
        alertTip: '', // 提示文本
        showAlert: false, // 是否显示提示框
      }
    },
    computed: {
      // 判断手机号是否规范
      rightPhone () {
        return /^1\d{10}$/.test(this.phone)
      }
    },
    methods: {
      //定义公用的弹出框方法
      showAlertFun (alertTip) {
        this.showAlert = true
        this.alertTip = alertTip
      },
      //异步获取短信验证码
      async getCode () {
        //如果当前没有计时
        if (!this.cdTime) {
          //启动倒计时
          this.cdTime = 60
          this.intervalId = setInterval(() => {
            this.cdTime--
            if (this.cdTime <= 0) {
              //停止计时
              clearInterval(this.intervalId)
            }
          }, 1000)
        }
        //发送ajax请求
        const result = await reqSendCode(this.phone)
        if (result.code === 1) {
          //显示提示
          this.showAlertFun(result.msg)
          //停止倒计时
          if (this.cdTime) {
            this.cdTime = 0
            clearInterval(this.intervalId)
            // this.intervalId = undefined
          }
        }
      },
      //异步登录
      async login () {
        let result
        //前台表单验证
        if (this.loginWay === 1) { //短信登录
          const {rightPhone, phone, code} = this
          if (!rightPhone) {
            //手机号不正确
            this.showAlertFun('手机号不正确')
            return
          } else if (!/^\d{6}$/.test(code)) {
            //验证码必须为6位数字
            this.showAlertFun('验证码不正确')
            return
          }
          //ajax请求短信登录
          result = await reqSmsLogin(phone, code)

        }
        else if (this.loginWay === 2) { //　密码登录
          const {name, pwd, captcha} = this
          if (!name) {
            // 姓名必须指定
            this.showAlertFun('请输入用户名')
            return
          } else if (!pwd) {
            // 密码必须指定
            this.showAlertFun('请输入密码')
            return
          } else if (!captcha) {
            // 验证码必须指定
            this.showAlertFun('请输入验证码')
            return
          }
          //发送ajax请求密码登录
          result = await reqPwdLogin({name, pwd, captcha})
        }
        //停止倒计时
        if (this.cdTime) {
          this.cdTime = 0
          clearInterval(this.intervalId)
          // this.intervalId = undefined
        }
        //根据结果数据处理
        if (result.code === 0) {
          const user = result.data
          //将user保存到state中
          this.$store.dispatch('recordUserInfo', user)
          // 跳转路由，去个人中心
          this.$router.replace('/profile')
        } else {
          this.getCaptcha()
          // this.captcha = ''  // 清空输入框
          const msg = result.msg
          this.showAlertFun(msg)
        }
      },
      // 关闭提示框
      closeTip () {
        this.showAlert = false
        this.alertTip = ''
      },
      //获取一个新的图片验证码
      getCaptcha () {
        // 需要每次访问的路径不同
        this.$refs.captcha.src = 'http://localhost:4000/captcha?time' + Date.now()
      },
      //
    }
  }
</script>

<style lang="stylus" rel="stylesheet/stylus" scoped>
  @import "../../common/stylus/mixins.styl"
  .loginContainer
    height 100%
    background #f5f5f5

    .loginInner
      padding-top 60px
      width 80%
      margin 0 auto

      .login_header
        .login_logo
          font-size 40px
          font-weight bold
          color #FFC0CB
          text-align center

        .login_header_title
          padding-top 40px
          text-align center

          > a
            color #333
            font-size 14px
            padding-bottom 4px

            &:first-child
              margin-right 40px

            &.on
              color #FFC0CB
              font-weight 700
              border-bottom 2px solid #FFC0CB

      .login_content
        > form
          > div
            display none

            &.on
              display block

            input
              width 100%
              height 100%
              padding-left 10px
              box-sizing border-box
              border 1px solid #ddd
              border-radius 4px
              outline 0
              font 400 14px Arial

              &:focus
                border 1px solid #02a774

            .login_message
              position relative
              margin-top 16px
              height 48px
              font-size 14px
              background #fff

              .get_verification
                position absolute
                top 50%
                right 10px
                transform translateY(-50%)
                border 0
                color #ccc
                font-size 14px
                background transparent

                &.right_phone
                  color black

            .login_verification
              position relative
              margin-top 16px
              height 48px
              font-size 14px
              background #fff

              .switch_button
                font-size 12px
                border 1px solid #ddd
                border-radius 8px
                transition background-color .3s, border-color .3s
                padding 0 6px
                width 30px
                height 16px
                line-height 16px
                color #fff
                position absolute
                top 50%
                right 10px
                transform translateY(-50%)

                &.off
                  background #fff

                  .switch_text
                    float right
                    color #ddd

                &.on
                  background #02a774

                > .switch_circle
                  &.right
                    transform translateX(27px)
                  position absolute
                  top -1px
                  left -1px
                  width 16px
                  height 16px
                  border 1px solid #ddd
                  border-radius 50%
                  background #fff
                  box-shadow 0 2px 4px 0 rgba(0, 0, 0, .1)
                  transition transform .3s

            .login_hint
              margin-top 12px
              color #999
              font-size 14px
              line-height 20px

              > a
                color #87CEFA

          .login_submit
            display block
            width 100%
            height 42px
            margin-top 30px
            border-radius 4px
            background #FFC0CB
            color #fff
            text-align center
            font-size 16px
            line-height 42px
            border 0

        .about_us
          display block
          font-size 12px
          margin-top 20px
          text-align center
          color #999

      .go_back
        position absolute
        top 5px
        left 5px
        width 30px
        height 30px

        > .iconfont
          font-size 20px
          color #999
</style>
