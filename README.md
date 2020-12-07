# OpenDGLab OpenProtocol
DG-Lab 应用程序本身的远程协议十分孱弱，不能很好的兼顾大部分的高级动作。也不支持同时操作多个设备的多个通道。所以 OpenDGLab OpenProtocol 应运而生。  
OpenDGLab OpenProtocol 采用 Google Protobuf 来描述协议结构，并提供一套推荐的方法来管理第三方软件的控制授权，让您可以有能力将 DG-Lab 设备带入其他软件和游戏中。  
在此建议所有符合控制客户端的软件可以称作 `OpenDGLab OPClient` 以方便进行搜索。

# 协议定义与规范
> 协议版本: v1

所有遇到编码的情况均使用 UTF-8  
下方所有表述均为伪代码，请使用对应语言的 Protobuf 书写方式实现。  
## 连接
### 发送
客户端发送 DGEvent 为 CONNECT 的数据包。
```
DGRequest {
    version = 1;
    event = DGEvent.CONNECT;
    connect = {
        appName = "测试";
        uuid = "2ca93e86-475e-42b6-9a6b-97d270a6723a"
        token = ""
    };
}
```
uuid 只需要与其他客户端不同即可，appName 为需要请求授权时显示的应用名。  
如果是首次连接，token 置空即可。如果已有 token 则可以在未被取消授权的情况下避免发出授权请求，如果已被取消授权，则会再次弹出授权请求。
### 返回
```
DGResponse {
    version = 1;
    event = DGEvent.CONNECT;
    connect = {
        token = "xxxxxxxx";
    };
}
```
如果返回的 token 非空说明已经进行了授权，可以开始发送其他命令了，如果 token 为空说明授权被拒绝。  
如果在发送时已发送 token，但返回时 token 为空说明授权已被撤销。  
如果在发送时已发送 token，但返回时 token 与上次相同说明授权有效。  
如果在发送时已发送 token，但返回时 token 为上次不同说明授权已被撤销并重新授权，请保存这个新的 token 用于下次连接。  

## 获取设备信息
### 发送
```
DGRequest {
    version = 1;
    event = DGEvent.GETDEVICE;
}
```
### 返回
```
DGResponse {
    version = 1;
    event = DGEvent.GETDEVICE;
    deviceList = {
        devices = [
            {
                id = "f00000";
                isLockedByRemote = false;
                isLockedByMe = false;
            },
            {
                id = "a3135b";
                isLockedByRemote = true;
                isLockedByMe = true;
            },
            {
                id = "e412c1";
                isLockedByRemote = true;
                isLockedByMe = false;
            }
        ];
    };
}
```
返回中 DGWaveList 为列表，内部为 DGDevice 对象，id 为设备 ID，isLockedByRemote 代表这个设备是否被某个 OPClient 客户端接管，isLockedByMe 代表是否是被自己接管的设备。  
只能锁定未被其他 OPClient 接管的设备，只能操作被自己接管的设备。

## 接管设备和取消接管设备
### 发送
接管设备
```
DGRequest {
    version = 1;
    event = DGEvent.LOCKDEVICE;
    device = {
        deviceId = "f00001";
    };
}
```
取消接管设备
```
DGRequest {
    version = 1;
    event = DGEvent.UNLOCKDEVICE;
    device = {
        deviceId = "f00001";
    };
}
```
仅可以接管未被其他客户端接管的设备。  
仅可以取消结果自己接管的设备。  
### 返回
```
DGResponse {
    version = 1;
    event = DGEvent.LOCKDEVICE;
    device = {
        id = "f00001";
        isLockedByRemote = true;
        isLockedByMe = true;
    }
}
```
如果返回中 isLockedByMe 为 true 则完成接管。

