---
title: 資料表實現
date: 2019-09-20
description: "flutter DataTable 使用"
tags: [flutter, DataTable]
draft: false
---

# 目標
- 頁面顯示資料表使用 `DataTable`
- 配合 `scroll` 使資料表能夠上下左右滑動

以下會先以描述 `DataTable` Widget 在描述實作過程。

## what is DataTable
`DataTable` 可實現數據的操作（排序、選擇等）和顯示。但這篇文章只有說明到顯示部分。但此 `DataTable` 並無做到表頭的固定，覺得可惜。少了表頭固定當滑動資料表時會無法對應該欄位是在針對什麼描述。

要組成一個 [`DataTable`](https://api.flutter.dev/flutter/material/DataTable-class.html) 可需要 [`DataColumn`](https://api.flutter.dev/flutter/material/DataColumn-class.html)、[`DataRow`](https://api.flutter.dev/flutter/material/DataRow-class.html) 和 [`DataCell`](https://api.flutter.dev/flutter/material/DataCell-class.html) 組件做配置。

而在 `DataTable` [原始碼](https://github.com/flutter/flutter/blob/master/packages/flutter/lib/src/material/data_table.dart) 中，可以知道他可以針對表頭高度、欄位間格等做設定。在原始碼中已給預設值，但可依照環境要求進行設定。

```dart
DataTable({
    Key key,
    @required this.columns,
    this.sortColumnIndex,
    this.sortAscending = true,
    this.onSelectAll,
    this.dataRowHeight = kMinInteractiveDimension,
    this.headingRowHeight = 56.0,
    this.horizontalMargin = 24.0,
    this.columnSpacing = 56.0,
    @required this.rows,
  }) : assert(columns != null),
       assert(columns.isNotEmpty),
       assert(sortColumnIndex == null || (sortColumnIndex >= 0 && sortColumnIndex < columns.length)),
       assert(sortAscending != null),
       assert(dataRowHeight != null),
       assert(headingRowHeight != null),
       assert(horizontalMargin != null),
       assert(columnSpacing != null),
       assert(rows != null),
       assert(!rows.any((DataRow row) => row.cells.length != columns.length)),
       _onlyTextColumn = _initOnlyTextColumn(columns),
       super(key: key);
```



>其中官方明了"如果數據非常大量可以使用 `PaginatedDataTable` 做分頁"


## Implementation

## Defined User Data
定義一個 `user` 資料。
```dart
class User {
  final String firstName;
  final String lastName;
  final int age;
  final String gender;
  final double height;
  final double weight;
  final String sport;
  final String phoneBrands;

  User({this.firstName, this.lastName, this.age, this.gender, this.height, this.weight, this.sport, this.phoneBrands});

  static final List<String> userDataColumn = [
    "firstName",
    'lastName',
    'age',
    'gender',
    'height', // U+2211 unicode for sum
    'weight',
    'sport',
    'phoneBrands'
  ];

  static List<User> getUsers() {
    return <User>[
      User(firstName: "A", lastName: "Bc",age: 10,gender: "M",height: 170.2, weight: 60, sport: "baseball", phoneBrands: "apple"),
      User(firstName: "Chen", lastName: "Hong",age: 10,gender: "M",height: 150.2, weight: 40, sport: "basketball", phoneBrands: "apple"),
      User(firstName: "Clair", lastName: "Mary",age: 13,gender: "F",height: 160, weight: 49, sport: "tennis", phoneBrands: "apple"),
      User(firstName: "Hatake", lastName: "Kakashi",age: 28,gender: "M",height: 165, weight: 60, sport: "golf", phoneBrands: "oppo"),
      User(firstName: "Uzumaki", lastName: "Naruto",age: 16,gender: "M",height: 158.3, weight: 52.2, sport: "soccer", phoneBrands: "apple"),
      User(firstName: "Uchiha", lastName: "Sasuke",age: 16,gender: "M",height: 161, weight: 58.9, sport: "tennis", phoneBrands: "apple"),
      User(firstName: "Haruno", lastName: "Sakura",age: 16,gender: "F",height: 152, weight: 43.2, sport: "tennis", phoneBrands: "apple"),
      User(firstName: "Jiraiya", lastName: "",age: 38,gender: "M",height: 182, weight: 79, sport: "baseball", phoneBrands: "samsung"),
      User(firstName: "Gai", lastName: "",age: 38,gender: "M",height: 182, weight: 79, sport: "baseball", phoneBrands: "football"),
      User(firstName: "Hyuga", lastName: "Neji" ,age: 38,gender: "M",height: 182, weight: 79, sport: "baseball", phoneBrands: "football"),
      User(firstName: "Rock", lastName: "Lee" ,age: 38,gender: "M",height: 182, weight: 79, sport: "baseball", phoneBrands: "football"),
      User(firstName: "Tenten", lastName: "" ,age: 38,gender: "M",height: 182, weight: 79, sport: "baseball", phoneBrands: "football"),
      User(firstName: "Sarutobi", lastName: "Asuma" ,age: 38,gender: "M",height: 182, weight: 79, sport: "baseball", phoneBrands: "football"),
      User(firstName: "Yamanaka", lastName: "Ino" ,age: 38,gender: "M",height: 182, weight: 79, sport: "baseball", phoneBrands: "football"),
      User(firstName: "Nara", lastName: "Shikamaru" ,age: 38,gender: "M",height: 182, weight: 79, sport: "baseball", phoneBrands: "football"),
      User(firstName: "Akimichi", lastName: "Choji" ,age: 38,gender: "M",height: 182, weight: 79, sport: "baseball", phoneBrands: "football"),
      User(firstName: "Yuuhi", lastName: "Kurenai" ,age: 38,gender: "M",height: 182, weight: 79, sport: "baseball", phoneBrands: "football"),
      User(firstName: "Hyuga", lastName: "Choji",age: 38,gender: "M",height: 182, weight: 79, sport: "baseball", phoneBrands: "football"),
      User(firstName: "Akimichi", lastName: "Hinata" ,age: 38,gender: "M",height: 182, weight: 79, sport: "baseball", phoneBrands: "football"),
      User(firstName: "Aburame", lastName: "Shino" ,age: 38,gender: "M",height: 182, weight: 79, sport: "baseball", phoneBrands: "football"),
      User(firstName: "Inuzuka", lastName: "Kiba",age: 38,gender: "M",height: 182, weight: 79, sport: "baseball", phoneBrands: "football"),
    ];
  }
}
```

### Display To Page
```dart
class UserDataTable extends StatelessWidget {
  List<String> value;
  List<dynamic> record;
  String title;
  UserDataTable(
      {@required this.value, @required this.record, @required this.title});

  @override
  Widget build(BuildContext context) {
    return new Padding(
      padding: EdgeInsets.all(10),
      child: new Column( /// Layout 設計
        children: <Widget>[
          new Align(
            alignment: Alignment.topLeft, /// 位置
            child: new Container(
              child: new Text(
                (title != null) ? title : "",
                style: TextStyle(color: Colors.blue, fontSize: 24),
              ),
            ),
          ),
          new Expanded(
            child: new Scrollbar(
              child: ListView( /// 縱軸移動
                children: <Widget>[
                  new SingleChildScrollView(
                    scrollDirection: Axis.horizontal, /// 橫軸移動 
                    child: new Container(
                      child: new DataTable(
                        dataRowHeight: 30, /// 每一列的高度
                        headingRowHeight: 25, /// 表頭列的高度
                        horizontalMargin: 10, 
                        columnSpacing: 12, /// 每一行的間距
                        columns: createDataColumns(context, this.value), /// 建立表頭部分
                        rows: createDataRows(context, this.record), /// 一個 DataRow 一筆資料，一筆資料的數值用 DataCells 表示
                      ),
                    ),
                  )
                ],
              ),
            ),
          ),
        ],
      ),
    );
  }

  createDataRows(BuildContext context, List<User> tableData) {
    return tableData.map((f) {
      return new DataRow(cells: [
        new DataCell(new Text(f.firstName)),
        new DataCell(new Text(f.lastName)),
        new DataCell(new Text(f.age.toString())),
        new DataCell(new Text(f.gender)),
        new DataCell(new Text(f.height.toString())),
        new DataCell(new Text(f.weight.toString())),
        new DataCell(new Text(f.sport)),
        new DataCell(new Text(f.phoneBrands)),
      ]);
    }).toList();
  }

  createDataColumns(BuildContext context, List<String> userDataColumn) {
    return userDataColumn.map((f) {
      return new DataColumn(
        label: new Text(f, style: TextStyle(color: Colors.red)),
        numeric: false,
        tooltip: "$f",
      );
    }).toList();
  }

}
```

### main 
```dart
import 'package:flutter/material.dart';
...
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
      body: new UserDataTable(
        value: User.userDataColumn,
        record: User.getUsers(),
        title: "User"),
    );
  }
}
```

程式的進入點這邊不顯示說明。

## Conclusion
當一個 `DataTable` 資料長度過長可搭配 `SingleChildScrollView` 要垂直還是水平移動，但這只能擇一。如果資料長度同時是橫軸過長和縱軸過長，需搭配 `ListView` 和 `SingleChildScrollView` 實現縱軸和橫軸的 `scroll`。

缺點部分像是表頭不移動，這從原始 `DataTable` 是無法設定的。目前也還在研究中 XD

<!-- <figure>
    <img src="../assets/gif/20190928.gif">
</figure> -->

{{< figure src="/images/gif/20190928.gif" width="auto" height="auto">}}