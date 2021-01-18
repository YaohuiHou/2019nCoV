<template>
  <view class="content">
    <map
      :latitude="latitude"
      :longitude="longitude"
      scale="10"
      :markers="marker"
      enable-poi
      show-compass
      include-points
      show-location
      @markertap="markertap"
    >
    <!-- 位置定位 -->
    <cover-view class="search">
      <button class="city-name" @click="chooseCity"><cover-view>{{cityName}}</cover-view></button>
      <button :class="['all', locus == 0 ? 'selected' : '']" @click="locus = 0">全部轨迹点</button>
      <button :class="['day7', locus == 1 ? 'selected' : '']" @click="locus = 1">7天轨迹点</button>
      <button :class="['day14', locus == 2 ? 'selected' : '']" @click="locus = 2">14天轨迹点</button>
    </cover-view>
    
      <!-- 详情 -->
      <cover-view class="cover-info" v-if="showInfo">
        <!-- 切换 -->
        <cover-view class="tabs">
          <button :class="newsShow ? '' : 'selected'" @click="newsShow = false">患者轨迹</button>
          <button :class="newsShow ? 'selected' : ''" @click="newsShow = true">本市概况</button>
        </cover-view>
        <!-- 关闭 -->
        <button class="close" @click="showInfo = !showInfo"></button>
        <cover-view class="content" v-if="!newsShow">
          <!-- 地点 -->
          <cover-view class="header" v-if="riskInfo">
            <cover-view class="h1">{{riskInfo.poi_name}} <cover-view class="mark">{{riskType}}</cover-view></cover-view>
            <cover-view class="h3">{{riskInfo.city}}{{riskInfo.area_name}}{{riskInfo.township}}</cover-view>
          </cover-view>
          <!-- 病患轨迹 -->
          <cover-view class="ul" v-if="riskInfo">
            <cover-view class="title" v-if="riskInfo.patient_list.length > 0"><cover-view >共有</cover-view><cover-view class="manlen">{{riskInfo.patient_list.length}}</cover-view><cover-view>人确诊</cover-view></cover-view>
            <cover-view class="title-no" v-if="riskInfo.patient_list.length <= 0">暂未公布</cover-view>
            <cover-view class="li" v-for="(item, index) in riskInfo.patient_list" :key="item.patient_id">
              <cover-view class="man-info">
              {{index+1}}. 患者： {{item.base_info}} {{sortTime(item.sort_time)}}确诊
              </cover-view>
              <cover-view class="p">{{item.extra_info}}</cover-view>
              <cover-view class="track-list">
                <cover-view class="track" v-for="(track,i) in item.track_list" :key="`track${index}${i}`">
                  <cover-view class="timer">{{track.poi_time_text}}</cover-view><cover-view>{{track.poi_text}}</cover-view>
                </cover-view>
              </cover-view>
            </cover-view>
          </cover-view>
          <cover-view class="nothing" v-if="!riskInfo">暂无轨迹公布~</cover-view>
        </cover-view>
        <!-- 全市疫情概况 -->
        <cover-view class="content" v-if="newsShow">
          <cover-view class="h1 h1-title">全市疫情概况</cover-view>
          <cover-view class="hine">非实时更新，根据国家卫健委每日通报更新</cover-view>
          <cover-view class="introduction">
            <cover-view class="zone">{{sortTime(introductions.updateTime)}} {{introductions.provinceName}}{{introductions.cityName}}</cover-view>
            <cover-view class="treat">现有确诊：{{introductions.series[0].treatingNum}} 治愈人数：{{introductions.series[0].curesNum}} 死亡人数：{{introductions.series[0].deathsNum}} 累计确诊：{{introductions.series[0].confirmedNum}} </cover-view>
          </cover-view>
          <cover-view v-for="(item,i) in trending" :key="i" class="confirmed-view">
            <cover-view class="city-name">{{item.name}}</cover-view>
            <cover-view class="new-num">新增：{{(item.confirmed_incr < 0) ? confirmIdincr[item.confirmed_incr] : item.confirmed_incr}}</cover-view>
            <cover-view class="confirmed">累计：{{item.confirmed_count}}</cover-view>
          </cover-view>
        </cover-view>
      </cover-view>
    </map>
  </view>
</template>

