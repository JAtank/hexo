---
title: vue.js长按自定义指令
date: 2017-10-31 21:16:23
tags:
---
适用于vue.js的长按指令
最近的项目一直做的是Android和前端的混编。大致是用一个Android的壳子，套在里边的是通过webview加载的前端页面。前端用的是vue.js。之前有个项目中有长按事件的需求，自己通过js写了一个，但是总觉得在vue里边使用不是很舒服，所以才想着通过自定义指令封装一下。其实运用的思路和我之前的写的js是相同的，长按是通过setTimeOut()来控制实现的。具体如下：
<pre><code>
	/**
	 * 自定义指令：元素的长按事件
	 * 使用：
	 * v-long-click="callback" //在一定时间后执行事件
	 * v-long-click.continuous="callback" //连续执行事件
	 */
	Vue.directive('long-click',{
	  bind:function (el,binding) {
	    if(typeof binding.value !== 'function'){ //不是方法时给出提示
	      console.log("-------------"+binding.value+"------------");
	    }
	    var time = 0;
	    const isContinuous = binding.modifiers.continuous; //判断是否为连续执行
	    const handlerStart = (e) => { //touchstart 执行事件
	      if(isContinuous==true){
	        el.longTime1 = setInterval(() => {
	          binding.value(e);
	        },300);
	      }else{
	        el.longTime2 = setInterval(() => {
	           time++;
	           if(time==15){
	             binding.value(e);
	             window.clearInterval(el.longTime2);
	             time=0;
	           }
	        },100)
	      }
	    };
	    el.__vueLongClick__ = handlerStart;
	    const handlerEnd = (e) => { //touchend 执行事件
	      if(isContinuous==true){
	        window.clearInterval(el.longTime1);
	      }else {
	        window.clearInterval(el.longTime2);
	      }
	    };
	    el.__vueLongClickOver__ = handlerEnd;
	    el.addEventListener('touchstart',handlerStart); //添加事件监听
	    el.addEventListener('touchend',handlerEnd);
	  },
	  unbind:function (el) {
	    el.removeEventListener('touchstart');
	    el.removeEventListener('touchend');
	    el.__vueLongClick__ = null;
	    el.__vueLongClickOver__=null;
	  }
	});
</code></pre>
这个指令是通过我在网上查询其他自定义指令的书写方法来写的。分为：长按之后执行某一事件和连续执行某一事件。测试是有效的，如果有不妥的地方，大家可以帮忙指正。当然指令的具体执行间隔时间大家可以自行更改。