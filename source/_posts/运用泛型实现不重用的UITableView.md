---
title: 运用泛型实现不重用的UITableView
date: 2016-12-01 18:32:44
tags:
	- iOS
	- Swift
	- 泛型
	- UITableView
---
`tableView`再常见不过了，现在的项目中基本上都会用到很多`tableView`。并且很多时候`tableView`上每一行的内容都不同。

如果你有这样的需求：
>一个展现用户信息的页面，有的`cell`最右侧是图片，有的`cell`最右侧显示的是文本（名字、手机号、性别、余额）

Or:

>一个填写用户信息的列表，有各种各样的`textField`

<!--more-->

上述的两种页面有两个共同的特点：

* `tableViewCell`的数量有限，并且数量不大。不需要重用`cell`也能搞定。

* 比起写出多个`cell`子类去适应这些情况，不如把这些`label`或者`textfield`作为`viewControler`的熟悉，在`tableView(tableView: UITableView, cellForRowAtIndexPath indexPath: NSIndexPath)`代理方法中把这些特定控件加到`cell`上，修改或者获取这些控件时非常方便。

然而这个时候`tableView的cell`重用机制就非常棘手：

* 注册多种`cell`很麻烦，在这种情况下很多余。

* 很多人应该遇到过的情况，重用`cell`会让视图变得很混乱，一些图片或空间因为重用的出现在了不该出现的地方


在`storyBoard`中可以设置`static cell`，来关闭重用。可是如果`tableView`是用代码建立的，就没有某个系统库的方法能够设置`static cell`。

于是在`swift`下我写了一个简单的`extension`可以实现关闭重用的效果。实现原理也非常简单，**show code**:

```swift	
extension UITableView {
	
    /*
     弹出一个静态的cell，无须注册重用，例如:
     let cell: GrayLineTableViewCell = tableView.mm_dequeueStaticCell(indexPath)
     即可返回一个类型为GrayLineTableViewCell的对象
     
     - parameter indexPath: cell对应的indexPath
     - returns: 该indexPath对应的cell
     */
    func mm_dequeueStaticCell<T: UITableViewCell>(indexPath: NSIndexPath) -> T {
        let reuseIdentifier = "staticCellReuseIdentifier - \(indexPath.description)"
        if let cell = self.dequeueReusableCellWithIdentifier(reuseIdentifier) as? T {
            return cell
        }else {
            let cell = T(style: .Default, reuseIdentifier: reuseIdentifier)
            return cell
        }
    }
}
```


无须注册。

将`cell`直接声明为其需要的类型，改方法会自动返回这个类型的`cell`。

最后：

泛型函数的调用必须是以下写法：

```swift	
let cell: GrayLineTableViewCell = tableView.mm_dequeueStaticCell(indexPath)
```	

如果写成：

```swift	
let cell = tableView.mm_dequeueStaticCell<GrayLineTableViewCell>(indexPath)
```

将会报错，这种写法只适用于 泛型类型，不适用于 泛型函数