<script>
const QQMapWX = require('../../static/qqmap-wx-jssdk.js');
var qqmapsdk;
export default {
  data() {
    return {
      latitude: '',
      longitude: '',
      cityName: '北京市',
      cityCode: null,
      riskName: '全部轨迹',
      marker:[],
      cityMarkers:{},
      riskInfo: {},
      riskType: '',
      showInfo: false,
      newsShow: false,
      locus: 0,
      introductions: [],
      trending: [],
      confirmIdincr:{
        '-123456': '未公布',
        '-1': 0
      }
    };
  },
  watch:{
    locus(l){
      switch (l) {
        case 0:
          this.marker = this.cityMarkers[this.cityCode].allMarker
          break;
        case 1:
          this.marker = this.cityMarkers[this.cityCode].day7
          break;
        case 2:
          this.marker = this.cityMarkers[this.cityCode].day14
          break;
      }
    }
  },
  onLoad() {
    // console.log(QQMapWX);
    qqmapsdk = new QQMapWX({
        key: 'NERBZ-3PPKW-5CKRY-OWRKI-Q4VCJ-UXBZT'
    });
    this.getLocal()
    wx.showShareMenu({
      withShareTicket: true,
      menus: ['shareAppMessage', 'shareTimeline']
    })
  },
  methods: {
    getLocal() {
      uni.getLocation({
        type: 'gcj02',
        success:  (res)=> {
          console.log(res);
          this.latitude = res.latitude
          this.longitude = res.longitude
          this.search()
        },
      });
    },
    search() {
      qqmapsdk.reverseGeocoder({
        location: `${this.latitude},${this.longitude}`,
        success: (res)=> {
          const data = res.result.ad_info
          this.cityCode = data.city_code.replace(data.nation_code,'')
          this.cityName = data.city
          this.getCityInfoFun()
          this.getNews()
        },
        fail: (res)=> {
          console.log(res);
        }
      });
    },
    // 选择其他城市
    chooseCity(){
      this.newsShow = false
      this.showInfo = false
      uni.chooseLocation({
        success: (res)=> {
          this.latitude = res.latitude
          this.longitude = res.longitude
          this.search()
        }
      });
    },
    add0(m){return m<10?'0'+m:m },
    // 确诊时间
    sortTime(time){
      const date = new Date(time*1000);
      const y = date.getFullYear();
      const m = date.getMonth()+1;
      const d = date.getDate();
      return `${y}年${this.add0(m)}月${this.add0(d)}`
    },
    // 获取地图全部数据
    // track_type 轨道等级 2为路过  1为确诊地点
    // risk_level 风险等级 3为高风险  2位中风险  1为低风险   0为无风险地
    getCityInfoFun(){
      // 已查询过
      if(this.cityMarkers[this.cityCode] && this.cityMarkers[this.cityCode].allMarker){
        this.marker = this.cityMarkers[this.cityCode].allMarker
        return
      }
      uni.showLoading({
          title: '加载中'
      });
      uni.request({
          url: 'https://i.snssdk.com/toutiao/normandy/pneumonia_trending/track_list/',
          data:{
            city_code: this.cityCode,
            city_name: this.cityName,
            activeWidget: 15,
            show_poi_list: 1
          },
          success: (res)=> {
            uni.hideLoading();
            if(res.data.data.list.length <= 0){
              uni.showToast({
                  title: '暂时未公布详细轨迹~',
                  duration: 3000,
                  icon: 'none'
              });
              this.riskInfo = null
              this.showInfo = true
              this.newsShow = true
              return
            }
            let riskArr = [],day14 = [], day7 = [];
            const time = parseInt(new Date().getTime()/1000)
            // 风险地点
            for (const riskObj of res.data.data.list) {
              const o = this.pushMarker(riskObj,riskArr)
              const poiDay = Math.floor((time - riskObj.create_ts)/24/60/60)
              if(poiDay <= 7){
                day7.push(o)
              }
              if(poiDay <= 14){
                day14.push(o)
              }

              riskArr.push(o)
            }
            this.marker = [...riskArr]
            // 存入全部数据
            if(!this.cityMarkers[this.cityCode]){
              this.cityMarkers[this.cityCode] = {}
            }
            this.cityMarkers[this.cityCode].allMarker = this.marker
            this.cityMarkers[this.cityCode].riskArr = res.data.data.list
            this.cityMarkers[this.cityCode].day14 = day14
            this.cityMarkers[this.cityCode].day7 = day7
          }
      });
    },
    // 疫情最新消息
    getNews(){
      // 地区疫情数
      this.getCityNum()
      uni.request({
          url: `https://i.snssdk.com/forum/ncov_data/`,
          data:{
            city_code: [this.cityCode],
            data_type: [1],
            src_type: 'local'
          },
          success: (res)=> {
            // introduction
            // this.newsShow = true
            this.introductions = JSON.parse(res.data.ncov_city_data[this.cityCode])
          }
      })
    },
    // 地区详细确诊数
    getCityNum(){
      uni.request({
          url: `https://i.snssdk.com/toutiao/normandy/pneumonia_trending/district_stat/?local_id=${this.cityCode}`,
          success: (res)=> {
            this.trending = res.data.data.list
          }
      })
    },
    // 点击气泡
    markertap(e){
      const poi = this.cityMarkers[this.cityCode].riskArr[e.detail.markerId].poi
      // const poi = '116.497662,40.140719'
      uni.request({
          url: 'https://i.snssdk.com/toutiao/normandy/pneumonia_trending/poi/',
          data:{
            city_code: this.cityCode,
            city_name: this.cityName,
            search_current_poi: poi,
            poi: poi,
            activeWidget: 15,
            show_poi_list: 1
          },
          success: (res)=> {
            console.log(res.data.data.data);
            // track_type 轨道等级 2为路过  1为确诊地点
            // risk_level 风险等级 3为高风险  2位中风险  1为低风险   0为无风险地
            this.riskInfo = res.data.data.data
            this.showInfo = true
            let str = ''
            // 轨迹等级
            switch (this.riskInfo.track_type) {
              // 路过
              case 2:
                str = '轨迹点'
                break;
              // 确诊地
              case 1:
                str = '确诊地点'
                break;
              default:
                str = '轨迹点'
                break;
            }
            // 风险等级
            switch (this.riskInfo.risk_level) {
              case 0:
                break;
              case 1:
                str = '低风险'
                break;
              case 2:
                str = '中风险'
                break;
              case 3:
                str = '高风险'
                break;
              default:
                str = '轨迹点'
                break;
            }
            this.riskType = str
          }
      })
    },
    // 插入标记数据
    pushMarker(item, riskArr){
      // 插入气泡
      const styles = {
        0: {
          color: '#f6761c'
        },
        1: {
          color: '#fa4630'
        },
        2: {
          color: '#fa4630'
        },
        3: {
          color: '#4f1202'
        },
        4: {
          color: '#4f1202'
        },
      }
      let o = {
        content: '',
        color: '#fff',
        fontSize: 16,
        padding: 10,
        textAlign: 'center',
        borderRadius: 20,
        bgColor: '',
        anchorX: '',
        anchorY: ''
      }
      let locals = item.poi.split(',');
      let iconImg;
      // 轨迹等级
      switch (item.track_type) {
        // 路过
        case 2:
          iconImg = '../../static/imgs/pug.png'
          break;
        // 确诊地
        case 1:
          iconImg = '../../static/imgs/virus1.png'
          break;
        default:
          iconImg = '../../static/imgs/map.png'
          break;
      }
      // 风险等级
      switch (item.risk_level) {
        case 0:
          break;
        case 1:
          iconImg = '../../static/imgs/virus1.png'
          break;
        case 2:
          iconImg = '../../static/imgs/virus2.png'
          break;
        case 3:
          iconImg = '../../static/imgs/virus3.png'
          break;
        default:
          iconImg = '../../static/imgs/map.png'
          break;
      }
      o.content = item.poi_name
      o.bgColor = styles[item.risk_level].color
      o.anchorX = 0
      o.anchorY = 0
      return {
        id: riskArr.length,
        width: 36,
        height: 36,
        latitude: locals[1],
        longitude: locals[0],
        iconPath: iconImg,
        callout: o
      }
    }
  },
};
</script>

