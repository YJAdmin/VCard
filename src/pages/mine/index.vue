<template>
    <div class="page">
        <div class="page__hd">
            <user-card-component :userCard="userInfo" @link-on-click="linkOnClick" @tab-on-click='tabOnClick'></user-card-component>
        </div>
        <div class="index-link">
            <div class="weui-media-box weui-media-box_small-appmsg index-link-container">
                <div class="weui-cells weui-cells_in-small-appmsg">
                    <navigator url="../rankinglist/rankinglist" class="weui-cell weui-cell_access" hover-class="weui-cell_active">
                        <div class="weui-cell__hd">
                            <image src="../../static/assets/charts.png" style="width: 20px;height: 20px;margin-right: 5px" />
                        </div>
                        <div class="weui-cell__bd weui-cell_primary">
                            <div>排行榜</div>
                        </div>
                        <div class="weui-cell__ft weui-cell__ft_in-access"></div>
                    </navigator>
                    <navigator class="weui-cell weui-cell_access" hover-class="weui-cell_active" @click="goCardHolder">
                        <div class="weui-cell__hd">
                            <image src="../../static/assets/cards.png" style="width: 20px;height: 20px;margin-right: 5px" />
                        </div>
                        <div class="weui-cell__bd weui-cell_primary">
                            <div>名片夹</div>
                        </div>
                        <div class="weui-cell__ft weui-cell__ft_in-access"></div>
                    </navigator>
                </div>
            </div>
        </div>

        <div class="weui-btn-area sub">
            <button class="weui-btn btn-defalut" hover-class="btn-hover" v-if="showShareBtn" @click="onShare">分享</button>
        </div>

        <div class="weui-cells__title" style="height: 20rpx;"></div>

        <view class="mask" @click="preventTouchMove" v-if="showModal"></view>
        <div class="modalDlg" v-if="showModal">
            <image :src='shareImagePath' mode='widthFix' @click="onPreview"></image>
        </div>
        <canvas canvas-id="myCanvas" style="width:545px;height:771px"></canvas>
    </div>
</template>



<script>
import api from "@/utils/api";
import store from "@/store/store";
import userCardComponent from "@/components/card-container/card-container";

