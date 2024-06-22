---
title: '使用OpenCv+Arduino实现挂机自动打怪'
date: 2022-07-21T00:00:00+08:00
lastmod: 2022-07-21T00:00:00+08:00
outdatedInfoWarning: true
featuredImage: "https://images.intotw.cn/blog/2023/11/40abddd196be7e9cb79b83534d4983a4.webp"
featuredImagePreview: "https://images.intotw.cn/blog/2023/11/40abddd196be7e9cb79b83534d4983a4.webp"
tags: ["opencv","arduino"]
categories: ["技术"]
lightgallery: true
---

# 使用OpenCv+Arduino实现挂机自动打怪

最近在玩某网游，练级十分枯燥和缓慢，就是挂机刷刷刷，所以研究一下自动化，找了个可以原地挂机刷怪的职业，然后用OpenCv检测技能冷却，冷却好了通过串口通知Arduino按下模拟键盘按键释放技能

大概流程如下：

OpenCv定时扫描屏幕->对技能区域截图->对比预设图判断技能冷却->按键事件加入队列

消费队列->取出按键事件->通过串口通知Arduino按下按键释放技能

## OpenCv代码

### 截图部分

```java
public static void ShotFrame(String fileName) throws AWTException, IOException {
        Integer x = Integer.valueOf(PropertiesHelper.getValue(窗口X));

        Integer y = Integer.valueOf(PropertiesHelper.getValue(窗口Y));

        Integer height = Integer.valueOf(PropertiesHelper.getValue(窗口高度));

        Integer width = Integer.valueOf(PropertiesHelper.getValue(窗口宽度));

        //创建一个robot对象
        Robot robut = new Robot();
        //获取屏幕分辨率
        //打印屏幕分辨率
        //创建该分辨率的矩形对象
        Dimension act=new Dimension();
        act.setSize(width,height);
        Point point=new Point(x,y);
        Rectangle screenRect = new Rectangle(point,act);
        //根据这个矩形截图
        BufferedImage bufferedImage = robut.createScreenCapture(screenRect);
        //保存截图
        File file = new File("D:\\maplePic\\"+fileName+".jpg");
        ImageIO.write(bufferedImage, "jpg", file);
    }
    
```

### OpenCv比较

这里用的是bytedeco的javacv,一个封装了OpenCv方法的库，Java用原生OpenCv比较麻烦，推荐用这个

```java
    /**
     * OpenCV-4.0.0 直方图比较
     *
     * @return: double 两张图片的相似值
     */
    public static double compareHist_2(String resource, String now) {
        Mat src_1 = imread("D:\\maplePic\\"+resource+".jpg");// 图片 1
        Mat src_2 = imread("D:\\maplePic\\"+now+".jpg");// 图片 2

        Mat hvs_1 = new Mat();
        Mat hvs_2 = new Mat();
        //图片转HSV
        cvtColor(src_1, hvs_1,COLOR_BGR2HSV);
        cvtColor(src_2, hvs_2,COLOR_BGR2HSV);

        Mat hist_1 = new Mat();
        Mat hist_2 = new Mat();


        final int[] channels = new int[]{0, 1, 2};

        final int[] histSize = new int[]{8, 8, 8};
        final float[] histRange = new float[]{0f, 255f};
        IntPointer intPtrChannels = new IntPointer(channels);
        IntPointer intPtrHistSize = new IntPointer(histSize);
        final PointerPointer<FloatPointer> ptrPtrHistRange = new PointerPointer<>(histRange, histRange, histRange);
        calcHist(hvs_1, 1, intPtrChannels, new Mat(), hist_1, 3, intPtrHistSize, ptrPtrHistRange, true, false);
        calcHist(hvs_2, 1, intPtrChannels, new Mat(), hist_2, 3, intPtrHistSize, ptrPtrHistRange, true, false);


        //图片归一化
        normalize(hist_1, hist_1, 1, hist_1.rows() , NORM_MINMAX, -1, new Mat() );
        normalize(hist_2, hist_2, 1, hist_2.rows() , NORM_MINMAX, -1, new Mat() );

        //直方图比较
        return compareHist(hist_1,hist_2,CV_COMP_CORREL);
    }
```

### 串口工具类

