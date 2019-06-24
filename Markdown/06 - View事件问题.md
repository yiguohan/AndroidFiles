[TOC]

### 1. 简述Android的事件分发机制？dispatchTouchEvent方法的作用是什么？说下View和ViewGroup分发事件？

#### 1.1 简述Android的事件分发机制？

事件分发顺序：Activty->ViewGroup->View

主要方法：dispatchTouchEvent-分发事件、onInterceptTouchEvent-当前View是否拦截该事件、onTouchEvent-处理事件

1. 父View调用dispatchTouchEvent开启事件分发。
2. 父View调用onInterceptTouchEvent判断是否拦截该事件，一旦拦截后该事件的后续事件(如DOWN之后的MOVE和UP)都直接拦截，不会再进行判断。
3. 如果父View进行拦截，父View调用onTouchEvent进行处理。
4. 如果父View不进行拦截，会调用子View的dispatchTouchEvent进行事件的层层分发。

#### 1.2 dispatchTouchEvent方法的作用是什么？

1. 用于进行事件的分发
2. 只要事件传给当前View，该方法一定会被调用
3. 返回结果受到当前View的onTouchEvent和下级View的dispatchTouchEvent影响
4. 表示是否消耗当前事件

#### 1.3 ViewGroup事件分发伪代码

