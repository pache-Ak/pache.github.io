## 基于mediapipe的动态捕捉以及Unity同步模型动作

### Mediapipe简介以及动作捕捉实现

MediaPipe是谷歌开源的多面体机器学习框架，里面包含了很多例如姿态、人脸检测、虹膜等各种各样的模型以及机器学习算法。

MediaPipe 的核心框架由 C++ 实现，主要概念包括Packet、Stream、Calculator、Graph以及子图Subgraph。数据包是最基础的数据单位，一个数据包代表了在某一特定时间节点的数据，例如一帧图像或一小段音频信号；数据流是由按时间顺序升序排列的多个数据包组成，一个数据流的某一特定Timestamp只允许至多一个数据包的存在；而数据流则是在多个计算单元构成的图中流动。MediaPipe 的图是有向的——数据包从Source Calculator或者 Graph Input Stream流入图直至在Sink Calculator 或者 Graph Output Stream离开。

![](https://aijishu.com/img/bVVzx)

下面的项目使用

#### MediaPipe的人物检测框架实现流媒体动作捕捉：

```python
import cv2
from cvzone.PoseModule import PoseDetector

cap = cv2.VideoCapture('hello.mp4')

detector = PoseDetector()
posList = []
while True:
    success, img = cap.read()
    img = detector.findPose(img)
    lmList, bboxInfo = detector.findPosition(img)

    if bboxInfo:
        lmString = ''
        for lm in lmList:
            lmString += f'{lm[1]},{img.shape[0] - lm[2]},{lm[3]},'
        posList.append(lmString)

    #print(len(posList))

    cv2.imshow("Image", img)
    key = cv2.waitKey(1)
    if cv2.waitKey(1) & 0xFF == ord('q'):
        with open("MotionFile.txt", 'w') as f:
            f.writelines(["%s\n" % item for item in posList])
```

我们通过opencv库来调用本地流媒体源

同时创建一个动作捕捉对象来获取人物对应的33个关节点

每个关节点的输出如下：

[0，x，y，z]

需要注意的是，对于y坐标opencv的出发位置与unity建模的出发位置不同，opencv从左上角开始而unity从左下角开始，因此我们对y坐标进行了处理

```python
y = img.shape[0] - lm[2]
```

将数据存入posList后我们就得到了unity可以对应的所有关节节点

最后我们将数据缓存以本地的方式保存到MotionFile文件内。

### 与Unity通信实现

通信方面，我们使用Unity提供支持的Udpservice

#### 修改动作捕捉代码，利用socket服务指定本地端口向unity服务端口发送数据

```python
import cv2
from cvzone.PoseModule import PoseDetector
import socket
cap = cv2.VideoCapture(0)

sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
serverAddressPort = ("127.0.0.1", 5056)   

detector = PoseDetector()
posList = []  

while True:
    success, img = cap.read()
    img = detector.findPose(img)
    lmList, bboxInfo = detector.findPosition(img)

    if bboxInfo:
        lmString = ''
        for lm in lmList:
            lmString += f'{lm[1]},{img.shape[0] - lm[2]},{lm[3]},'
     
        print(lmString)
        date = lmString
        sock.sendto(str.encode(str(date)), serverAddressPort)

    cv2.imshow("Image", img)
```

指定本地5056端口，由unity服务端脚本接收数据

#### 在Unity内增加Udp服务脚本编写

创建UDPReceive脚本

```csharp
using UnityEngine;
using System;
using System.Text;
using System.Net;
using System.Net.Sockets;
using System.Threading;

public class UDPReceive : MonoBehaviour
{

	Thread receiveThread;
	UdpClient client;
	public int port = 5054;
	public bool Recieving = true;
	public bool printToConsole = false;
	public string data;


	public void Start()
	{

		receiveThread = new Thread(
			new ThreadStart(ReceiveData));
		receiveThread.IsBackground = true;
		receiveThread.Start();
	}

	private void ReceiveData()
	{

		client = new UdpClient(port);
		while (true) {
			if (Recieving)
			{
					try
					{
						IPEndPoint anyIP = new IPEndPoint(IPAddress.Any, 0);
						byte[] dataByte = client.Receive(ref anyIP);
						data = Encoding.UTF8.GetString(dataByte);

						if (printToConsole) { print(data); }
					}
					catch (Exception err)
					{
						print(err.ToString());
					}
			}
		}
	}
}

```


这个部分负责接受mediapipe识别后python发送的的实时数据。

定义函数 `ReciveData`在创建的接收线程 `receiveThread`中运行。
通过将 `IsBackground`设置为 `true`， 将线程设置为后台线程， 随前台结束而结束。

设置 `UdpClinet`和端口 `port`。`IPAdress.Any`表示本机地址的所有可用IP。 0表示所有可用端口。
调用 `receive`接受数据， 使用 `Encoding.UTF8.GetSting`转换为字符串。

布尔变量 `Recieving`控制是否接受数据。

布尔变量 `printToConsole`控制是否进行debug输出。

至此完成了基本的动作输入与接受

### Unity内部动作实现

#### 调整关节点响应

创建Actions脚本

```csharp
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class Actions : MonoBehaviour
{
    // Start is called before the first frame update
    public UPD udpReceive;
    public GameObject[] bodyPoints;
    void Start()
    {
      
    }

    // Update is called once per frame
    void Update()
    {
        string data = udpReceive.data;
      
        print(data);
        string[] points = data.Split(',');
        print(points[0]);


        for (int i = 0; i < 32; i++)
        {

            float help;
            float.TryParse(points[0 + (i * 3)],out help);
            float x = help/100;
            float.TryParse(points[1 + (i * 3)],out help);
            float y = help/100;
            float.TryParse(points[2 + (i * 3)],out help);
            float z = help/300;

            bodyPoints[i].transform.localPosition = new Vector3(x, y, z);

        }

    }
}


```

接受到后台数据后为动作脚本编写内容，由于动作拉伸问题我们需要对z轴做额外处理，将其值多除以3

#### 编写骨骼渲染脚本

通过Unity内置的线渲染器来编写连接关节的骨骼

```csharp
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class LineCode : MonoBehaviour
{

    LineRenderer lineRenderer;

    public Transform origin;
    public Transform destination;

    void Start()
    {
        lineRenderer = GetComponent<LineRenderer>();
        lineRenderer.startWidth = 0.1f;
        lineRenderer.endWidth = 0.1f;
    }
// 连接两个点
    void Update()
    {
        lineRenderer.SetPosition(0, origin.position);
        lineRenderer.SetPosition(1, destination.position);
    }
}

```

同时创建多个渲染副本，根据MediaPipe的采集样本将对应的骨骼点绑定。

至此实现完毕

> 联合作者 ：Pache([https://pache-ak.github.io/](https://pache-ak.github.io/)) Azula([https://limafang.github.io/Azula_blogs.github.io/](https://limafang.github.io/Azula_blogs.github.io/))
>
> 参考学习内容：https://www.youtube.com/watch?v=BtMs0ysTdkM
