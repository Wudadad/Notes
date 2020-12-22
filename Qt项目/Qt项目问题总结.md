#### 1、做后端通信拆包打包数据的时候，如何做到数据正确解析？

##### 1、字节序，大小端问题

使用udp通信的另一端是c语言编写，我们这边解析数据用java接收，通过对数据进行比对发现字节序是反的。java的默认字节序是大端字节序，顾名思义大端字节序就是高位在前，例如：0x1234，高位字节为x12，低位字节为0x34，符合人类的读取习惯。但是计算机处理高低电平的逻辑是从低位先开始处理，所以存在另一种小端字节序。顾名思义小端字节序就是低位在前，例如：0x1234，高位字节为：0x34，低位字节为：0x12。虽然在内存里存放的形式一样，但表达的意义却不相同，所以在接收数据时必须按照和发送方相应的字节序来解析数据，否则就会得到错误且超出预期的结果。

##### 2、如何打包解包数据

在java的nio包里有个ByteOrder的类可以用来标志字节序，再配合上ByteBuffer的包装，就可以很容易地对收到的字节流数据进行拆包分析。附上拆包步骤：

```java
public static final ByteOrder _byteOrder = ByteOrder.LITTLE_ENDIAN; //定义字节序
```

```java
//自定义wrap方法替换ByteBuffer自带的wrap方法，已达到自定义字节序的效果
public static ByteBuffer wrap(byte[] buffer) {
        //return ByteBuffer.wrap(array)
        ByteBuffer result = ByteBuffer.allocate(buffer.length).order(_byteOrder);    
        result.put(buffer);
        result.position(0);
        return result;
    }
```

```java
//示例如何如何使用ByteBuffer的get方法读取数据
public static EquipmentStatusPacket unpackEquipmentStatusPacket(byte[] EquipmentStatusPacket) throws Exception{
     	ByteBuffer bytebuffer = wrap(EquipmentStatusPacket); 
     	//数据长度检查
     	if (EquipmentStatusPacket.length != _EquipmentStatusPacketLength) {
     		throw new Exception("状态报长度为：" + EquipmentStatusPacket.length + "，状态报长度应为：" + 108);
         }

         //信息类别号检查
        if (EquipmentStatusPacket[0] != _EquipmentStatusPacketID) {
         	throw new Exception("首字节为：" + EquipmentStatusPacket[0] + "，不是设备状态报");
         }
        //设备标识检查
        if (EquipmentStatusPacket[1] != _EquipmentIdentifyID) {
        	throw new Exception("设备标识为：" + EquipmentStatusPacket[1] + "，设备标识应为：" + 0x03);
        }
        bytebuffer.position(1);																//从第二个字节开始是设备标识
        short equipmentID = (short)(bytebuffer.get() & 0x00ff);								//设备标识
        //获取设备状态报时间日期
        byte[] timeArray = new byte[_datePacketLength];
        bytebuffer.get(timeArray, 0, _datePacketLength);
        LocalDateTime time = unpackDatePacket(wrap(timeArray));					//时间日期
    		
        short technicalStatus = (short)(bytebuffer.get() & 0x00ff);							//技术状态
        short workStatus = (short)(bytebuffer.get() & 0x00ff);								//工作状态
    
        short infraredStatus = (short)(bytebuffer.get() & 0x00ff);							//红外组件状态
        short moduleStatus = (short)(bytebuffer.get() & 0x00ff);							//各模块状态
    
```

##### 3、C++里没有byte这种类型，如何正确打包拆包数据？

按位读取数据，每次读取时移动指针即可。下面这段代码我定义了一些打包和解包byte，short，int数据的方法，在收到字节流后调用这些方法。这个代码的正确性没有在工程中得到验证，因为考虑到时间和效率的问题，原本打算用Qt重写原有的整个java项目后来改成只重写界面部分，所以这段解析数据报的部分就没有用到，但是经过我自己的测试，解出来的数据对象都是正确的。

```c++
//字节序为小端读取，todo：大小端模式可切换
void EquipmentPacketProcesser::packUByte(uint8_t *&buffer, uint8_t value)
{
    *buffer = value;
    buffer = buffer + sizeof(uint8_t);
}
void EquipmentPacketProcesser::packUShort(uint8_t *&buffer, uint16_t value)
{
    packUByte(buffer, value & 0x00FF);
    packUByte(buffer, (value >> 8) & 0x00FF);
}
void EquipmentPacketProcesser::packUInt(uint8_t *&buffer, uint32_t value)
{
    packUByte(buffer, value & 0x000000FF);
    packUByte(buffer, (value >> 8) & 0x000000FF);
    packUByte(buffer, (value >> 16) & 0x000000FF);
    packUByte(buffer, (value >> 24) & 0x000000FF);
}
```