![image](https://upload-images.jianshu.io/upload_images/4432347-f6eea7476ed0e05c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 1.4 View事件分发伪代码

![image](https://upload-images.jianshu.io/upload_images/4432347-4ea710caf6112abe.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 1.5 View和ViewGroup在dispatchTouchEvent上的区别

- ViewGroup在dispatchTouchEvent()中会进行事件的分发。
- View在dispatchTouchEvent()中会对该事件进行处理。

### 2. onInterceptTouchEvent方法作用是什么？onTouchEvent的方法的作用是什么？

#### 2.1 onInterceptTouchEvent方法作用是什么？

在dispatchTouchEvent的内部调用，用于判断是否拦截某个事件

View和ViewGroup在onInterceptTouchEvent上的区别：View没有该方法，View会处理所有收到的事件，但不一定会消耗该事件。onInterceptTouchEvent是ViewGroup中添加的方法，用于判断是否拦截该事件。

#### 2.2 onTouchEvent的方法的作用是什么？

- 在dispatchTouchEvent的中调用，用于处理点击事件
- 返回结果表示是否消耗当前事件

### 3. 滑动冲突有哪些场景？滑动冲突处理原则是什么？滑动冲突解决办法有哪些？分别是如何解决的？

#### 3.1 滑动冲突有哪些场景？

1. 内层和外层滑动方向不一致：一个垂直，一个水平。比如轮播图ViewPager和ScrollView

2. 内存和外层滑动方向一致：均垂直or水平。比如scrollView和RecyclerView

3. 前两者层层嵌套。比如ScrollView和RecyclerView(recyclerView中又嵌套recyclerView)

#### 3.2 滑动冲突处理原则

- 对于内外层滑动方向不同，只需要根据滑动方向来给相应控件拦截
- 对于内外层滑动方向相同，需要根据业务来进行事件拦截，规定何时让外部View拦截事件何时由内部View拦截事件。
- 前两者嵌套的情况，根据前两种原则层层处理即可。

#### 3.3 滑动冲突解决办法有哪些？

- 外部拦截法：指点击事件都先经过父容器的拦截处理，如果父容器需要此事件就拦截，否则就不拦截。具体方法：需要重写父容器的onInterceptTouchEvent方法，在内部做出相应的拦截。
  - 父容器的onInterceptTouchEvent方法中处理
  - ACTION_DOWN不拦截，一旦拦截会导致后续事件都直接交给父容器处理。
  - ACTION_MOVE中根据情况进行拦截，拦截：return true，不拦截：return false（外部拦截核心）
  - ACTION_UP不拦截，如果父控件拦截UP，会导致子元素接收不到UP进一步会让onClick方法无法触发。此外UP拦截也没什么用。
- 内部拦截法：指父容器不拦截任何事件，而将所有的事件都传递给子容器，如果子容器需要此事件就直接消耗，否则就交由父容器进行处理。具体方法：需要配合requestDisallowInterceptTouchEvent方法。

#### 4. onTouch()、onTouchEvent()和onClick()关系是怎样的，哪一个先执行？如果设置了onClickListener, 但是onClick()没有调用，可能产生的原因？ 

#### 4.1 onTouch()、onTouchEvent()和onClick()关系是怎样的，哪一个先执行？

>  onTouch->onTouchEvent->onClick

当一个View需要处理事件时，如果它设置了OnTouchListener，那么OnTouchListener的onTouch方法会被回调。

这时事件如何处理还得看onTouch的返回值，如果返回false，则当前View的onTouchEvent方法会被调用；如果返回true，那么onTouchEvent方法将不会被调用。由此可见，给View设置的onTouchListener，其优先级比onTouchEvent要高。

如果当前方法中设置了onClickListener，那么它的onClick方法会被调用。可以看出，常用的OnClickListener，其优先级别最低。

#### 4.2 如果设置了onClickListener, 但是onClick()没有调用，可能产生的原因？

1. 父View拦截了事件，没有传递到当前View
2. View的Enabled = false(setEnabled(false)): view处于不可用状态，会直接返回。
3. View的Clickable = false(setClickable\setLongClickable(false)):view不可以点击，不会执行onClick
4. View设置了onTouchListener，且消耗了事件。会提前返回。
5. View设置了TouchDelegate，且消耗了事件。会提前返回。

### 5. View滑动有哪些方法？这些方法分别是如何实现滑动的？分别有什么优缺点？

#### 5.1 View滑动有哪些方法？

1. layout：对View进行重新布局定位。在onTouchEvent()方法中获得控件滑动前后的偏移。然后通过layout方法重新设置。

   ![image](https://upload-images.jianshu.io/upload_images/4432347-d1d63c609f2c3c12.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

2. offsetLeftAndRight和offsetTopAndBottom:系统提供上下/左右同时偏移的API。onTouchEvent()中调用

   ![image](https://upload-images.jianshu.io/upload_images/4432347-973bba1e87a55917.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

3. LayoutParams: 更改自身布局参数

   通过父控件设置View在父控件的位置，但需要指定父布局的类型，不好 

   用ViewGroup的MariginLayoutParams的方法去设置margin

   ```java
   //方法一：通过布局设置在父控件的位置。但是必须要有父控件, 而且要指定父布局的类型，不好的方法。 
   RelativeLayout.LayoutParams layoutParams = (RelativeLayout.LayoutParams) getLayoutParams(); 
   layoutParams.leftMargin = getLeft() + offsetX; 
   layoutParams.topMargin = getTop() + offsetY; 
   setLayoutParams(layoutParams);
   /**===============================================
    * 方法二：用ViewGroup的MarginLayoutParams的方法去设置marign
    * 优点：相比于上面方法, 就不需要知道父布局的类型。
    * 缺点：滑动到右侧控件会缩小
    *===============================================*/ 
   ViewGroup.MarginLayoutParams mlayoutParams = (ViewGroup.MarginLayoutParams) getLayoutParams();
   mlayoutParams.leftMargin = getLeft() + offsetX; 
   mlayoutParams.topMargin = getTop() + offsetY; 
   setLayoutParams(mlayoutParams);
   ```

   

4. scrollTo/scrollBy: 本质是移动View的内容，需要通过父容器的该方法来滑动当前View

   都是View提供的方法。scrollTo-直接到新的x,y坐标处。scrollBy-基于当前位置的相对滑动。scrollBy-内部是调用scrollTo。scrollTo\scrollBy, 效果是移动View的内容，因此需要在View的父控件中调用。

   - scrollTo/By内部的mScrollX和mScrollY的意义
     - mScrollX的值，相当于手机屏幕相对于View左边缘向右移动的距离，手机屏幕向右移动时，mScrollX的值为正；手机屏幕向左移动(等价于View向右移动)，mScrollX的值为负。
     - mScrollY和X的情况相似，手机屏幕向下移动，mScrollY为+正值；手机屏幕向上移动，mScrollY为-负值。
     - mScrollX/Y是根据第一次滑动前的位置来获得的，例如：第一次向左滑动200(等于手机屏幕向右滑动200)，mScrollX = 200；第二次向右滑动50, mScrollX = 200 + （-50）= 150，而不是（-50）。

5. Scroller: 平滑滑动，通过重载computeScroll()，使用scrollTo/scrollBy完成滑动效果。

6. 属性动画: 动画对View进行滑动

   - 可以通过传统动画或者属性动画的方式实现
   - 传统动画需要通过设置fillAfter为true来保留动画后的状态(但是无法在动画后的位置进行点击操作，这方面还是属性动画好)
   - 属性动画会保留动画后的状态，能够点击。

7. ViewDragHelper: 谷歌提供的辅助类，用于完成各种拖拽效果。

   通过ViewDragHelper去自定义ViewGroup让其子View具有滑动效果。不过用的很少

### 6. 事件的传递规则是什么？View处理事件的优先级？点击事件传递过程遵循如下顺序？事件传递规则要点？

#### 6.1 事件的传递规则是什么？

- 点击事件产生后，会先传递给根ViewGroup，并调用dispatchTouchEvent
- 之后会通过onInterceptTouchEvent判断是否拦截该事件，如果true，则表示拦截并交给该ViewGroup的onTouchEvent方法进行处理
- 如果不拦截，则当前事件会传递给子元素，调用子元素的dispatchTouchEvent，如此反复直到事件被处理

#### 6.2 View处理事件的优先级？

- 在View需要处理事件时，会先调用OnTouchListener的onTouch方法，并判断onTouch的返回值
- 返回true，表示处理完成，不会调用onTouchEvent方法
- 返回false，表示未完成，调用onTouchEvent方法进行处理
- 可见，onTouchEvent的优先级没有OnTouchListener高
- onTouchEvent没有消耗的话就会交给TouchDelegate的onTouchEvent去处理。
- 如果最后事件都没有消耗，会在onTouchEvent中执行performClick()方法，内部会执行OnClickListener的onClick方法，优先级最低，属于事件传递尾端

#### 6.3 点击事件传递过程遵循如下顺序？

Activity->Window->View

如果View的onTouchEvent返回false，则父容器的onTouchEvent会被调用，最终可以传递到Activity的onTouchEvent

#### 6.4 事件传递规则要点？

- View一旦拦截事件，则整个事件序列都由它处理(ACTION_DOWN\UP等)，onInterceptTouchEvent不会再调用(因为默认都拦截了)
- 但是一个事件序列也可以通过特殊方法交给其他View处理(onTouchEvent)
- 如果View开始处理事件(已经拦截)，如果不消耗ACTIO_DOWN事件(onTouchEvent返回false)，则同一事件序列的剩余内容都直接交给父onTouchEvent处理
- View消耗了ACTION_DOWN，但不处理其他的事件，整个事件序列会消失(父onTouchEvent)不会调用。这些消失的点击事件最终会传给Activity处理。
- ViewGroup默认不拦截任何事件(onInterceptTouchEvent默认返回false)
- View没有onInterceptTouchEvent方法，一旦有事件传递给View，onTouchEvent就会被调用
- View的onTouchEvent默认都会消耗事件return true, 除非该View不可点击(clickable和longClickable同时为false)
- View的enable属性不影响onTouchEvent的默认返回值。即使是disable状态。
- onClick的发生前提是当前View可点击，并且收到了down和up事件
- 事件传递过程是由父到子，层层分发，可以通过requestDisallowInterceptTouchEvent让子元素干预父元素的事件分发(ACTION_DOWN除外)

### 7.  Scroller的作用？Scroller的要点有哪些？Scroller的使用步骤？Scroller工作原理？