```java
public class SerialPortTool {

    /**
     * 查找电脑上所有可用 com 端口
     *
     * @return 可用端口名称列表，没有时 列表为空
     */
    public static final ArrayList<String> findSystemAllComPort() {
        /**
         *  getPortIdentifiers：获得电脑主板当前所有可用串口
         */
        Enumeration<CommPortIdentifier> portList = CommPortIdentifier.getPortIdentifiers();
        ArrayList<String> portNameList = new ArrayList<>();

        /**
         *  将可用串口名添加到 List 列表
         */
        while (portList.hasMoreElements()) {
            String portName = portList.nextElement().getName();//名称如 COM1、COM2....
            portNameList.add(portName);
        }
        return portNameList;
    }

    /**
     * 打开电脑上指定的串口
     *
     * @param portName 端口名称，如 COM1，为 null 时，默认使用电脑中能用的端口中的第一个
     * @param b        波特率(baudrate)，如 9600
     * @param d        数据位（datebits），如 SerialPort.DATABITS_8 = 8
     * @param s        停止位（stopbits），如 SerialPort.STOPBITS_1 = 1
     * @param p        校验位 (parity)，如 SerialPort.PARITY_NONE = 0
     * @return 打开的串口对象，打开失败时，返回 null
     */
    public static final SerialPort openComPort(String portName, int b, int d, int s, int p) {
        CommPort commPort = null;
        try {
            //当没有传入可用的 com 口时，默认使用电脑中可用的 com 口中的第一个
            if (portName == null || "".equals(portName)) {
                List<String> comPortList = findSystemAllComPort();
                if (comPortList != null && comPortList.size() > 0) {
                    portName = comPortList.get(0);
                }
            }
//            System.out.println("开始打开串口：portName=" + portName + ",baudrate=" + b + ",datebits=" + d + ",stopbits=" + s + ",parity=" + p);
            //通过端口名称识别指定 COM 端口
            CommPortIdentifier portIdentifier = CommPortIdentifier.getPortIdentifier(portName);
            /**
             * open(String TheOwner, int i)：打开端口
             * TheOwner 自定义一个端口名称，随便自定义即可
             * i：打开的端口的超时时间，单位毫秒，超时则抛出异常：PortInUseException if in use.
             * 如果此时串口已经被占用，则抛出异常：gnu.io.PortInUseException: Unknown Application
             */
            commPort = portIdentifier.open(portName, 5000);
            /**
             * 判断端口是不是串口
             * public abstract class SerialPort extends CommPort
             */
            if (commPort instanceof SerialPort) {
                SerialPort serialPort = (SerialPort) commPort;
                /**
                 * 设置串口参数：setSerialPortParams( int b, int d, int s, int p )
                 * b：波特率（baudrate）
                 * d：数据位（datebits），SerialPort 支持 5,6,7,8
                 * s：停止位（stopbits），SerialPort 支持 1,2,3
                 * p：校验位 (parity)，SerialPort 支持 0,1,2,3,4
                 * 如果参数设置错误，则抛出异常：gnu.io.UnsupportedCommOperationException: Invalid Parameter
                 * 此时必须关闭串口，否则下次 portIdentifier.open 时会打不开串口，因为已经被占用
                 */
                serialPort.setSerialPortParams(b, d, s, p);
//                System.out.println("打开串口 " + portName + " 成功...");
                return serialPort;
            } else {
                System.out.println("当前端口 " + commPort.getName() + " 不是串口...");
            }
        } catch (NoSuchPortException e) {
            e.printStackTrace();
        } catch (PortInUseException e) {
            System.out.println("串口 " + portName + " 已经被占用，请先解除占用...");
            e.printStackTrace();
        } catch (UnsupportedCommOperationException e) {
            System.out.println("串口参数设置错误，关闭串口，数据位[5-8]、停止位[1-3]、验证位[0-4]...");
            e.printStackTrace();
            if (commPort != null) {//此时必须关闭串口，否则下次 portIdentifier.open 时会打不开串口，因为已经被占用
                commPort.close();
            }
        }
        System.out.println("打开串口 " + portName + " 失败...");
        return null;
    }

    /**
     * 往串口发送数据
     *
     * @param serialPort 串口对象
     * @param order      待发送数据
     */
    public static void sendDataToComPort(SerialPort serialPort, byte[] orders) {
        OutputStream outputStream = null;
        try {
            if (serialPort != null) {
                outputStream = serialPort.getOutputStream();
                outputStream.write(orders);
                outputStream.flush();
//                System.out.println("往串口 " + serialPort.getName() + " 发送数据：" + Arrays.toString(orders) + " 完成...");
            } else {
                System.out.println("gnu.io.SerialPort 为null，取消数据发送...");
            }
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            if (outputStream != null) {
                try {
                    outputStream.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }
    }

    /**
     * 往串口发送数据
     *
     * @param serialPort 串口对象
     */
    public static String readDataToComPort(SerialPort serialPort) {

        try (InputStream inputStream = serialPort.getInputStream(); ByteArrayOutputStream byteArrayOutputStream=new ByteArrayOutputStream()) {
            byte[] a=new byte[1024];
            while (inputStream.read(a)>0){
                byteArrayOutputStream.write(a);
            }
            return byteArrayOutputStream.toString();
        } catch (IOException e) {
            e.printStackTrace();
        }
        return "";
    }

    /**
     * 关闭串口
     *
     * @param serialport 待关闭的串口对象
     */
    public static void closeComPort(SerialPort serialPort) {
        if (serialPort != null) {
            serialPort.close();
//            System.out.println("关闭串口 " + serialPort.getName());
        }
    }

    /**
     * 16进制字符串转十进制字节数组
     * 这是常用的方法，如某些硬件的通信指令就是提供的16进制字符串，发送时需要转为字节数组再进行发送
     *
     * @param strSource 16进制字符串，如 "455A432F5600"，每两位对应字节数组中的一个10进制元素
     *                  默认会去除参数字符串中的空格，所以参数 "45 5A 43 2F 56 00" 也是可以的
     * @return 十进制字节数组, 如 [69, 90, 67, 47, 86, 0]
     */
    public static byte[] hexString2Bytes(String strSource) {
        if (strSource == null || "".equals(strSource.trim())) {
            System.out.println("hexString2Bytes 参数为空，放弃转换.");
            return null;
        }
        strSource = strSource.replace(" ", "");
        int l = strSource.length() / 2;
        byte[] ret = new byte[l];
        for (int i = 0; i < l; i++) {
            ret[i] = Integer.valueOf(strSource.substring(i * 2, i * 2 + 2), 16).byteValue();
        }
        return ret;
    }

    public static void main(String[] args) {
//        //发送普通数据
        byte[] bytes = new byte[]{0x31};
        SerialPort serialPort = SerialPortTool.openComPort("COM5", 9600, 8, 1, 0);
        SerialPortTool.sendDataToComPort(serialPort, bytes);
        String s = SerialPortTool.readDataToComPort(serialPort);
        System.out.println(s);
        SerialPortTool.closeComPort(serialPort);
    }
}
```


