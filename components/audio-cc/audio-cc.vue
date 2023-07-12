<template>
  <view class="audio-cc">
    <view class="info-box">
      <view class="title">{{ title }}</view>
      <view v-if="show" class="time">{{ time }}</view>
    </view>
    <img class="play-btn" @click="play=!play" mode="widthFix" :src="currentImg">

    <slider class="audio-slider" activeColor="#FFCE00" backgroundColor="#E8E8E8" block-color="#FFCE00" block-size="12" v-if="show" :max="sliderMax" :value="sliderCurrent" @change="sliderChange" @changing="sliderChanging"/>
  </view>
</template>
<script>
//GEP默认音频播放组件模块
const audioContainer = {};
export default {
  name:"audio-cc",
  // 此处定义传入的数据
  props: {
    //音频地址
    src: {
      type: String,
      value: ''
    },
    //标题
    title: {
      type: String,
      value: ''
    },
    //是否自动播放
    autoplay: {
      type: Boolean,
      value: false
    },
    //是否循环播放
    loop: {
      type: Boolean,
      value: false
    },
    playImg:{
      type: String,
      value: '../../static/audio-play.png'
    },
    pauseImg:{
      type: String,
      value: '../../static/audio-stop.png'
    }
  },
  data() {
    return {
      uuid: 0,
      max: 0,
      current: 0,
      play: false,
      stop: false,
      innerAudioContext: uni.createInnerAudioContext(),
      accuracy: 3000
    };
  },
  //在实例创建完成后被立即调用。
  mounted() {
    this.createUuid();
    this.audioInit();
  },
  //实例销毁之前调用。在这一步，实例仍然完全可用。
  beforeDestroy(){
    console.info('实例销毁;播放状态切换：停止');
    this.innerAudioContext.stop();
    this.innerAudioContext.destroy();
    audioContainer[this.uuid] = null;
  },
  watch:{
    play(val){
      console.info(`play: ${val}`);
      if (this.src.length > 0) {
        if (val) {
          console.info(`uuid：${this.uuid};播放状态切换：播放`);
          this.innerAudioContext.play();
        } else {
          console.info(`uuid：${this.uuid};播放状态切换：暂停`);
          this.innerAudioContext.pause();
        }
      }
    },
    stop(val){
      if (val){
        console.info(`uuid：${this.uuid};播放状态切换：停止`);
        this.current = 0;
        this.innerAudioContext.stop();
        this.play = false;
        this.stop = false;
      }
    },
    src(newVal, oldVal){
      if (newVal !== oldVal) {
        this.audioInit();
      }
    }
  },
  computed:{
    currentImg(){
      if (this.play) {
        return this.pauseImg;
      } else {
		return this.playImg;
      }
    },
    time(){
      if (this.max > 0) {
        let end = this.max - this.current,
            minute = parseInt(end / 60),
            second = parseInt(end % 60);

        if (minute < 10) {
          minute = `0${minute}`;
        }

        if (second < 10) {
          second = `0${second}`;
        }

        return `${minute}:${second}`;
      } else {
        return '00:00';
      }
    },
    show(){
      return this.max > 0;
    },
    sliderMax(){
      return this.max > 0 ? parseInt(this.max * this.accuracy) : 0;
    },
    sliderCurrent(){
      return this.current > 0 ? parseInt(this.current * this.accuracy) : 0;
    }
  },
  methods:{
    //音频初始化
    audioInit() {
      let _self = this,
          //是否单独播放
          alone = true;

      if (this.src.length > 0) {
        let time = parseInt(Date.parse(new Date()) / (1000 * 60 * 30)),
            src = `${this.src}?v=${time}`;

        console.info(`uuid：${this.uuid}`);
        console.info(`加载音频地址：${src}`);

        _self.innerAudioContext.src = src;
        _self.innerAudioContext.autoplay = _self.autoplay;
        _self.innerAudioContext.loop = _self.loop;

        audioContainer[_self.uuid] = _self.innerAudioContext;
      }

      //初始化加载事件
      this.innerAudioContext.onCanplay(() => {
        console.info('初始化加载事件');
        _self.innerAudioContext.duration;
        setTimeout(function (){
          _self.max = _self.innerAudioContext.duration;
          console.info(`max: ${_self.max};src: ${_self.innerAudioContext.src}`);
        }, 500)
      });

      //开始播放事件
      this.innerAudioContext.onPlay(() => {
        console.info('开始播放', `是否单独播放${alone}`);
        if (alone) {
          for (let uuid in audioContainer){
            if (this.uuid !== uuid) {
              audioContainer[uuid] && audioContainer[uuid].pause();
            }
          }
        }
      });

      //音频播放进度更新事件
      this.innerAudioContext.onTimeUpdate(() => {
        console.info(`当前播放进度: ${_self.innerAudioContext.currentTime}`);
        if (this.max <= 0) {
          setTimeout(function (){
            _self.max = _self.innerAudioContext.duration;
            console.info(`max: ${_self.max};src: ${_self.innerAudioContext.src}`);
          }, 200)
        } else {
          _self.current = _self.innerAudioContext.currentTime;
        }
      });

      //音频暂停事件
      this.innerAudioContext.onPause((e) => {
        console.info(`播放暂停`, e);
        if (_self.play) {
          _self.pauseAudio();
        }
      });

      //音频停止事件
      this.innerAudioContext.onStop((e) => {
        console.info(`播放停止`, e);

      });

      //音频自然播放结束事件
      this.innerAudioContext.onEnded((e) => {
        console.info(`播放结束`, e);
        _self.stopAudio();
      });

      //异常监听
      this.innerAudioContext.onError((res) => {
        uni.showToast({
          title: res.errMsg || "网络故障请稍后重试"
        });
        console.info(res.errMsg, res.errCode);
      });
    },
    //播放
    playAudio(){
      this.play = true;
    },
    //暂停
    pauseAudio(){
      this.play = false;
    },
    //停止
    stopAudio(){
      this.stop = true;
    },
    //快进到指定位置
    seek(index){
      this.innerAudioContext.seek(index / this.accuracy);
    },
    //快进到指定位置,并播放
    seekAndPlay(index){
      setTimeout(() =>{
        this.seek(index);
        this.playAudio();
      }, 200);
    },
    //进度改变
    sliderChange(e) {
      this.pauseAudio();
      this.seekAndPlay(e.detail.value);
    },
    //进度连续改变
    sliderChanging(e){
      this.pauseAudio();
    },
    //创建唯一uuid
    createUuid() {
      let s = [], hexDigits = '0123456789abcdef', i;

      for (i = 0; i < 36; i++) {
        s[i] = hexDigits.substr(Math.floor(Math.random() * 0x10), 1);
      }

      s[14] = '4'; // bits 12-15 of the time_hi_and_version field to 0010
      s[19] = hexDigits.substr((s[19] & 0x3) | 0x8, 1); // bits 6-7 of the clock_seq_hi_and_reserved to 01
      s[8] = s[13] = s[18] = s[23] = '-';

      this.uuid = s.join('');
    }
  }
}
</script>

<style lang="scss">
.audio-cc{
  width: 630rpx;
  height: 130rpx;
  background: #F9F9F9;
  border-radius: 16rpx;
  position: relative;

  .info-box{
    width: 100%;
    padding: 20rpx 30rpx;

    .title{
      height: 40rpx;
      font-size: 28rpx;
      font-family: PingFangSC-Regular, PingFang SC;
      font-weight: 400;
      color: #262626;
      line-height: 40rpx;
    }

    .time{
      height: 34rpx;
      font-size: 24rpx;
      font-family: PingFang-SC-Light, PingFang-SC;
      font-weight: 300;
      color: #262626;
      line-height: 34rpx;
    }
  }

  .play-btn{
    position: absolute;
    width: 80rpx;
    height: 80rpx;
    right: 30rpx;
    top: 20rpx;
  }

  .audio-slider{
    position: absolute;
    width: 555rpx;
    left: 0;
    right: 0;
    bottom: -18rpx;
  }
}
</style>
