---
title: navigation drawer
date: 2019-11-08
description: "使用 ListTile 與 ListView 完成"
tags: [flutter]
draft: false
---

## 目標
實現抽屜樣式的下拉式選單，或許 `row` 和 `column` 也能夠實現，但是會比較麻煩。這次實現步驟是透過 `ListTile` 和 `ListView` 實現。

## ListTle 和 ListView
### ListTile
官方說 `A single fixed-height row that typically contains some text as well as a leading or trailing icon.` 是一個高度固定，但單一個列，通常包含一些文字和前導或尾隨的圖示。

原始碼
```dart
class ListTile extends StatelessWidget {
  /// Creates a list tile.
  ///
  /// If [isThreeLine] is true, then [subtitle] must not be null.
  ///
  /// Requires one of its ancestors to be a [Material] widget.
  const ListTile({
    Key key,
    this.leading, /// 前導，通常都用 Icon Widget
    this.title, /// 標題，通常都用 Text widget
    this.subtitle, /// 副標題
    this.trailing, /// 尾隨(末端)，通常都用 Icon Widget
    this.isThreeLine = false, /// 是否顯示三行的文字
    this.dense, /// 把 ListTitle 高度變小
    this.contentPadding, /// 調整邊距，預設是 `EdgeInsets.symmetric(horizontal: 16.0)`
    this.enabled = true, /// 點擊動作，預設為可點擊
    this.onTap, /// 點擊後的行為
    this.onLongPress, /// 長按
    this.selected = false, /// 選擇列表時，該列表被選重時會以 `primary color` 作為顏色，`ListTileTheme` 可覆寫
  }) : assert(isThreeLine != null),
       assert(enabled != null),
       assert(selected != null),
       assert(!isThreeLine || subtitle != null),
       super(key: key);
```
>`isThreeLine` 為 `true` 時，`subtitle` 不能為 `null`，因為需要第二行或以上

官方示意圖
<!-- ![](../assets/img/flutter/flutter-listtile.png) -->
{{< figure src="/images/flutter/flutter-listtile.png" width="auto" height="auto">}}

### ListView
讓 widget 能夠滾動列表。

官方說明，構造 ListView 有四個選項：
1. 默認構造函數採用 `children` 顯式 `List <Widget>`。適用於少量的 `children` 列表。
2. `ListView.builder` 構造函數採用 `IndexedWidgetBuilder`，它建立在 `children` 的需求。適用於大量 `children` 或無限多個。
3. `ListView.separated` 構造函數有兩個 `IndexedWidgetBuilders:itemBuilder` 按需求建立多個子項目，`separatorBuilder` 建立在出現在子項之間的分隔符的 `children`。適用於具有固定數量的 `children`
4. `ListView.custom` 構造需要 `SliverChildDelegate`，它提供了自定義 `child` 模型的其他方面的能力。

原始碼，這邊從官網查看參數即可得知定義。
```dart
ListView({
  ...  
  /// 可滾動 widget 共同參數
  Axis scrollDirection = Axis.vertical,
  bool reverse = false,
  ScrollController controller,
  bool primary,
  ScrollPhysics physics,
  EdgeInsetsGeometry padding,

  ///各個構造函數共同參數
  double itemExtent,
  bool shrinkWrap = false,
  bool addAutomaticKeepAlives = true,
  bool addRepaintBoundaries = true,
  double cacheExtent,

  /// 子 widget 列表
  List<Widget> children = const <Widget>[],
})
```

## 實作
下面是部分程式碼，無法完整運行。需要自行定義 `List<NavigationModel> navigationItem`。這邊付上在 `Codelabs` 的完整實作[鏈接](https://github.com/CCH0124/flutter-codelabs/tree/master/first_flutter)。

```dart
import 'package:flutter/cupertino.dart';
import 'package:flutter/material.dart';

import 'navigation_model.dart';


class NavDrawer extends StatefulWidget {
  List<NavigationModel> navigationItem;
  NavDrawer({Key key, @required this.navigationItem }): super(key:key);
  
  @override
  NavDrawerState createState() => NavDrawerState();
}

class NavDrawerState extends State<NavDrawer> {
  @override
  Widget build(BuildContext context) {
    Map<String, String> routeValue = ModalRoute.of(context).settings.arguments; // 不同路由傳值接值
    // TODO: implement build
    return new Material(
      child: new Container(
        width: 200,
        child: new Drawer(
          child: new Column(
            children: <Widget>[
              new NavListTitle(
                title: (routeValue == null) ? "F1" : routeValue['title'],
                icon: Icons.person,
              ),
              new SizedBox(
                width: 50,
              ),
              new Divider(
                color: Colors.grey,
                height: 10.0,
              ),
              new Expanded(
                child: new ListView.separated( /// child 之間的分割符
                  separatorBuilder: (context, index) {
                    return new Divider(
                      color: Colors.grey,
                      height: 10.0,
                    );
                  },
                  itemBuilder: (context, index) {
                    return new NavListTitle(
                      title: widget.navigationItem[index].title,
                      icon: widget.navigationItem[index].icon,
                      onTap: () {
                        setState(() {
                          return null;
                        });
                      },
                    );
                  },
                  itemCount: widget.navigationItem.length,
                ),
              ),
            ],
          ),
        ),
      ),
    );
  }
}

class NavListTitle extends StatefulWidget {
  final String title;
  final IconData icon;
  final Function onTap;
  NavListTitle({@required this.title, @required this.icon, this.onTap});

  @override
  _NavListTitleState createState() => _NavListTitleState();
}

class _NavListTitleState extends State<NavListTitle> {
  @override
  Widget build(BuildContext context) {
    // TODO: implement build
    return new ListTile(
      leading: Icon( 
        widget.icon,
        size: 20,
      ),
      title: new Text(
        widget.title,
        style: TextStyle(fontSize: 15),
      ),
      onTap: widget.onTap,
    );
  }
}
```

## 參考資料
- [ListTile (Flutter Widget of the Week)](https://www.youtube.com/watch?v=l8dj0yPBvgQ)
- [flutter-dev-listtile](https://api.flutter.dev/flutter/material/ListTile-class.html)
- [ListView (Flutter Widget of the Week)](https://www.youtube.com/watch?v=KJpkjHGiI5A)
- [flutter-dev-listview](https://api.flutter.dev/flutter/widgets/ListView-class.html)