<style lang="scss">
.content{
  position: relative;
  width: 100%;
  height: 100vh;
  overflow: hidden;
  map{
    position: relative;
    width: 100%;
    height: 100vh;
  }
  .search{
    position: absolute;
    left: 40rpx;
    top: 50rpx;
    width: 650rpx;
    height: 70rpx;
    display: flex;
    justify-content: space-between;
    background: #fff;
    border-radius: 70rpx;
    padding: 0 20rpx;
    box-sizing: border-box;
    button{
      height: 70rpx;
      background: #fff;
      color: #333;
      font-size: 28rpx;
      &:after{
        border: none;
      }
      &.selected{
        color: #f6761c;
        font-weight: bold;
      }
    }
  }
  .cover-info{
    position: absolute;
    left: 0;
    bottom: 0;
    background: #fff;
    width: 100%;
    height: 50vh;
    box-shadow: 0 1px 1px 1px #333;
    overflow: hidden;
    button{
      &:after{
        border: none;
      }
    }
    .close{
      position: absolute;
      right: 10rpx;
      top: 10rpx;
      width: 60rpx;
      height: 60rpx;
      background: url('../../static/imgs/close.png') no-repeat 50% 50%;
      background-size: 30rpx 30rpx;
      outline: none;
      border: none;
      z-index: 3;
    }
    .tabs{
      position: absolute;
      top: 10rpx;
      left: 50%;
      transform: translateX(-50%);
      height: 60rpx;
      width: 400rpx;
      display: flex;
      justify-content: space-between;
      z-index: 4;
      button{
        width: 160rpx;
        font-size: 28rpx;
        height: 60rpx;
        line-height: 60rpx;
        color: #666;
        &.selected{
          background: #71d5a1;
          color: #fff;
        }
      }
    }
    .content{
      box-sizing: border-box;
      width: 100%;
      height: 50vh;
      padding: 80rpx 20rpx 20rpx;
      overflow-y: scroll;
      .h1-title{
        line-height: 50rpx;
        font-size: 30rpx;
        font-weight: bold;
        color: #000;
        margin-bottom: 10rpx;
        white-space:pre-wrap;
        margin-bottom: 30rpx;
      }
      .hine{
        line-height: 40rpx;
        font-size: 24rpx;
        color: #999;
        margin-bottom: 20rpx;
      }
      .introduction{
        padding: 20rpx;
        border-radius: 20rpx;
        background: #f5f5f5;
        line-height: 46rpx;
        font-size: 28rpx;
        white-space:pre-wrap;
        margin-bottom: 20rpx;
        .zone{
          line-height: 46rpx;
          font-size: 30rpx;
          color: #000;
          font-weight: bold;
          margin-bottom: 10rpx;
        }
        .treat{
          font-size: 26rpx;
          line-height: 36rpx;
          color: #fa4630;
        }
      }
      .confirmed-view{
        display: flex;
        font-size: 30rpx;
        line-height: 60rpx;
        height: 60rpx;
        margin-bottom: 10rpx;
        border-radius: 10rpx;
        .city-name{
          padding-left: 20rpx;
          font-size: 30rpx;
          line-height: 60rpx;
          color: #000;
          flex-grow: 1;
          background: #f5f5f5;
        }
        .confirmed{
          box-sizing: border-box;
          padding-left: 20rpx;
          width: 200rpx;
          line-height: 60rpx;
          background: rgb(247, 232, 234);
          color: rgb(174, 33, 44);
        }
        .new-num{
          box-sizing: border-box;
          padding-left: 20rpx;
          width: 200rpx;
          line-height: 60rpx;
          background: rgb(252, 236, 233);
          color: rgb(230, 68, 38)
        }
      }
      .header{
        padding: 10rpx 20rpx 30rpx;
        border-bottom: 1rpx solid #eee;
        .h1{
          display: flex;
          align-items: center;
          line-height: 50rpx;
          font-size: 30rpx;
          font-weight: bold;
          color: #000;
          margin-bottom: 10rpx;
          white-space:pre-wrap;
          .mark{
            background: #fa4630;
            color: #fff;
            font-size: 24rpx;
            line-height: 36rpx;
            padding: 0 10rpx;
            border-radius: 8rpx;
            margin-left: 14rpx;
          }
        }
        .h3{
          line-height: 40rpx;
          font-size: 28rpx;
          color: #666;
          white-space:pre-wrap;
        }
      }
      .ul{
        padding: 20rpx 20rpx 40rpx;
        .title-no{
          width: 110rpx;
          padding: 0 10rpx;
          font-size: 24rpx;
          line-height: 44rpx;
          color: #ff9158;
          background: #fff7f2;
          border-radius: 4rpx;
          text-align: center;
        }
        .title{
          display: flex;
          align-items: center;
          font-size: 24rpx;
          line-height: 36rpx;
          padding-bottom: 10rpx;
          color: #666;
          .manlen{
            font-size: 24rpx;
            font-weight: bold;
            line-height: 36rpx;
            color: #333;
            font-weight: bold;
            padding: 0 3rpx;
          }
        }
        .li{
          margin-bottom: 20rpx;
          .man-info{
            font-size: 30rpx;
            font-weight: bold;
            line-height: 50rpx;
            color: #333;
          }
          .p{
            line-height: 36rpx;
            font-size: 24rpx;
            color: #666;
            padding: 10rpx 0;
            white-space:pre-wrap;
          }
          .track-list{
            padding: 20rpx;
            border-radius: 20rpx;
            background: #f5f5f5;
            .track{
              padding: 10rpx 0 18rpx;
              line-height: 46rpx;
              font-size: 28rpx;
              color: #666;
              white-space:pre-wrap;
              .timer{
                display: inline;
                color: #333;
                font-weight: bold;
                line-height: 40rpx;
              }
              &:last-child{
                border: none;
              }
            }
          }
        }
      }
      .nothing{
        padding: 100rpx 0;
        text-align: center;
        color: #666;
        font-size: 30rpx;
      }
    }
  }
}
</style>
