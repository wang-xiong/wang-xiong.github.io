---
layout: post
title: "[View系列] 六、ConstraintLayout使用"
subtitle: '学习View相关知识'
date:       2018-08-06
author: "Wangxiong"
header-style: text
tags:
  - Android
  - View
---
## 1. ConstraintLayout简介

ConstraintLayout（约束布局）是Android新推出的布局，AndroidStudio默认使用此布局，ConstraintLayout能够减少布局的层级并改善布局性能，ConstraintLayout能够灵活定位和调整View的大小，子View依靠约束关系确定位置。

## 2. ConstraintLayout基本属性

ConstraintLayout的基本属性主要是layout_constraintXXX_toYYYOf的格式属性 ，表示ViewA的XXX方向置于ViewB的YYY方向，如果ViewB是父容器即ConstraintLayout，可以用parent表示。

- layout_constraintBaseline_toBaselineOf （View A 内部文字与 View B 内部文字对齐）
- layout_constraintLeft_toLeftOf （View A 与 View B 左对齐）
- layout_constraintLeft_toRightOf （View A 的左边置于 View B 的右边）
- layout_constraintRight_toLeftOf （View A 的右边置于 View B 的左边）
- layout_constraintRight_toRightOf
- layout_constraintTop_toTopOf
- layout_constraintTop_toBottomOf
- layout_constraintBottom_toTopOf
- layout_constraintBottom_toBottomOf
- layout_constraintStart_toEndOf
- layout_constraintStart_toStartOf
- layout_constraintEnd_toStartOf
- layout_constraintEnd_toEndOf

## 2. 约束力的强度

基本属性为控件添加了方向上的约束力，根据某个方向上的有无或者强弱，确定控件的位置。约束的强度力依靠layout_constraintHorizontal_bias和layout_constrantVertical_bias两个属性，即设置控件在水平和垂直方向的百分比的偏移量，如0.1表示百分十。

## 3. Visibility属性

在ConstraintLayout布局中，如果View的Visibility属性设置为gone，其他针对该View的约束力仍然生效，相当于该View无线缩小为点。

## 4. 控件的宽高比

ConstraintLayout布局可以为控件设置固定的宽高比，利用layout_constraintDimensionRatio属性进行设置，使用layout_constraintDimensionRatio属性必须设置宽度或者高度为0dp，宽高的通过"float"指或者""宽度：高度""的方式设置。如果宽度和高度都是0dp，系统会满足所以约束条件和宽高比率的最大值。如果要根据一个尺寸来约束另一个尺寸，则可以在比率值前添加W或者H来指明约束宽度或者高度，W代表用宽度约束高度的尺寸。

## 5. 控件与控件的宽或者高比例

LinearLayout布局通过为控件设置layout_weight属性来控制控件与控件之间的比例，ConstraintLayout布局可以使用app:layout_constraintHorizontal_weight属性为控件设置比例。同时使用属性layout_constraintVertical_chainStyle和app:layout_constraintHorizontal_chainStyle分别指明约束的方式是垂直和水平方向。

## 6. GuildLine锚向指示线

当需要一个任意位置时，可以使用指示线来帮助定位，GuildLine就是一个VIew的子类，宽度和高度都为0，可见性为View.GONE。指示线主要是帮助其他控件进行约束布局确定位置的。GuildLine的orientation属性表示了指示线的方向，layout_constraintGuide_percent属性来设置指示线的位置，可以是具体的值，也可以是float类型的百分比。

## 7.  Chains链

在ConstraintLayout布局中，可以未多个通过Chains链连接的View来分发剩余的位置。如layout_constraintHorizontal_weight就是一个种链，链条分为水平和竖直链条。分别用

layout_constraintHorizontal_chainStyle属性和layout_constraintVertical_chainStyle表示，属性的值有三种spread、spread_inside、packed。

## 8. 圆形定位约束

在ConstraintLayout的1.1.2新增了圆形定位约束，使用属性app:layout_constraintCircle指明约束定位的View控件，app:layout_constraintCircleAngle设置对齐的角度，顺时针方向0-360度，app:layout_constraintCircleRadius与定位控件的距离。

## 9. 强制约束Enforcing constraints

在1.1版本之前，如果控件设置为wrap_content，那么对控件设置的maxWidth、minHeight是不起作用的，而强制约束就是使控件在设置wrap_content时生效，强制约束的属性是app:layout_constrainedHeight和app:layout_constrainedWidth，属性的值为true和false。

## 10. 控件占父控件的百分比

当控件的高度或者宽度尺寸设置为0时，可以使用layout_constrainWidth_percent或layout_constrainHeight_percent设置当前控件宽度或高度占父控件宽高的百分比。

## 11. Barrier

多个控件之间的相互约束，可以使用Barrier进行组合，Barrier的属性有

- barrierDirection：用于设置 Barrier 的位置，属性值有：bottom、top、start、end、left、right。
- constraint_referenced_ids：用于设置 Barrier 所引用的控件的 ID，可同时设置多个
- barrierAllowsGoneWidgets：默认为 true，当 Barrier 所引用的控件为 Gone 时，则 Barrier 的创建行为是在已 Gone 的控件已解析的位置上进行创建。如果设置为 false，则不会将 Gone 的控件考虑在内

## 12. Group

使用Group控件，可以同时设置多个控件的可见性。其属性constraint_referenced_ids是用来设置所引用的控件Id。

## 13. Placeholder

Placeholder站位符控件，可以使用Placeholder的setContentId() 方法将占位符控件转换为指定的视图，即占位符所在的位置上显示，如果此时布局中已经包含该视图，则将视图从布局中的原位置消失，在占位符位置显示，也可以使用属性app:content，直接在布局中将占位符和视图Id绑定在一起。