export default {
  components: {
    userCardComponent
  },
  data() {
    return {
      wxCode: "",
      grids: [
        {
          title: "人气",
          num: 0,
          icon: "../../static/assets/hot.png",
          type: 1
        },
        {
          title: "点赞",
          num: 0,
          icon: "../../static/assets/zan-active.png",
          type: 2
        },
        {
          title: "收藏",
          num: 0,
          icon: "../../static/assets/collection-active.png",
          type: 3
        }
      ],
      showShareBtn: false,
      showModal: false,
      shareImagePath: "",
      qrCode: ''
    };
  },
  mounted() {
      this.onShow();
  },
  computed: {
    userInfo() {
      return store.state.userInfo;
    }
  },
  onPullDownRefresh() {
    this.getUserInfo();
  },
  onShareAppMessage(res) {
    const strOpenId = this.userInfo.strOpenId;
    const userName = this.userInfo.strName;
    return {
      title: `${userName}的名片`,
      path: `pages/othercard/othercard?strOpenId=${strOpenId}`,
      success: function(res) {
        // 转发成功
      },
      fail: function(res) {
        // 转发失败
      }
    };
  },
  methods: {
    async getUserInfo() {
      const _this = this;
      const openId = store.state.openId;
      let par = {
        strOpenId: openId
      };
      const data = await api.post_login(par);
      
      store.commit("inUserInfo", data);
      this.userInfo = data
      _this.showShareBtn = store.state.inSendBtn;
      // _this.onShow()
      // 停止下拉刷新
      wx.stopPullDownRefresh();
    },
    async checkIsOnline() {
      const _this = this;
      const RES = await api.get_isOnline();
    },
    linkOnClick(type) {
      this.$router.open({
        name: "编辑名片",
        url: "../cardinfo/cardinfo",
        type: "PUSH",
        params: {
          params: this.userInfo
        }
      });
    },
    tabOnClick(info) {
      this.$router.open({
        name: "名片列表",
        url: "../cardlist/cardlist",
        type: "PUSH",
        params: {
          params: info
        }
      });
    },
    goCardHolder() {
      this.$router.open({
        name: "名片夹",
        url: "../cardholder/cardholder",
        type: "PUSH"
      });
    },
    onShare() {
      
      this.onshareImage();
    },
    onshareImage () {
        var _this =this;
        this.showModal = true;
        wx.canvasToTempFilePath({
            x: 0,
            y: 0,
            width: 540,
            height: 687,
            destWidth: 540,
            destHeight: 687,
            canvasId: "myCanvas",
            success: function(res) {
              _this.shareImagePath = res.tempFilePath;
            },
            fail: function(res) {
            }
          });
    },
    async onShow() {
        const _this = this;
        if(this.qrCode == '') {
          const qrCode = await api.get_qrCode({
              url: 'pages/othercard/othercard',
              scene: this.userInfo.strOpenId
          })
          this.qrCode = qrCode.url
        }
        Promise.all([api.wxGetImageInfo({src : this.qrCode}) , api.wxGetImageInfo({src : this.userInfo.strAvatarUrl})]).then(res => {
            _this.drawCanvas(res)
        }).catch(err => {
            _this.drawCanvas()
        })
            
    },
    drawCanvas (res) {
        var res = res ? res : []
        const _this =this;
        const headImage = res[1].path
        const qrCodeImage = res[0].path
        const ctx = wx.createCanvasContext("myCanvas");
            // 头部
            ctx.setFillStyle('#f8f8f8')
            ctx.fillRect(0, 0, 545, 176);
            ctx.drawImage( headImage,  28, 24, 116, 116);
            /* 绘制文字 */
            ctx.setFontSize(23)
            ctx.setFillStyle('#333333')
            ctx.fillText(this.userInfo.strName, 170, 58)
            ctx.setFontSize(20)
            ctx.setFillStyle('#999999')
            ctx.fillText(this.userInfo.strPosition, 170, 94)
            const len = this.userInfo.strIntro.length;
            const strArr = this.userInfo.strIntro.split('')
            let strIntro = ''
            strArr.map((item , index) => {
                if(index < 15) {
                    strIntro += item
                }else  if (index == 15) {
                    strIntro += '......'
                }
            })
            ctx.fillText(strIntro, 170, 128)
            // 底部
            ctx.setFillStyle('white')
            ctx.fillRect(0, 176, 540, 510);
            ctx.setFontSize(24)
            ctx.setFillStyle('#999999')
            ctx.fillText('不用面对面也能递交名片的神器', 105, 637)
            ctx.drawImage(qrCodeImage, 90, 215, 360 , 360);
            /* 绘制 */
            ctx.stroke();
            ctx.draw();
            this.showShareBtn = true
    },
    preventTouchMove() {
      this.showModal = false;
    },
    onPreview () {
        var arr = [this.shareImagePath]
        wx.previewImage({
            current: 0, 
            urls: arr
        })
    }

  }
};
</script>

<style>
.index-link {
  padding-left: 30rpx;
  padding-right: 30rpx;
}
.index-link-container {
  position: relative;
  width: 690rpx;
  border-radius: 4rpx;
  overflow: hidden;
}
.index-button-content {
  margin: 15rpx 30rpx 0.3em;
}

canvas{
  position: fixed;
  top: 0;
  left: 400px;
}
.modalDlg image{
  width: 100%;
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  z-index: 3;
  border: 1px dashed #fff;
}
.mask {
  width: 100%;
  height: 100%;
  position: fixed;
  top: 0;
  left: 0;
  background: #000;
  z-index: 9000;
  opacity: 0.7;
}

.modalDlg {
  width: 545rpx;
  height: 687rpx;
  position: fixed;
  top: 50%;
  left: 0;
  z-index: 99999999999;
  margin: -375rpx 100rpx;
  /* background-color: #fff; */
  border-radius: 12rpx;
  display: flex;
  flex-direction: column;
  align-items: center;
}
</style> 