```c++
uint8_t EquipmentPacketProcesser::unpackUByte(uint8_t *&buffer)
{
    uint8_t value = *buffer;
    buffer = buffer + sizeof(uint8_t);
    return value;
}
uint16_t EquipmentPacketProcesser::unpackUShort(uint8_t *&buffer)
{
    uint16_t value = *(uint16_t *)buffer;
    buffer = buffer + sizeof(uint16_t);
    return value;
}
uint32_t EquipmentPacketProcesser::unpackUInt(uint8_t *&buffer)
{
    uint32_t value = *(uint32_t *)buffer;
    buffer = buffer + sizeof(uint32_t);
    return value;
}
```

#### 2、Qt中如何获取鼠标的位置，鼠标位置和界面的相对关系是什么样的？

可以通过这张图来了解鼠标的位置关系：

![img](https://img-blog.csdn.net/20170930142105221?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvYW5mbHlpbmdjYXQ=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

对于QWidge界面，想要获得相对于Widge界面的鼠标位置，只需要用全局坐标减去窗口顶点坐标即可：

```c++
//鼠标当前的坐标
    int x = event->globalPos().x() - geometry().x();
    int y = event->globalPos().y() - geometry().y();
```

对于QMainWindow界面，会有一个menubar，这个menubar是不算在绘制界面的起始坐标的，所以项目里不同区域的坐标属性里y值是不算这个menubar的height，y值从0开始而不是从menubar.height()开始。而在鼠标事件中想要在相对于窗口的鼠标位置进行绘图等操作时，上述获取鼠标的操作y值会从menubar.y()开始算起，这样就会导致真实绘制出来的区域和预期的区域相差了一个menubar.height()。这个问题困扰了我一个下午，最后是将所有需要使用鼠标坐标的区域y值都加上一个menubar.height()来使他们的坐标值保持相对一致，这样看起来绘制图形的效果看起来就正确了。

#### 3、如何将位图载入成QImage?

通过Qt里的avro配置收到std:vector<uint8_t>类型的位图，如何将他载入成QImage显示在界面上呢？我采取的是将std::vector<uint8_t>类型先转换为QByteArray，再使用Qimge的构造函数读取QByteArray来实现。代码如下：

```c++
QByteArray * imgData = new QByteArray(reinterpret_cast<const char*>(imageData->image.data()), imageData->image.size());
this->UDPSource.loadFromData(*imgData);
```

#### 4、如何转换含毫秒的时间字符串？

在javaScript里面用到的format格式为：yyyy-MM-dd hh:mm:ss:SSS，以下代码将当前时间格式化：

```javascript
//yyyy-MM-dd-hh:mm:ss:SSS
  public setTime() {
    var date = new Date();
    var o = {
      "M+": date.getMonth() + 1,                 //月份 
      "d+": date.getDate(),                    //日 
      "h+": date.getHours(),                   //小时 
      "m+": date.getMinutes(),                 //分 
      "s+": date.getSeconds(),                 //秒 
      "S+": date.getMilliseconds()             //毫秒 
    };
    var fmt = "yyyy-MM-dd hh:mm:ss:SSS"
    if (/(y+)/.test(fmt)) {
      fmt = fmt.replace(RegExp.$1, (date.getFullYear() + "").substr(4 - RegExp.$1.length));
    }
    for (var k in o) {
      if (new RegExp("(" + k + ")").test(fmt)) {
        var q = o[k];
        if(RegExp.$1.length == 2){
          o[k] = ("00" + o[k]).substr(("" + o[k]).length);
        }else if(RegExp.$1.length == 3){
          o[k] = ("000" + o[k]).substr(("" + o[k]).length);
        }
        fmt = fmt.replace(RegExp.$1,o[k]);

      }
    }
    return fmt;
  }
```

在QT中用到的format格式为：yyyy-MM-dd hh:mm:ss.zzz，以下代码为解析带毫秒的开始时间字串

```
QDateTime startDatetime = QDateTime::fromString(pd->startTime, "yyyy-MM-dd hh:mm:ss.zzz");
```

#### 5、在radio中如何使已经点击过的选择清空，处于什么都不选的状态？

初始化的时候radio可以不选，但是一旦选择过，要想设置为未选中状态单纯使用setchecked函数就不管用了

需要使用setsetAutoExclusive

```c++
ui->roundScanRadio->setAutoExclusive(false);
ui->roundScanRadio->setChecked(status);
ui->roundScanRadio->setAutoExclusive(true);
```