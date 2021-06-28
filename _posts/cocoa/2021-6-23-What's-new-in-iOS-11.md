---
title: What's new in iOS 11
author: strayRed
date: 2021-6-23 10:26:00 +0800
categories: [ios, cocoa]
tags: [ios, cocoa]
---
# Navigation
## Large Title

```Swift
navigationBar.prefersLargeTitles = true
//Automatic：与上一个item相同
//Never：不使用Large Title
//Always：使用Large Title
navigationItem.largeTitleDisplayMode
```

## UISearchController

```Swift
//设置searchController实例，关联当前navigationItem
navigationItem.searchController
//Bool，ScrollView滑动时是否隐藏searchBar
navigationItem.hidesSearchBarWhenScrolling
```

## UIToolbar and UINavigationBar

`UIToolbar` 和 `UINavigationBar`内部的View已支持auto layout。

UINavigationBar and UIToolbar provide position You must provide size 

- Constraints for width and height 
- Implement intrinsicContentSize 
- Connect your subviews via constraints

# Margins and Insets

- 添加了新的`directionalLayoutMargins`，并且可以与`layoutMargins`相互替换。

  `.left` = `.leading`，`.right` = `.trailing`

- 通过设置`UIViewController `的`systemMinimumLayoutMargins`属性，可以设置其`View`默认的最小`NSDirectionalEdgeInsets`。

  此外，设置`viewRespectsSystemMinimumLayoutMargins`为`flase`，上述的属性将不再起作用，即systemLayoutMargins会变成(0, 0, 0, 0)
  
- `UIViewController`的`topLayoutGuide`与`bottomLayoutGuide`现在以及被替换为`UIView.safeAreaInsets`。

- 通过使用`UIViewController.additionalSafeAreaInsets `，可以附加一个开发者自定义的额外Insets，此外，如果我们不需要某一一视图的内容显示受到`SafeArea`的影响，即不想让其`LayoutMargins`附加上`SafeAreaLayoutMargins`，那么可以设置`UIView.insetsLayoutMarginsFromSafeArea`为`fasle`即可。

# Scroll Views

- 新属性`UIScrollView.adjustedContentInset`用于决定`SafeArea`或者``SystemLayoutMargins`与 scroll view content的insets。当`contentInsetAdjustmentBehavior`为`Never`时，这个值不会加上`SafeArea`。（iOS 11之前是直接使用`contentInsets`来实现的）。
- 在`scrollView`内部，也添加了新的用于自动布局的LayoutGuide对象，`UIScrollView.frameLayoutGuide`用于布局固定的子视图，`UIScrollView.contentLayoutGuide`则用于布局需要随着滚动而变化的视图。

# Table Views

在iOS11以前，`UITableView.separatorInset`是相对于从tableView的 内部的layout margins而言的，这个值在横竖屏幕的情况下会发生变化，因为这个值相当于 readable content guide的 insets， 所以无论如何调整`separatorInset`，都不能扩大tableView的可读区域。

在iOS11之后，`UITableView.separatorInset`默认会和Cell的inset相关，即tableViewCell的宽度。通过这种方式设置left和right为（0，0）则可以将Cell底部的`separator`拉满。

通过设置`UITableView.separatorInsetReference`为`fromAutomaticInsets`与`fromCellEdges`（default），可以在上述两种类型中切换。

使用`fromAutomaticInsets`所依赖的tableView的内部的layout margins可能会随着`SafeArea`发生变化（因为其要保证readable content不会被遮挡）所以在实际开发中，我们需要使用`UITableViewCell`和`UITableViewHeaderFooterView`并在其`contentView`中添加子视图，因为`contentView`相对于`view`会默认加上`SafeArea`的insets。