---
title: -updateViewConstraints与-updateConstraints
date: 2016-12-02 13:43:59
tags:
	- AutoLayout
	- Swift
	- iOS
---

## `updateViewConstraints`与`updateConstraints`简介

`updateViewConstraints`与`updateConstraints`是`AutoLayout`出现后新增的`api`，`updateConstraints`主要功能是更新`view`的约束,并会调用其所有子视图的该方法去更新约束。

<!--more-->

而`updateViewConstraints`的出现方便了`viewController`，不用专门去重写`controller`的`view`，当`view`的`updateConstraints`被调用时，该`view`若有`controller`，该`controller`的`updateViewConstraints`便会被调用。

两个方法都需要在方法实现的最后调用父类的该方法。并且这两个方法不建议直接调用。

在使用过程中我发现这两个方法有时候不会被系统调用。后来我看到`public class func requiresConstraintBasedLayout() -> Bool`方法的描述：

>constraint-based layout engages lazily when someone tries to use it (e.g., adds a constraint to a view).  If you do all of your constraint set up in -updateConstraints, you might never even receive updateConstraints if no one makes a constraint.  To fix this chicken and egg problem, override this method to return YES if your view needs the window to use constraint-based layout.  

大意是说，视图并不是主动采用`constraint-based`的。在非`constraint-based`的情况下`-updateConstraints`,可能一次都不会被调用，解决这个问题需要重写该类方法并返回true。

这里要注意，如果一个`view`或`controller`是由interface builder初始化的，那么这个实例的`updateViewConstraints`或`updateConstraints`方法便会被系统自动调用，起原因应该就是对应的`requiresConstraintBasedLayout`方法返回`true`。而纯代码初始化的视图`requiresConstraintBasedLayout`方法默认返回`false`。

所以在纯代码自定义一个`view`时，想把约束写在`updateConstraints`方法中，就一定要重写`requiresConstraintBasedLayout`方法，返回`true`。

至于纯代码写的`viewController`如何让其`updateViewConstraints`方法被调用。我自己的解决办法是手动调用其`view`的`setNeedsUpdateConstraints`方法。



## How to use updateConstraints?



文档中对于这两个方法提的最多的就是，重写这两个方法，在里面设置约束。所以一开始我认为这两个方法是苹果提供给我们专门写约束的。于是便开始尝试使用。

直到后来在`UIView`中看到这样一句话：
>You should only override this method when changing constraints in place is too slow, or when a view is producing a number of redundant changes.

“你只因该在添加约束过于慢的时候，或者一次要修改大量约束的情况下重写次方法。”

简直是让人觉得又迷茫又坑爹。`updateConstraints`方法到底应该何时使用

后来看到[how to use updateConstraints](http://oleb.net/blog/2015/08/how-to-use-updateconstraints/)这篇文章。给出了一个合理的解释：

* 尽量将约束的添加写到类似于viewDidLoad的方法中。

* `updateConstraints`并不应该用来给视图添加约束，它更适合用于周期性地更新视图的约束，或者在添加约束过于消耗性能的情况下将约束写到该方法中。

* 当我们在响应事件时（例如点击按钮时）对约束的修改如果写到`updateConstraints`中，会让代码的可读性非常差。

关于性能，我也做了一个简单的测试：

```swift
class MMView: UIView {
    override init(frame: CGRect) {
        super.init(frame: frame)
        self.backgroundColor = UIColor.grayColor()
        initManyButton()
        //初始化时添加约束
        test() //每次只有一个test()不被注释就好
    }
    
    override func touchesBegan(touches: Set<UITouch>, withEvent event: UIEvent?) {
        //响应事件时添加约束
        //test()
    }
    
    override func updateConstraints() {
        //updateConstraints中添加约束
        //test()
        super.updateConstraints()
    }
    
    func test(){
        let then = CFAbsoluteTimeGetCurrent()
        addConstraintsToButton()
        let now = CFAbsoluteTimeGetCurrent()
        print(now - then)
    }
    
    required init?(coder aDecoder: NSCoder) {
        fatalError("init(coder:) has not been implemented")
    }
    
    let buttonTag = 200
    func initManyButton(){
        for index in 0...1000{
            let button = UIButton(type: .System)
            button.tag = buttonTag + index
            self.addSubview(button)
        }
    }
    func addConstraintsToButton(){
        for index in 0...1000{
            if let button = self.viewWithTag(index+buttonTag){
                button.snp_makeConstraints{ make in
                    make.center.equalTo(self)
                    make.size.equalTo(self)
                }
            }
        }
    }
}
```
分别对 将 设置约束 写在`init`中、写在`updateConstraints`中、写在事件响应方法中 的时间消耗进行测试，对1000个button添加约束，每个添加4个约束。

* `init`中,时间消耗约为0.37秒
* 写在`updateconstraints`中，时间消耗约为0.52秒
* 写在事件响应方法中，时间消耗约为0.77秒

**所以，结论，还是将约束的设置写在`viewDidLoad`中或者`init`中。没事儿尽量不去碰`updateConstraints`。除非对性能有要求。**

## 关于UIView的translatesAutoresizingMaskIntoConstraints属性



最近在对`AutoLayout`的学习中发现，很多人似乎对`translatesAutoresizingMaskIntoConstraints`的误解非常大，很多时候遇到问题总有人会在下面回答到：把`translatesAutoresizingMaskIntoConstraints`设置成`false`就可以解决问题。。。实际上并没有什么用。

那么这个属性到底是做什么的呢？

其实这个属性的命名已经把这个属性的功能解释的非常清楚了。

除了`AutoLayout`，`AutoresizingMask`也是一种布局方式。这个想必大家都有了解。默认情况下，`translatesAutoresizingMaskIntoConstraints ＝ true` , 此时视图的`AutoresizingMask`会被转换成对应效果的约束。这样很可能就会和我们手动添加的其它约束有冲突。此属性设置成`false`时，`AutoresizingMask`就不会变成约束。也就是说 `当前` 视图的 `AutoresizingMask`失效了。

**那我们什么时候需要设置这个属性呢？**

**当我们用代码添加视图时**,视图的`translatesAutoresizingMaskIntoConstraints`属性默认为`true`，可是`AutoresizingMask`属性默认会被设置成`.None`。也就是说如果我们不去动`AutoresizingMask`，那么`AutoresizingMask`就不会对约束产生影响。

**当我们使用interface builder添加视图时**，`AutoresizingMask`虽然会被设置成`非.None`，但是`translatesAutoresizingMaskIntoConstraints`默认被设置成了`false`。所以也不会有冲突。

反而有的视图是靠`AutoresizingMask`布局的，当我们修改了`translatesAutoresizingMaskIntoConstraints`后会让视图失去约束，走投无路。例如我自定义转场时就遇到了这样的问题，转场后的视图并不在视图的正中间。

**所以，这个属性，基本上我们也不用设置它。**