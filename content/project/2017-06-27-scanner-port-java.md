---
title: "scanner port"
date: 2017-06-27
tags: ["JAVA", "school"]
categories: ["school"]
description: "使用 JAVA 掃描服務 Port"
draft: false
---

```java
import java.io.IOException;
import java.net.ServerSocket;

public class PortScanner{
    public void scan(){
        for(int i=1; i < 65535 ; i++ ){//port 是 0 - 65535
            try{
                ServerSocket ss = new ServerSocket(i);//  ServerSocket(int port) 建構
            }catch(IOException ex){// port 被占用則拋出例外
                System.out.println("Port: "+ i + " Occupied");
            }
        }
    }

    public static void main(String[] args) {
        PortScanner pserver = new PortScanner();
        pserver.scan();
    }
}
// 在 TCP 協定中，埠號 0 是被保留的，不可使用。在 UDP 協定中，來源埠號是可以選擇要不要填上，如果設為 0，則代表沒有來源埠號。
```