# 微信小程序云开发——模版消息聊天

* 在小程序中大量储存formId
* 使用云开发下发模版消息
* 使用模版消息相互聊天

## 截图

![](https://blogimg-1252809090.cos.ap-chengdu.myqcloud.com/WeChat_miniapp_template_chat/IMG.jpg)

## 结构

```
├─cloudfunctions 
│  ├─login
│  |   └─index.js
│  ├─moban
|  |   ├─index.js
│  |   ├─package-lock.json
│  |   └─package.json  
│  └─remove
|      ├─index.js
│      ├─package-lock.json
│      └─package.json           　　
├─miniprogram
│  ├─app.js
│  ├─app.json
│  ├─app.wxss
│  └─pages
│      └─fromId
│         ├─index.wxml
│         ├─index.js
│         ├─index.json
│         └─index.wxss
└─project.config.json

```

## 云开发下发模版消息

[详细请移步之前文章](http://www.tysb7.cn/2018/10/15/%E5%B0%8F%E7%A8%8B%E5%BA%8F%E4%BA%91%E5%BC%80%E5%8F%91%E6%A8%A1%E7%89%88%E6%B6%88%E6%81%AF/)

## 思路

1. 在前端中需要大量获取formId，将获取到的formID打上时间戳和用户openid存储到数据库
2. 由于formId只有7天时效，所以需要打上时间戳，定期将7天前的数据删除掉
3. 在使用的时候通过目标用户openid获取formId，使用对应的openid和formId下发模版消息
4. 每使用一个/无效的formId，需要将其删掉

## 使用Demo

1. 修改project.config.json中`appid`
2. 修改miniprogram=>app.js中`env`为自己的云开发环境
3. 上传cloudfunctions中的三个云函数
4. 修改cloudfunctions=>moban=>index.js中的`APPID`和`SECRET`

## 代码

index.wxml

``` html

<form report-submit="true" bindsubmit="button_three">
  <button formType="submit">存储formId</button>
</form>
<form report-submit="true" bindsubmit="button_one">
  <button formType="submit">发给自己模版消息</button>
</form>
<form report-submit="true" bindsubmit="button_two">
  <view class="section">
    <input style='font-size:30rpx;text-align: center;line-height:100rpx;margin:30rpx;' name="input" placeholder="please input here" />
  </view>
  <button formType="submit">发给别人模版消息</button>
</form>

<view style='margin:100rpx;' hidden='{{receiveDataShow}}'>
  <view>收到{{receiveData.sender_openid}}的消息:</view>
  <view style='font-size:50rpx;text-align: center;line-height:100rpx;'>{{receiveData.value}}</view>
  <form report-submit="true" bindsubmit="receive">
    <view class="section">
      <input style='font-size:30rpx;text-align: center;line-height:100rpx;margin:30rpx;' name="input" placeholder="please input here" />
    </view>
    <button formType="submit">回复消息</button>
  </form>
</view>

```
index.js

``` javascript

const db = wx.cloud.database();
const _ = db.command;
Page({
  data: {
    receiveDataShow: true //接收框开关
  },

  /**
   * 生命周期函数--监听页面加载
   */
  onLoad: function(options) {
    var that = this
    if (options.sender_openid) {
      that.setData({
        receiveData: options, //接收数据
        receiveDataShow: false //打开接受信息页面
      })
    }
    console.log((new Date()).valueOf())
    console.log(new Date() - (1000 * 60 * 60 * 24 * 7))
    console.log(new Date(new Date() - (1000 * 60 * 60 * 24 * 7)))
    //调用云函数获取openid
    wx.cloud.callFunction({
      name: 'login',
      data: {},
      success: res => {
        console.log('[云函数] [login] user openid: ', res.result.openid)
        wx.setStorageSync("openid", res.result.openid)
      },
      fail: err => {
        console.error('[云函数] [login] 调用失败', err)
      }
    })
  },

  /**
   * 生命周期函数--监听页面初次渲染完成
   */
  onReady: function() {

  },
  //发送给自己模版消息
  button_one(e) {
    let form_id = e.detail.formId
    let date = new Date();
    let data = JSON.stringify({
      "keyword1": {
        "value": date
      },
      "keyword2": {
        "value": "2015年01月05日 12:30"
      }
    })
    //调用云函数发送模版消息
    wx.cloud.callFunction({
      name: 'moban',
      data: {
        openid: wx.getStorageSync("openid"),
        template_id: "Lwh1NYwTUzHIjDwWlbPVhUJWQLW__nLkj31p2lMH8AM",
        page: "",
        form_id,
        data,
        emphasis_keyword: "keyword1.DATA"
      },
      success: res => {
        console.log('[云函数] [login] : ', res)
      },
      fail: err => {
        console.error('[云函数] [login] 调用失败', err)
      }
    })
  },
  //调用云函数发送给指定用户
  button_two(e) {
    console.log(e.detail.value.input)
    let value = e.detail.value.input //获取输入
    let week = new Date() - (1000 * 60 * 60 * 24 * 7) //建立7天时间戳
    //储存formId，并打时间戳
    db.collection('formId').add({
        data: {
          openid: wx.getStorageSync("openid"),
          formId: e.detail.formId,
          date: (new Date()).valueOf()
        }
      })
      .then(res => {
        console.log(res)
      })
    //获取formId数据 
    db.collection('formId').where({
      _openid: "ogff70LGj__6xlA_lXbB-KoBWEo0",
      date: _.gt(week) //获取7天内
    }).get().then(res => {
      console.log(res.data)
      var formIdList = res.data
      let date = new Date();
      let data = JSON.stringify({
        "keyword1": {
          "value": value
        },
        "keyword2": {
          "value": date
        }
      })
      //调用云函数发送模版消息
      wx.cloud.callFunction({
        name: 'moban',
        data: {
          openid: formIdList[0].openid,
          template_id: "Lwh1NYwTUzHIjDwWlbPVhUJWQLW__nLkj31p2lMH8AM",
          page: "/pages/fromID/index?sender_openid=" + wx.getStorageSync("openid") + "&value=" + value, //携带参数
          form_id: formIdList[0].formId,
          data,
          emphasis_keyword: "keyword1.DATA"
        },
        success: res => {
          console.log('模版消息发送成功: ', res)
          this.remove(formIdList[0]._id); //调用删除formId函数
        },
        fail: err => {
          console.error('模版消息发送失败：', err)
        }
      })
    })
  },
  button_three(e) {
    console.log(e.detail.formId)
    console.log(new Date())
    db.collection('formId').add({
        data: {
          openid: wx.getStorageSync("openid"),
          formId: e.detail.formId,
          date: (new Date()).valueOf()
        }
      })
      .then(res => {
        console.log(res)
      })
  },
  //回复消息
  receive(e) {
    console.log(e.detail.value.input)
    let value = e.detail.value.input
    let week = new Date() - (1000 * 60 * 60 * 24 * 7)
    db.collection('formId').add({
        data: {
          openid: wx.getStorageSync("openid"),
          formId: e.detail.formId,
          date: (new Date()).valueOf()
        }
      })
      .then(res => {
        console.log(res)
      })
    db.collection('formId').where({
      _openid: this.data.receiveData.sender_openid,
      date: _.gt(week)
    }).get().then(res => {
      console.log(res.data)
      var formIdList = res.data
      let date = new Date();
      let data = JSON.stringify({
        "keyword1": {
          "value": value
        },
        "keyword2": {
          "value": date
        }
      })
      wx.cloud.callFunction({
        name: 'moban',
        data: {
          openid: formIdList[0].openid,
          template_id: "Lwh1NYwTUzHIjDwWlbPVhUJWQLW__nLkj31p2lMH8AM",
          page: "/pages/fromID/index?sender_openid=" + wx.getStorageSync("openid") + "&value=" + value,
          form_id: formIdList[0].formId,
          data,
          emphasis_keyword: "keyword1.DATA"
        },
        success: res => {
          console.log('模版消息发送成功: ', res)
          this.remove(formIdList[0]._id);
        },
        fail: err => {
          console.error('模版消息发送失败：', err)
        }
      })
    })
  },
  //调用云函数删除使用过的formId数据
  remove(id) {
    wx.cloud.callFunction({
      name: 'remove',
      data: {
        id,
      },
      success: res => {
        console.log('删除成功：', res)
        if (res.result.stats.removed == 1) {
          wx.showToast({
            title: '删除formId成功',
          })
        }
      },
      fail: err => {
        console.log('删除失败：', err)
      }
    })
  }
})

```