### 主流程函数

```java
 public static void main(String[] args) {
        //每秒一次
        Timer timer = new Timer();
        TimerTask timerTask = new TimerTask() {
            @Override
            public void run() {
                try {
                    lhbrother();
                } catch (IOException e) {
                    e.printStackTrace();
                } catch (AWTException e) {
                    e.printStackTrace();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        };
        // 第一次任务延迟时间
        long delay = 2000;

        // 任务执行频率
        long period = 1000;

        // 开始调度
        timer.schedule(timerTask, delay, period);

    }
    public static void lhbrother() throws IOException, AWTException, InterruptedException {
        //截图生成结果
        ScreenShotUtil.Shot1("n1");
        if(OpenCvUtil.compareHist_2("n1","1")>0.90){
            System.out.println("1冷却完毕");
            Random random=new Random();
            Thread.sleep(random.nextInt(500)+500);
            //这里发送的是ascii码，Arduino读取串口后直接KeyBorad.Press读到的内容即可。
            //因为ascii码没有上下左右等方向键，所以如果需要控制上下走动，可以使用一个特殊的ascii码，然后在Arduino代码里特殊判断下
            byte[] bytes = new byte[]{0x31};
            SerialPort serialPort = SerialPortTool.openComPort("COM5", 9600, 8, 1, 0);
            SerialPortTool.sendDataToComPort(serialPort, bytes);
            String s = SerialPortTool.readDataToComPort(serialPort);
            System.out.println(s);
            SerialPortTool.closeComPort(serialPort);
            return ;
        }
    }
```
