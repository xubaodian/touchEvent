## 原生js实现移动端点击、长按、左滑、右滑、上滑、下滑等事件模拟
为什么要模拟这些事件？<br>
1、上述这些事件中，浏览器直接支持的事件只有点击，而其它事件使用频率也很高。<br>

2、移动端web原生点击事件会有300ms的延迟，因为用户肯能双击，为了判断用户是单击还是双击，所以会有这个延迟，这个延迟会衍生很多问题，例如点击穿透。所以我们可以不用原生的点击事件，而使用模拟点击事件。<br>

如何模拟这些事件呢？<br>
我们可以总结这些操作，都是手指先触摸屏幕，然后在离开。不同点在于滑动事件手指有位移，而点击事件手指没有位移。<br>
首先想到的就是所有浏览器都是支持touchstart、touchmove和touchend事件的，我们可以利用这些事件来模拟上述事件。<br>

原理如下：<br>
1、监听dom的touchstart和touchend事件。<br>
2、分别记录touchstart、touchend事件的位置和时间，计算位移delta（包括x和y）和时间间隔timegap。<br>
3、根据delta和timegap的值，判断属于哪种事件。有两种情况：<br>
- delta中x和y都很小<br>
这是点击操作，用户点击按钮等时，理论上是不会有位移的，但是实际中也可能发生一个很小的位移，毕竟手指不是精密仪器。<br>
如果时间间隔timegap较小，则属于点击，如果timegap较大，属于长按操作。<br>
- delta中的x或y比较大<br>
这种情况下，就是手指发生滑动操作了，至于是左右滑动，还是上下滑动，根据x和y的大小来判断。<br>
|x| > |y|（|x|代表x的绝对值），左右滑动，x>0,右滑，反之左滑。<br>
|x| <= |y|，上下滑动，y>0,下滑，反之上滑。<br>

这样就模拟了移动端web中的这几个事件了。代码如下：
```javascript
/**
     * 用touch事件模拟点击、左滑、右滑、上拉、下拉等时间，
     * 是利用touchstart和touchend两个事件发生的位置来确定是什么操作。
     * 例如：
     * 1、touchstart和touchend两个事件的位置基本一致，也就是没发生位移，那么可以确定用户是想点击按钮等。
     * 2、touchend在touchstart正左侧，说明用户是向左滑动的。
     * 利用上面的原理，可以模拟移动端的各类事件。
    **/
   const EventUtil = (function() {

    //支持事件列表
    let eventArr = ['eventswipeleft', 'eventswiperight', 'eventslideup', 'eventslidedown', 'eventclick', 'eventlongpress'];

    //touchstart事件，delta记录开始触摸位置
    function touchStart(event) {
      this.delta = {};
      this.delta.x = event.touches[0].pageX;
      this.delta.y = event.touches[0].pageY;
      this.delta.time = new Date().getTime();
    }

    /**
     * touchend事件，计算两个事件之间的位移量
     * 1、如果位移量很小或没有位移，看做点击事件
     * 2、如果位移量较大，x大于y，可以看做平移，x>0,向右滑，反之向左滑。
     * 3、如果位移量较大，x小于y，看做上下移动，y>0,向下滑，反之向上滑
     * 这样就模拟的移动端几个常见的时间。
     * */
    function touchEnd(event) {
      let delta = this.delta;
      delete this.delta;
      let timegap = new Date().getTime() - delta.time;
      delta.x -= event.changedTouches[0].pageX;
      delta.y -= event.changedTouches[0].pageY;  
      if (Math.abs(delta.x) < 5 && Math.abs(delta.y) < 5) {
        if (timegap < 1000) {
          if (this['eventclick']) {
            this['eventclick'].map(function(fn){
              fn(event);
            });
          }
        } else {
          if (this['eventlongpress']) {
            this['eventlongpress'].map(function(fn){
              fn(event);
            });
          }
        }
        return;
      }
      if (Math.abs(delta.x) > Math.abs(delta.y)) {
        if (delta.x > 0) {
          if (this['eventswipeleft']) {
            this['eventswipeleft'].map(function(fn){
              fn(event);
            });
          }
        } else {
          this['eventswiperight'].map(function(fn){
            fn(event);
          });
        }
      } else {
        if (delta.y > 0) {
          if (this['eventslidedown']) {
            this['eventslidedown'].map(function(fn){
              fn(event);
            });
          }
        } else {
          this['eventslideup'].map(function(fn){
            fn(event);
          });
        }
      }
    }

    function bindEvent(dom, type, callback) {
      if (!dom) {
        console.error('dom is null or undefined');
      }
      let flag  = eventArr.some(key => dom[key]);
      if (!flag) {
        dom.addEventListener('touchstart', touchStart);
        dom.addEventListener('touchend', touchEnd);
      }
      if (!dom['event' + type]) {
        dom['event' + type] = [];
      }
      dom['event' + type].push(callback);
    }

    function removeEvent(dom, type, callback) {
      if (dom['event' + type]) {
        for(let i = 0; i < dom['event' + type].length; i++) {
          if (dom['event' + type][i] === callback) {
            dom['event' + type].splice(i, 1);
            i--;
          }
        }
        if (dom['event' + type] && dom['event' + type].length === 0) {
          delete dom['event' + type];
          let flag  = eventArr.every(key => !dom[key]);
          if (flag) {
            dom.removeEventListener('touchstart', touchStart);
            dom.removeEventListener('touchend', touchEnd);
          }
        }
      }
    }
    return {
      bindEvent,
      removeEvent
    }
   })();
```
在闭包中定义了几个事件处理操作，EventUtil有两个方法，bindEvent绑定事件，removeEvent是移除事件绑定。<br>
支持六个事件：<br>
swipeleft是左滑事件，swiperight是右滑事件，slideup是上滑事件，slidedown下滑事件，click点击事件，longpress长按点击事件。<br>

