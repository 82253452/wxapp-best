# 小程序性能优化实战
- swiper动态扩容
- 长列表优化
## swiper动态高度
解决方案：直接获取swier-item内容的高度并设置为swiper高度
```
   selectQueryPromise(id) {
           return new Promise((resove, reject) => {
               const query = wx.createSelectorQuery()
               query.select(id).boundingClientRect().exec(res => {
                   if (res && res.length && res[0]) {
                       resove(res[0])
                   } else {
                       reject(res)
                   }
   
               })
           })
       },
```
应用demo
```
//刷新第几个index 防止加载数据过程中 切换了swiper 导致错误
    async refreshSwiperHeight(index) {
        const list = await this.selectQueryPromise(`.list-${this.data.index}`)
        this.data.swiperTop || (this.data.swiperTop = list.top)
        this.data.index === index && this.setData({
            swiperHeight: list.height,
        })
    },
```
## list 长列表优化
解决方案 
- 一个list分页列表一般为一维数组 列表组件内转为二维数组第一维为第几页第二维为具体数据
set数据的时候只需要set第几页数据提升列表渲染
```
    properties: {
            data: { //是否显示所属圈子
                type: Array,
                value: [],
                observer: function(newVal, oldVal) {
                    const dataIndex = this.data.totalData.length
                    newVal && newVal.length && this.setData({
                        [`totalData[${dataIndex}]`]: newVal
                    },()=>{
                        wx.createIntersectionObserver(this,{ thresholds: [0, 1] }).relativeToViewport().observe(`.test-index-${dataIndex}`, res => {
                            if (res.intersectionRatio === 0) {
                                this.setData({
                                    [`cusData[${dataIndex}].obHidden`]: true,
                                    [`cusData[${dataIndex}].obHeight`]: res.boundingClientRect.height,
                                })
                            } else if (res.intersectionRatio === 1) {
                                this.setData({
                                    [`cusData[${dataIndex}].obHeight`]: res.boundingClientRect.height,
                                    [`cusData.obHidden`]: false,
                                })
                            }else{
                                this.setData({
                                    [`cusData[${dataIndex}].obHidden`]: false,
                                })
                            }
                        })
                    })
                }
    
            },
        },
    data: {
        totalData: [

        ],
        cusData:[]
```
此时组件内传递数据只只需要传递最新的一维数据 组件data内部保存总的数据列表<br/>
此时面临的问题
- 列表刷新
```
methods: {
        clearData(){
            this.setData({totalData:[]})
        }
    }
```
组件内部提供清空数据方法在page中获取组件并且条用组件内部方法如有更好方案欢迎提出
使用demo
```
    const listf = this.selectComponent('.list-0');
    listf.clearData()
```
- 页面无线滚动占用内存过高
在页面增加数据的时候添加监听方法当每页数据离开当前视图的时候
<br/>记录当前view高度 隐藏 pageview中的内容当pageview进入当前视图的时候显示pageview
```$xslt
 wx.createIntersectionObserver(this,{ thresholds: [0, 1] }).relativeToViewport().observe(`.test-index-${dataIndex}`, res => {
                            if (res.intersectionRatio === 0) {
                                this.setData({
                                    [`cusData[${dataIndex}].obHidden`]: true,
                                    [`cusData[${dataIndex}].obHeight`]: res.boundingClientRect.height,
                                })
                            } else if (res.intersectionRatio === 1) {
                                this.setData({
                                    [`cusData[${dataIndex}].obHeight`]: res.boundingClientRect.height,
                                    [`cusData.obHidden`]: false,
                                })
                            }else{
                                this.setData({
                                    [`cusData[${dataIndex}].obHidden`]: false,
                                })
                            }
                        })
```
```$xslt
<view class="contentList">
    <view wx:for="{{totalData}}" wx:key="{{index}}" id="item-page-{{index}}" class="test-index-{{index}}" style="{{cusData[index].obHeight?'height:'+cusData[index].obHeight+'px':''}}">
	   <view wx:if="{{!cusData[index].obHidden}}">
		   <block wx:for="{{item}}" wx:key="{{item.id}}">
			   <content-item content-data="{{item}}" pic-data="{{item.postImages}}" status="{{item.status}}" bindshareimg="setShareInfo" bindpraisestatus="setWdnmd" wx:key="{{item.id}}"></content-item>
		   </block>
	   </view>
    </view>
</view>
```
到这里就暂时解决了页面卡顿的问题
