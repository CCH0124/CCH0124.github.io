---
title: flutter Button route 應用
date: 2019-09-01
description: "bottom button "
tags: [flutter, route]
draft: false
---


## 目標
在手機底部做一個能夠左右滑動的多個按鈕集合。希望按下第一個按鈕能切換至他的頁面；第二個按鈕能切換至他的頁面以此類推

## flutter 路由方式
1. Route
route 是應用程式的`螢幕`或`頁面`的抽象，`Navigator` 是管理 `route` 的 `widget`
2. Navigator
建立維護由 stack 的歷史記錄的 Widget。`Navigator` 可以 push 和 pop 出 route，幫助使用者從一個螢幕畫面移動到另一個屏幕畫面
3. Material Page Route
一種樣式 route，用 `platform-adaptive transition` 替換整個螢幕畫面

### 什麼是 platform-adaptive transition？

從一個螢幕畫面路由到其他螢幕換面時看到的轉換。
- Android
  - 頁面的入口轉換將頁面向上滑動並淡入。退出的轉換是相同的，但滑動是相反
- iOS
  - 頁面從右側滑入並反向退出。當另一頁進入以覆蓋它時，頁面也會在視覺中向左移動

此轉換僅因 `MaterialPageRoute` 而發生。要修改此轉換動畫，需使用 `MaterialPageRoute` 或 `PageRouteBuilder`

## flutter Code

```dart
import 'package:flutter/material.dart';
import 'package:flutter/painting.dart';
import 'package:flutter/widgets.dart';
/**
 * 定義一個按鈕的結構
 * title 該按鈕名稱
 * routeName 該按鈕路由名稱
 */
class BottomButtonModel {
  String title;
  String routeName;
  /// 建構方法(Constructor)
  BottomButtonModel(String title, String routeName) {
    this.title = title;
    this.routeName = routeName;
  }
}

/**
 * 用 List 資料結構放入按鈕資料結構
 */
List<BottomButtonModel> bottomButtonItems = [
  BottomButtonModel("F1", "F1"),
  BottomButtonModel("F2", "F2"),
  BottomButtonModel("F3", "F3"),
  BottomButtonModel("F4", "F4"),
  BottomButtonModel("F5", "F5"),
  BottomButtonModel("F6", "F6"),
  BottomButtonModel("F7", "F7"),
  BottomButtonModel("F8", "F8"),
  BottomButtonModel("F9", "F9"),
  BottomButtonModel("F0", "F0"),
  BottomButtonModel("F.", "Fdot"),
];

/**
 * 建立底部按鈕滑動功能
 */
class BottomButtonListBuild extends StatefulWidget {
  @override
  BottomButtonListBuildState createState() {
    // TODO: implement createState
    return new BottomButtonListBuildState();
  }
}

class BottomButtonListBuildState extends State<BottomButtonListBuild> {
  @override
  Widget build(BuildContext context) {
    // TODO: implement build
    return new Material(
      child: new Container(
        color: Colors.grey,
        child: new SingleChildScrollView( /// 滑動的 Widget
          scrollDirection: Axis.horizontal, /// 設定橫向
          child: new Row( /// 用 Row Widget 布局，將 Button 由左至右，非由上而下
            verticalDirection: VerticalDirection.down,
            children: bottomButtonItems
                .map((f) => BottomButton(
                      title: f.title,
                      routeName: f.routeName,
                    ))
                .toList(),
          ),
        ),
      ),
    );
  }
}

/**
 * 建構 Button
 */ 
class BottomButton extends StatefulWidget {
  final String title;
  // final Function opPress;
  final String routeName;
  BottomButton({@required this.title, @required this.routeName});

  @override
  _BottomButtonState createState() => _BottomButtonState();
}

class _BottomButtonState extends State<BottomButton> {
  @override
  Widget build(BuildContext context) {
    // TODO: implement build
    return new Container(
      margin: EdgeInsets.only(left: 5.0, right: 5.0), /// 邊距
      child: new RaisedButton(
        padding: new EdgeInsets.fromLTRB(10.0, 0.0, 10.0, 0.0), /// 該 button 的至 container 邊距
        clipBehavior: Clip.antiAlias, /// 將該按鈕圓弧化
        colorBrightness: Brightness.light, /// 亮度屬性
        textColor: Colors.black,
        color: Colors.white,
        splashColor: Colors.yellow,
        shape: new RoundedRectangleBorder(
            borderRadius: new BorderRadius.circular(30.0)),
        child: new Text(
          widget.title,
        ),
        onPressed: () {
          // widget.opPress;
          /**
           * 路由部分
           * arguments 用來傳遞參數用
           */
          Navigator.pushNamed(context, '/' + widget.routeName,
              arguments: <String, String>{'title': widget.title});
        },
      ),
    );
  }
}

```

