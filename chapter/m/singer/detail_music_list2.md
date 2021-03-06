# 歌手详情页-music-list组件2

前面章节把基础的交互滚动做到了，但是出现了问题，在顶部的时候，由于我们要做的是留出一部分距离，但是这部分距离由于css 和 我们实现的思路原因，出现了问题。下面来讲解怎么解决这个问题

解决问题的思路：

1. 查看dom结构和css的实现，找到当改变背景图部分的z-index则会把列表盖住
2. 那么，当滚动到不再滚动的距离时，改变 背景图部分的z-index和背景图部分的高度
3. 当可以滚动的时候，再重置回去

总结：动画效果完全依赖css和js不断的改变

```javascript
scrollY (newY) {
        let translateY = Math.max(this.minTranslateY, newY)
        if (translateY === this.minTranslateY) {
//          let zindex = 0
          let bgImageStyle = this.$refs.bgImage.style
          bgImageStyle['padding-top'] = 0
          bgImageStyle['height'] = `${RESERVED_HEIGHT}px`
          bgImageStyle['z-index'] = 10
          console.log(`  --- translateY:`, translateY, `newY:`, newY)
        } else {
          // 让背景层跟着网上移动
          this.$refs.bgLayer.style['transform'] = `translate3d(0,${translateY}px,0)`
          let bgImageStyle = this.$refs.bgImage.style
          bgImageStyle['padding-top'] = `70%`
          bgImageStyle['height'] = 0
          bgImageStyle['z-index'] = 0
          console.log(`translateY:`, translateY, `newY:`, newY)
        }
```

## 下拉让图片有放大缩小的交互效果和上拉高斯模糊
![](/assets/musicapp/歌手详情歌曲列表交互效果图上拉ios高斯模糊.png)

我们要做的效果有两个

1. 下拉的时候 图片随着下拉的力度（距离）放大，放开后或则上拉会恢复正常
2. 上拉的时候如上图，开始高斯模糊，到最顶部的时候，就很模糊

要做这两个效果，先来说说思路：

1. 下拉的时候 图片随着下拉的力度（距离）放大，放开后或则上拉会恢复正常
  
  下拉（在scorll中就是y轴距离为0的时候，才会产生下拉效果）和放开后自动回到原位 都是浏览器和bscorll自带的效果。我们只要监听大于 0 的时候就是在下拉了；
  
  1. 监听scorll.y > 0 的时候，更改图片部分的transform.scale()属性
  2. 还要根据y的下拉距离计算scale的数值，越下拉越放大
  
2. 上拉的时候如上图，开始高斯模糊，到最顶部的时候，就很模糊

  上拉的时候不能用缩小来做，图片缩小 铺不满容器，太丑。所以使用其他的效果来弥补，比如ios上的高斯模糊
  
  1. 监听scorll.y < 0 的时候，更改 filter的模糊度，backdrop-filter=blur(${blur}px)
  2. 越往上拉越模糊，所以还要根据上拉距离计算模糊值
  

```javascript
watch: {
      scrollY (newY) {
        let translateY = Math.max(this.minTranslateY, newY)
        let zindex = 0
        if (translateY === this.minTranslateY) {
          let bgImageStyle = this.$refs.bgImage.style
          bgImageStyle['padding-top'] = 0
          bgImageStyle['height'] = `${RESERVED_HEIGHT}px`
          zindex = 10
          console.log(`  --- translateY:`, translateY, `newY:`, newY)
        } else {
          // 让背景层跟着网上移动
          this.$refs.bgLayer.style['transform'] = `translate3d(0,${translateY}px,0)`
          let bgImageStyle = this.$refs.bgImage.style
          bgImageStyle['padding-top'] = `70%`
          bgImageStyle['height'] = 0
          zindex = 0
          console.log(`translateY:`, translateY, `newY:`, newY, 'this.minTranslateY:', this.minTranslateY)
        }

        let percent = Math.abs(newY / this.bgImageHeight)
        // 通过缩放来控制图片的大小
        let scale = 1
        let blur = 0
        if (newY > 0) {
          scale = 1 + percent
          zindex = 10  // 解决在下拉的时候，图片虽然有放大缩小的效果，但是由于后来居上，覆盖了放大的显示部分
        } else {
          // 往上拉的时候，不能使用缩放了，因为图片可以放大，但是缩小的话，背景肯定铺不满容器了，很丑陋
          // 那么实现一个高斯模糊的效果，越到顶部越模糊
          blur = Math.min(20 * percent, 20) // 最大限制到20
        }
        // backdrop-filter 可能只有ios才能看到的高斯模糊效果
        this.$refs.filter.style['backdrop-filter'] = `blur(${blur}px)`
        this.$refs.filter.style['webkitBackdrop-filter'] = `blur(${blur}px)`
        let bgImageStyle = this.$refs.bgImage.style
        bgImageStyle['z-index'] = zindex
        bgImageStyle['transform'] = `scale(${scale})`
        bgImageStyle['webkitTransform'] = `scale(${scale})`
      }
    }
```