## 获取强度
### 发送
```
DGRequest {
    version = 1;
    event = DGEvent.GETSTRENGTH;
    device = {
        deviceId = "f00001";
    };
}
```
### 接收
```
DGRequest {
    version = 1;
    event = DGEvent.GETSTRENGTH;
    strength = {
        strengthA = 0;
        strengthB = 10;
    };
}
```
注意：此返回将在任何强度变更时触发。

## 设置强度
### 发送
```
DGRequest {
    version = 1;
    event = DGEvent.SETSTRENGTH;
    device = {
        deviceId = "f00001";
    };
    strength = {
        strengthA = 1;
        strengthB = 4;
    };
}
```
## 获取波形列表
### 发送
```
DGRequest {
    version = 1;
    event = DGEvent.GETWAVELIST;
}
```
### 接收
```
DGResponse {
    version = 1;
    event = DGEvent.GETWAVELIST;
    waveList = {
        wave = [
            "Flick", "Click"
        ]
    }
}
```
## 获取当前的波形
### 发送
```
DGRequest {
    version = 1;
    event = DGEvent.GETWAVE;
    device = {
        deviceId = "f00001";
        deviceChannel = DGDeviceChannel.CHANNEL_A;
    };
}
```
### 接收
```
DGResponse {
    version = 1;
    event = DGEvent.GETWAVE;
    deviceId = {
        deviceId = "f00001";
        deviceChannel = DGDeviceChannel.CHANNEL_A;
    }
    waveName = {
        wave = "Flick"
    };
}
```

## 设定当前波形
### 发送
```
DGRequest {
    version = 1;
    event = DGEvent.SETWAVE;
    device = {
        deviceId = "f00001";
        deviceChannel = DGDeviceChannel.CHANNEL_A;
    };
    wave = {
        waveName = "Flick";
    };
}
```
注意： 发送不在列表内的波形名称将切换波形为外部输入模式，请使用 DGEvent.CUSTOMWAVE 来输入波形序列。

## 发送自定义波形
请注意，推荐先发送清空自定义波形指令后切换至外部输入模式再发送自定义波形，否则波形可能不同步。
### 发送
```
DGRequest {
    version = 1;
    event = DGEvent.CUSTOMWAVE;
    device = {
        deviceId = "f00001";
        deviceChannel = DGDeviceChannel.CHANNEL_A;
    }
    customWave = [
        {
            bytes = [{ 65, 32, 45 }]
        },
        {
            bytes = [{ 45, 43, 23 }]
        }
    ];
}
```
发送的波形数据 bytes 必须为3个字节。使用 OpenDGLab Core 的 waveTick 输出即可。  
每个波形序列将消耗 200 毫秒时间，如果一次性发送3个序列请 600 毫秒推送一次，以此类推。  
在切换发送和不发送（强度零和非零）时最好清除一次队列，以保证队列同步。同时如果设定为关闭（强度 0）的状态时，请不要继续发送。
## 清空自定义波形队列
### 发送
```
DGRequest {
    version = 1;
    event = DGEvent.CLEARCUSTOM;
    device = {
        deviceId = "f00001";
        deviceChannel = DGDeviceChannel.CHANNEL_B;
    };
}
```
这将清空自定义波形的数据队列。
## 设备重置通知
此通知只会从提供设备连接的服务端发送至 OPClient 客户端。
### 接收
```
DGResponse {
    version = 1;
    event = DGEvent.DEVICERESET;
    deviceId = {
        deviceId= "f00001";
    };
}
```
当收到设备重置信息时，设备可能已断开连接或者进行了其他操作。此设备将恢复为未被接管的状态。
## 错误通知
此通知只会从提供设备连接的服务端发送至 OPClient 客户端。
### 接收
```
DGResponse {
    version = 1;
    event = DGEvent.CANTDOTHIS;
    deviceId = {
        deviceId= "f00001";
    };
    error = DGError.UNKNOWN;
}
```
收到此数据包说明发生了错误。请根据错误信息自行处理。
# 开源协议
本协议以 AGPLv3 开源协议开放。