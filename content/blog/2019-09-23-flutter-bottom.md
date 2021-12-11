---
title: 底部滑動按鈕
date: 2019-09-23
description: "flutter 實作底部滑動按鈕"
tags: [flutter, dart]
draft: false
---

# 想法
- 使用 `button`
- 用 `row` 集成 `button` 為同一列   
- `SingleChildScrollView` 滑動
- `EdgeInsets` 設定 `button` 之間距離
- 用 `container` 包住元件


## 程式碼
```dart
import 'package:flutter/material.dart';
import 'package:flutter/painting.dart';
import 'package:flutter/widgets.dart';

/**
 * 定義資料結構，button 要有名稱和動作
 */
class BottomButtonModel {
  String title;
  String routeName;
  BottomButtonModel(String title, String routeName) {
    this.title = title;
    this.routeName = routeName;
  }
}

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
        color: Colors.grey, // 這邊可以當背景顏色，這邊使用 container 將我定義的 row 集成 button 包在裡面
        child: new SingleChildScrollView( // 設定 scroll 讓元素能夠滑動
          scrollDirection: Axis.horizontal, // 以水平方式移動
          child: new Row(  // 用 row 實現，將每個 button 集成為一列
            verticalDirection: VerticalDirection.down, // 向下對齊
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
 * 定義 button 外型及動作
 */
class BottomButton extends StatefulWidget {
  final String title;
  // final Function opPress;
  final String routeName;
  // required 表示必要參數
  BottomButton({@required this.title, this.routeName});

  @override
  _BottomButtonState createState() => _BottomButtonState();
}

class _BottomButtonState extends State<BottomButton> {
  @override
  Widget build(BuildContext context) {
    // TODO: implement build
    return new Container(
      margin: EdgeInsets.only(left: 5.0, right: 5.0), // 左右各 5 元素
      child: new RaisedButton( 
        padding: new EdgeInsets.fromLTRB(10.0, 0.0, 10.0, 0.0), // 定義邊距
        clipBehavior: Clip.antiAlias, // 點擊時觸發的顏色
        colorBrightness: Brightness.light, 
        textColor: Colors.black,
        color: Colors.white,
        splashColor: Colors.yellow,
        shape: new RoundedRectangleBorder( // 把按鈕切成圓弧形
            borderRadius: new BorderRadius.circular(30.0)), // circular 切的弧度
        child: new Text(
          widget.title, // 取得按鈕名稱
        ),
        onPressed: () { // 點擊時觸發的動作，這邊定義用 route 切換不同頁面
          // widget.opPress;
          Navigator.pushNamed(context, '/' + widget.routeName,
              arguments: <String, String>{'title': widget.title});
        },
      ),
    );
  }
}

```

## result

[嚴禁抄襲](https://i.imgur.com/2uMvPd8.gif)
<!-- {{< figure src="https://i.imgur.com/2uMvPd8.gif">}} -->

## Ref
[RaisedButton](https://api.flutter.dev/flutter/material/RaisedButton-class.html)

[button 路由](https://cch0124.github.io/flutter-button-route/)