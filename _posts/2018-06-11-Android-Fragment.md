## 1. Fragment简介

Fragment(碎片)是一种可以嵌入在活动当中的UI片段，能够让程序更加合理和充分地利用大屏幕空间，和活动类似同样包含布局、同样有自己的生命周期。

## 2. Fragment的生命周期

onAttach、onCreate、onCreateView、onActivityCreated、onStart、onResume、onPause onStop、onDestroyView、onDestroy、onDetach。

## 3. Fragment的数据传递

- 通过setArgument方法传递数据。
- 在Fragment中直接调用Activity的方法
- 在Fragment申明接口，Activity实现接口（Fragment的onAttach方法中得到的Activity就是实现接口的实例）
- 使用ViewModel共享统一个Activity下Fragment数据。