## 优化 - css前缀

在上面的过程中，在js里面写的css是不会经过vue-loader处理的，没有于处理器处理，那么用一些css属性的时候就要兼顾前缀，比较麻烦。接下来就用js来处理这个前缀；

在什么浏览器下就用什么前缀（但是在这个测试用我看最后显示的还是用的标准的，不知道是为什么，但是代码中的确是没有错的）

思路：

1. 在js中创建一个div
2. 检测这个div是否拥有指定的前缀属性
3. 得到当前的浏览器css前缀，然后加工返回

src/common/js/dom.js
```javascript
// 能力检测
let elementStyle = document.createElement('div').style
// 供应商,一个立即执行函数
let vendor = (() => {
  let transformNames = {
    webkit: 'webkitTransform',
    Moz: 'MozTransform',
    ms: 'msTransform',
    standard: 'transform'  // 标准
  }

  // 查看dom的style里面是否包含这个属性，包含则返回
  for (let key in transformNames) {
    if (elementStyle[transformNames[key]] !== undefined) {
      return key
    }
  }
  // 所有的前缀都不支持的话，就有问题了
  return false
})()

export function prefixStyle (style) {
  if (vendor === false) {
    return false
  }

  if (vendor === 'standard') {
    return style
  }

  // 加上前缀，首字母大写，比如：transform 加工之后返回 webkitTransform
  return vendor + style.charAt(0).toUpperCase() + style.substr(1)
}
```

使用处调用
```javascript
 import { prefixStyle } from 'common/js/dom'
 
  const transform = prefixStyle('transform')
  const backdropFilter = prefixStyle('backdrop-filter')
  
  
  比如这个：
  this.$refs.bgLayer.style[transform] = `translate3d(0,${translateY}px,0)`
```

## 收尾 - 返回按钮
```javascript
 <div class="back" @click="back">
      <i class="icon-back"></i>
    </div>
    
 // 直接使用路由提供的返回即可   
 back () {
    this.$router.back()
  }   
```

## 收尾 - 随机播放按钮
![](/assets/musicapp/歌手详情列表随机播放全部按钮.png)

看上图，在背景图的上面，所以dom写在背景图部分
```html
 <div class="bg-image" :style="bgStyle" ref="bgImage">
      <div class="play-wrapper">
        <div class="play">
          <i class="icon-play"></i>
          <span class="text">随机播放全部</span>
        </div>
      </div>
      <div class="filter" ref="filter"></div>
    </div>
```
```css
.play-wrapper {
        position absolute
        bottom 20px
        z-index 50
        width 100%
        .play {
          box-sizing border-box
          border 1px solid $color-theme
          width 135px
          border-radius 100px
          padding 7px 0
          margin 0 auto //外部宽度百分白，内部宽度定死，然后 margin auto 水平居中
          text-align center // 然后让内部的文字和图标居中
          font-size 0
          color: $color-theme
          .icon-play {
            display inline-block
            vertical-align middle
            margin-right 6px
            font-size $font-size-medium-x
          }
          .text {
            display inline-block
            vertical-align middle
            font-size $font-size-small
          }
        }
      }
```

## 优化 - 按钮等待列表渲染完成后再渲染
```html
 <div class="play-wrapper" v-show="songs.length >0 ">
```

思路是，等待歌曲列表有数据后，再显示按钮

## 优化 - 按钮在列表滚动到顶部的时候样式错乱
![
](/assets/musicapp/歌手详情列表随机播放全部按钮滚动到顶部的时候样式错乱.png)

造成上面图片的原因是： 还记得我们在实现这里滚动交互效果的时候，滚动到顶部是修改了背景图片区域的容器高度，而这里的按钮又是一个绝对定位，背景图片容器是 position relative，所以这个按钮就会跑上去。

要解决这个问题呢；我们在滚动到顶部的时候，让这个按钮消失，其他地方的时候就显示。

这里为什么直接操作dom呢？上面为了按钮渲染的实际已经用了 v-show，所以这里就直接操作dom了
```javascript
        if (translateY === this.minTranslateY) {
          let bgImageStyle = this.$refs.bgImage.style
          bgImageStyle['padding-top'] = 0
          bgImageStyle['height'] = `${RESERVED_HEIGHT}px`
          zindex = 10
          this.$refs.playBtn.style.display = 'none'
        } else {
          // 让背景层跟着网上移动
          this.$refs.bgLayer.style[transform] = `translate3d(0,${translateY}px,0)`
          let bgImageStyle = this.$refs.bgImage.style
          bgImageStyle['padding-top'] = `70%`
          bgImageStyle['height'] = 0
          zindex = 0
          
          this.$refs.playBtn.style.display = ''
        }
```