使用案例如下：<br>
在页面中引用上述代码：
```html
<script src="./EventUtil.js"></script>
```
测试代码如下,代码中有注释，可以看到如何应用这些模拟事件：
```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8" />
  <meta http-equiv="X-UA-Compatible" content="IE=edge">
  <title>Page Title</title>
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <style>
    #main{
      width: 100%;
      height: 300px;
      background: blue;
      color: white;
      font-size: 20px;
      text-align: center;
    }
  </style>
  <!--引入事件-->
  <script src="./EventUtil.js"></script>
</head>
<body style="font-size: 14px;">
  <div id="main"></div>
  <button id="test">接触左滑绑定</button>
   <script>
     //获取dom
     let domContent = document.querySelector('#main');

     //定义各类事件，为了可以解除事件绑定，事件回调不使用匿名函数
     function handleClick() {
       alert('点击事件');
     }
     function handleLeft() {
       alert('左滑事件');
     }
     function handleRight() {
       alert('右滑事件');
     }
     function handleUp() {
       alert('下滑事件');
     }
     function handleDown() {
       alert('上滑事件');
     }
     function handleLong() {
       alert('长按事件');
     }
     //绑定点击事件
     EventUtil.bindEvent(domContent, 'click', handleClick);
     //绑定两次左滑事件
     EventUtil.bindEvent(domContent, 'swipeleft', handleLeft);
     EventUtil.bindEvent(domContent, 'swipeleft', handleLeft);
     //绑定右滑事件
     EventUtil.bindEvent(domContent, 'swiperight', handleRight);
     //上滑事件
     EventUtil.bindEvent(domContent, 'slideup', handleUp);
     //下滑事件
     EventUtil.bindEvent(domContent, 'slidedown', handleDown);
     //长按点击事件
     EventUtil.bindEvent(domContent, 'longpress', handleLong);

     //接触绑定按钮
     let btnTest = document.querySelector('#test');
     function removeLeft() {
       //接触左滑事件绑定
       EventUtil.removeEvent(domContent, 'swipeleft', handleLeft);
     }
     //绑定点击事件
     EventUtil.bindEvent(btnTest, 'click', removeLeft);
   </script>
</body>
</html>
```
测试效果如下:
<img src="./touch.gif"><br>
有疑问的可以留言或发送邮件至472784995@qq.com。
