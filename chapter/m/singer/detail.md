# 歌手详情页

![](/assets/musicapp/歌手页和歌手详情页面.png)

上图右侧效果图，分析下功能：

1. 顶部有歌手名称 和 返回按钮
2. 接着是歌手图片
3. 下面是歌手的歌曲列表
    
    歌手列表可以滚动，但是当向下滚动的时候，需要把歌曲列表顶部往上移动一段距离，然后再固定住，滚动到最顶端的时候，需要还原成上图那样。
    
功能分析出来了，来讲一下大概的思路：

1. 在歌手列表中使用子路由
2. 歌手详情页在子路由中全屏渲染，z-index稍微大一点（合理运用好css特性也可以不使用z-index，反正就是在歌手列表上面）
3. 路由路径中传递歌手id
4. 点击歌手列表中的item进行跳转（所以给封装的listView暴露点击事件）

## 子路由配置

**src/components/singer-detail/Singer-detail.vue 详情页面**
```javascript
<template>
  <div class="singer-detail">
      xxx
  </div>
</template>

<script type="text/ecmascript-6">
  export default {
    data () {
      return {}
    }
  }
</script>

<style scoped lang="stylus" rel="stylesheet/stylus">
  @import "~common/stylus/variable"

  .singer-detail
    position fixed
    z-index 100
    top 0
    left 0
    right 0
    bottom 0
    background $color-background
</style>
```

**路由配置**

这里注意看下，参数的用法
```javascript
 {
      path: '/singer',
      component: Singer,
      children: [{
        path: ':id',
        component: SingerDetail
      }]
    }
```

**路由的跳转**
```javascript
    this.$router.push({
      path: `/singer/${singer.id}`
    })
```

上面跳转之后在地址栏呈现:`/singer/5062`

## 给路由跳转增加跳转效果

进入的时候，从右侧往左侧滑动，离开的时候从左侧(当前)滑动到右侧
```html
<transition name="slide">
      <router-view></router-view>
    </transition>
```
```css

  .slide-enter-active, .slide-leave-active
    transition all 0.3s

  // 进入的时候 加上.slide-enter-active slide-enter，在路由进入的时候把页面重置到最右边，然后立即移除（100% -> 0 加0.3秒的过渡效果）
 // 接着 过度效果结束的时候，元素被添加  .slide-enter-active .slide-enter-to(然而这里to并没有添加任何效果)，然后立即移除
 // 离开开始时候元素被添加 .slide-leave-active .slide-leave（这里没有定义任何效果）
 // 离开过渡效果结束的时候，元素被添加 .slide-leave-active .slide-leave-to(当前页面的x轴就是0，过度到100%，再加上0.3的过度时间就有效果了）
 
  .slide-enter, .slide-leave-to
    transform translate3d(100%, 0, 0)
```

具体的 css 类名 查看官网教程：https://cn.vuejs.org/v2/guide/transitions.html；
上面的文字解说其实有点蒙，结合说明和自己手动把这两个css效果放到一个元素上，移除添加查看效果，然后再结合官网对于这几个css类名的解释；

![](/assets/musicapp/vue过度效果css类名说明.png)

看上面的图，通过实际表现出来的现象可以理解为：
把上面的进入从右侧划入，离开从当前往右侧划出的效果翻译下
enter -> enter-to : 100%，马上移除 -> 0 ,加上active的0.3秒过渡效果
leave -> leave-to : 0 -> 100%（过度完成后被移除）

官网文档解释说明：leave-to在leave开始时触发，也就是leave被移除时，leave-to被添加到元素上，过度效果被完成后，再移除

