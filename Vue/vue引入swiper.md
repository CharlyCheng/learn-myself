
---
title: vue引入swiper实现轮播图
categories:
- 前端开发
tags:
- Vue
- Swiper
---
图片轮播是前端中经常需要实现的一个功能，因为需要[实现年度H5传播](http://activity.xyqb.com/move-ahead),实现滑动切换、动画等功能。考察过vue-awesome-swiper/vue-swiper等封装的库，都会出现一些问题，而且自己的业务代码Vue1.0，最后选择swiper.js

## Swiper

#### Swiper的具体使用教程及详细API，参考[Swiper中文网](http://www.swiper.com.cn/)

Swiper是纯Javascript打造的滑动特效插件，面向手机、平板电脑等移动终端。

Swiper能实现触屏焦点图、触屏Tab切换、触屏多图切换等常用效果。

Swiper开源、免费、稳定、使用简单、功能强大，是架构移动终端网站的重要选择

#### 一、引入swiper
安装swiper
``` bash
npm install --save swiper
```
引用两个文件
``` bash
import Swiper from "swiper";
import "swiper/dist/css/swiper.min.css";
```

#### 二、HTML代码

``` bash
<template>
  <div class="swiper-container" :class="swipeid">
      <div class="swiper-wrapper">
          <!-- 存放具体的轮播内容 -->
          <slot name ="swiper-con"></slot>
      </div>
      <!-- 分页器 -->
      <div :class="{'swiper-pagination':pagination}"></div>
  </div>
</template>
```

#### 三、初始化swiper
初始化之前，根据Swiper用法的了解，先确定轮播组件需要的属性信息，然后通过父组件props传递给封装的Swiper组件

``` bash
props: {
    swipeid: {
      type: String,
      default: ""
    },
    effect: {
      type: String,
      default: "slide"
    },
    loop: {
      type: Boolean,
      default: false
    },
    direction: {
      type: String,
      default: "horizontal"
    },
    pagination: {
      type: Boolean,
      default: true
    },
    paginationType: {
      type: String,
      default: "bullets"
    },
    autoPlay: {
      type: Number,
      default: 3000
    }
  }
```

下面解释属性含义
*   swipeid         
**  轮播容器class属性的类名
*   effect          
**  图片的 切换效果，默认为"slide"，还可设置为"fade", "cube", "coverflow","flip"
*   loop            
**  设置为true 则开启loop模式。loop模式：会在原本图片前后复制若干个图片并在合适的时候切换，让Swiper看起来是循环的
*   direction       
**  图片的滑动方向，可设置水平(horizontal)或垂直(vertical)
*   pagination      
**  使用分页导航
*   paginationType
**  分页器样式类型，可设置为"bullets", "fraction", "progressbar", "custom"
*   autoPlay	    
**  设置为true启动自动切换，并使用默认的切换设置
全都是swiper.js定义的属性

初始化Swiper时，需要传入两个参数，轮播容器的类名和代表图片轮播组件详细功能的对象

```bash
var that = this;
    this.dom = new Swiper("." + that.swipeid, {
      //循环
      loop: that.loop,
      //分页器
      pagination: {
        el: ".swiper-pagination",
        bulletClass : 'swiper-pagination-bullet',
            },
      //分页类型
      paginationType: that.paginationType,
      //自动播放
      autoPlay: that.autoPlay,
      //方向
      direction: that.direction,
      //特效
      effect: that.effect,
      //用户操作swiper之后，不禁止autoplay
      disableOnInteraction: false,
      //修改swiper自己或子元素时，自动初始化swiper
      observer: true,
      //修改swiper的父元素时，自动初始化swiper
      observeParents: true
    });
  }
```
#### 四、示例代码
1.HTML代码
```bash
<m-swipe swipeid="swipe" ref="swiper" :autoPlay="3000" effect="slide">
      <div v-for="top in tops" :key="top.id" class="swiper-slide" slot="swiper-con" >
        <img :src="top.image">
        <h3>{{top.title}}</h3>
      </div>
</m-swipe>
```

2.css代码
```bash
.swiper-container {
    width: 100%;
  }
  .swiper-slide {
    height: 8rem;
    overflow: hidden;
    position: relative;
  }
.swiper-slide {
  div {
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    opacity: 0.4;
    position: absolute;
    background-color: @blue;
  }
  img {
    top: 50%;
    position: relative;
    transform: translate(0, -50%);
  }
  h3 {
    width: 70%;
    color: #fff;
    margin: 0;
    font-size: 0.5rem;
    line-height: 1rem;
    right: 5%;
    bottom: 2.6rem;
    text-align: right;
    position: absolute;
    text-shadow: 1px 1px 10px rgba(0, 0, 0, 0.5);
    &:before {
      content: "";
      width: 3rem;
      bottom: -0.6rem;
      right: 0;
      display: block;
      position: absolute;
      border: 2px solid @yellow;
    }
  }
}
.swiper-pagination-bullet-active {
  background: #fff;
}
.swiper-container-horizontal > .swiper-pagination-bullets {
    bottom: 1rem;
    width: 95%;
    text-align: right;
  }

```

#### 总结
1.引入swiper-animation 代码时候会各种报错，原因是压缩的代码断行出现问题，使用未压缩的版本
2.vue-awesome-swiper 初始化后height 100%，总会出现各种问题，添加自定义动画后滑动快慢都会出现问题
3.vue-Swiper    只有简单的的轮播功能，没有添加动画库

#### 本文参考
[https://juejin.im/post/5a7957ed6fb9a0633d71bb94](https://juejin.im/post/5a7957ed6fb9a0633d71bb94)