## 建立 Page 測試
```dart
class F1Page extends StatefulWidget {
  @override
  _F1PageState createState() => _F1PageState();
}

class _F1PageState extends State<F1Page> {
  @override
  Widget build(BuildContext context) {
    // TODO: implement build
    return Scaffold(
      appBar: AppBar(
        elevation: 0.0,
        backgroundColor: drawerBackgroundColor,
        title: Text("F1"),
      ),
      drawer: new NavDrawer(), /// 自訂義的，用來測試傳值，可將此拿掉
      body: new Column(
        children: <Widget>[
          new Text("F1"),
        ],
      )
      bottomNavigationBar: BottomButtonListBuild(),
    );
  }
}
```

```dart
class F2Page extends StatefulWidget {
  @override
  _F2PageState createState() => _F2PageState();
}

class _F2PageState extends State<F2Page> {
  @override
  Widget build(BuildContext context) {
    // TODO: implement build
    return Scaffold(
      appBar: AppBar(
        elevation: 0.0,
        backgroundColor: drawerBackgroundColor,
        title: Text("F2"),
      ),
      drawer: new NavDrawer(),
      body: new Column(
        children: <Widget>[
          new Text("F2"),
        ],
      ),
      bottomNavigationBar: BottomButtonListBuild(),
    );
  }
}
```

```dart
class F3Page extends StatefulWidget {
  @override
  _F3PageState createState() => _F3PageState();
}

class _F3PageState extends State<F3Page> {
  @override
  Widget build(BuildContext context) {
    // TODO: implement build
    return Scaffold(
      appBar: AppBar(
        elevation: 0.0,
        backgroundColor: drawerBackgroundColor,
        title: Text("F3"),
      ),
      body: new Column(
        children: <Widget>[
          new Text("F3")
        ],
      )
      ,
      bottomNavigationBar: BottomButtonListBuild(),
    );
  }
}
```

## Named Routing

本篇使用的是 `Named Routing`。使用 `Navigator.pushNamed` 路由到螢幕新頁面並使用 `Navigator.pop` 返回。`Navigator` 維護基於 stack 的 route 歷史記錄。`Navigator.pushNamed` 需要兩個必需的參數（`BuildContext`,`String`,`{Object}`）和一個可選的 `Argument`(用來傳遞資訊至螢幕的新頁面)。在 `String` 的位置，傳遞在路由中預定義的 `String`，就是定義該路徑的路徑名稱。


如果在的路線 `'/'` 中，不必定義初始 route 位置。它將 `'/'` 作為初始路線。如果希望你的初始路線不同於 `'/'`，那麼可以如下定義。預設將 F1Page 打開。

```dart
routes: <String, WidgetBuilder>{
        '/': (context) => F1Page(),
        '/F1': (context) => F1Page(),
        '/F2': (context) => F2Page(),
        '/F3': (context) => F3Page(),
      },
```

初始 route 和 F1 相同，可以在任何地方使用任何位置(路徑)。需要注意，如果是初始 route，那麼已經定義了 `route` 或 `onGenerateRoute`，route 中應該有 `/`。
```dart
routes: <String, WidgetBuilder>{
        '/F1': (context) => F1Page(),
        '/F2': (context) => F2Page(),
        '/F3': (context) => F3Page(),
      },
```

如果想要在任何 route 不匹配時它應該顯示 404 錯誤螢幕頁面，那麼可以在 `UnknownRoute` 上使用。但因為我做的沒有此問題，因此不配置。

## 主程式

```dart
//MyApp()
void main() => runApp(new MaterialApp(
      initialRoute: '/F1',
      routes: <String, WidgetBuilder>{
        '/': (context) => F1Page(),
        '/F1': (context) => F1Page(),
        '/F2': (context) => F2Page(),
        '/F3': (context) => F3Page(),
      },
    ));

class MyApp extends StatelessWidget {
  // This widget is the root of your application.
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Flutter Demo',
      theme: ThemeData(
        primarySwatch: Colors.blue,
      ),
      home: F1Page(),
    );
  }
}
```

## onGenerateRoute 方式
為給定的 route 設置創建 route。

```dart
onGenerateRoute: (RouteSettings settings) {
        switch (settings.name) {
          case '/':
            return MaterialPageRoute(builder: (context) => F1Page());
            break;
          case '/F1':
            return MaterialPageRoute(builder: (context) => F1Page());
            break;
          case '/F2':
            return MaterialPageRoute(builder: (context) => F2Page());
            break;
          case '/F3':
            return MaterialPageRoute(builder: (context) => F3Page());
            break;
        }
      },
```

## 畫面
主要這邊說明的是 route 部分。

[結果鏈接](https://i.imgur.com/KaHV2RM.gif)

## ref

[routing 方式](https://blog.usejournal.com/flutter-advance-routing-and-navigator-df0f86f0974f)