---
title: Android低功耗蓝牙
tags: 
    - Android
    - bluetooth
categories: Android
---

# 关键术语和概念
## 下面是一些BLE关键术语和概念的摘要

* Generic Attribute Profile（GATT）：GATT profile是一种关于发送和接收简短数据片段的一般规范，这种简短数据片段例如在BLE的连接上众所周知的“attribute（属性）”等。当前所有低功耗应用程序的profile都基于GATT。另外，蓝牙技术联盟（Bluetooth SIG）已经为很多BLE设备定义了profile。profile就是一种在指定的应用程序中定义设备如何工作的规范。注意，一台设备可以实现多个profile。例如，一台设备可以包含心率监视器和电池电量探测器。
* Attribute Protocol（ATT，属性协议）：GATT构建在ATT的基础之上，因此也总被成为GATT/ATT。ATT针对BLE设备的运行进行了优化。为此，它会尽可能的使用更少的字节数据。每一个属性都通过UUID来唯一确定。UUID就是一个标准128位格式的字符串ID，用于唯一确定一个信息。属性通过ATT协议格式化为characteristics和services后进行传输。
* Characteristic：一个characteristic中包含一个值，以及0个或多个用于描述characteristic值的descriptor。可以将characteristic认为是一种类型，类似于一个类。
* Descriptor：Descriptor（描述符）中定义的属性用于描述一个characteristic值。例如，一个descriptor可以为一个characteristic的值指定一个在可接受范围内的可读性描述，或者为一个characteristic的值指定一个计量单位。
* Service：一个service是一个characteristic的集合。例如，你可以持有一个名为“心率监视器”的service，它包含的characteristic例如“心率测量”。你可以在bluetooth.org上找到一系列基于GATT的profile和service。

## 角色和职能

以下是一台Android设备与BLE设备交互时的一些适用角色和职能：



* 中央设备和外围设备。这适用于BLE自身的连接。担任中央设备角色的设备负责扫描和搜索广告，担任外围设备的角色负责发送广告。

* GATT服务端和GATT客户端。这取决于两台设备在建立连接后如何互相通信。


为了理解这之间的区别，想象你有一台Android手机和一台BLE设备作为活动追踪器。手机将担任中央设备角色；活动追踪器将担任外围设备角色（你需要具备两种角色才能建立一个BLE连接，两者都担任外围设备角色不能互相通信，同样两者都担任中央设备角色也不能互相通信）。


一旦手机和活动追踪器建立连接，它们就可以互相传输GATT媒体数据。根据它们传输的数据，其中一方需要担任服务端的角色。例如，如果活动追踪器想要发送传感器数据给手机，活动追踪器就需要担任服务端的角色。如果活动追踪器想要接收手机的数据，手机就需要担任服务端的角色。


在本片文档的例子中，Android应用程序（运行在Android设备上）是GATT客户端。应用程序从GATT服务端获取数据，这个服务端由支持Heart Rate Profile的BLE心率监视器设备担任。但是你可以交替让你的Android应用程序扮演GATT服务端的角色。具体参考[BluetoothGattService](http://developer.android.com/reference/android/bluetooth/BluetoothGattServer.html)。


# BLE Permissions（BLE权限）
为了在你的应用程序中使用Bluetooth的功能，你必须声明android.permission.BLUETOOTH权限。你需要这个权限来执行一些蓝牙通信的操作，例如请求链接，接受连接，还有传输数据。


如果你想让你的应用程序进行设备扫描或者管理蓝牙设置，你必须同时声明android.permission.BLUETOOTH_ADMIN权限。注意，如果你使用BLUETOOTH_ADMIN权限，你必须同时声明BLUETOOTH权限。


在你的应用程序manifest文件中声明蓝牙权限，例如：


```xml
<uses-permission android:name="android.permission.BLUETOOTH"/>  
<uses-permission android:name="android.permission.BLUETOOTH_ADMIN"/>
```
如果你想声明你的应用程序只能在支持BLE的设备上运行，可以将下面声明包含进你的应用程序manifest文件中：

```xml
<uses-feature android:name="android.hardware.bluetooth_le" android:required="true"/> 
```

然而，如果你想让你的应用程序也能够在不支持BLE的设备上运行，你就应该将上面标签中的属性设置为`required="false"`。然后在运行的过程中使用`PackageManager.hasSystemFeature()`方法来判断设备是否支持BLE：


```java
// 使用下面的方法来确定设备是否支持BLE, 然后你可以选择禁用BLE的功能  
if (!getPackageManager().hasSystemFeature(PackageManager.FEATURE_BLUETOOTH_LE)) {  
    Toast.makeText(this, R.string.ble_not_supported, Toast.LENGTH_SHORT).show();  
    finish();  
}
```

# Setting Up BLE（设置BLE）
在你的应用程序通过BLE进行通信之前，你需要确认设备是否支持BLE。如果支持，还要再确认是否已经启用。注意这个检查步骤只有在`<uses-festure.../>`设置为`false`的情况下才需要执行。


如果不支持BLE，你应该优雅的禁止一些使用BLE的功能。如果支持BLE，但是目前禁用了，那么你需要在不离开你的应用程序状态下，请求用户启用蓝牙用能。这个过程需要使用BluetoothAdapter，分两个步骤完成：

## 获取BluetoothAdapter

基本上所有使用蓝牙的activity都需要BluetoothAdapter。BluetoothAdapter代表了设备本身的蓝牙适配器（蓝牙发送接收器）。在整个系统中有一个BluetoothAdapter对象，你的应用程序可以使用这个对象进行交互。下面的代码片段展示了如果获取这个适配器。注意下面的这种方法使用getSystemService()方法来获取一个BluetoothManager实例，之后再通过BluetoothManager获取BluetoothAdapter。Android 4.3（API Level 18）才开始支持BluetoothManager：

```java
// 初始化蓝牙适配器.  
final BluetoothManager bluetoothManager =  
        (BluetoothManager) getSystemService(Context.BLUETOOTH_SERVICE);  
mBluetoothAdapter = bluetoothManager.getAdapter();  

```

## 启用蓝牙
下一步，你需要确保蓝牙已经启动。调用`isEnable()`方法来检查蓝牙当前是否已经启用。如果方法返回false，说明蓝牙被禁用了。下面的代码片段中检查蓝牙是否已经启用。如果没有启用，代码片段会显示一个错误提示用户去设置中启用蓝牙：

```java
private BluetoothAdapter mBluetoothAdapter;  
...  
// 确认设备支持蓝牙并且已经启用. 如果没有,  
// 显示一个对话框要求用户授权启用蓝牙.  
if (mBluetoothAdapter == null || !mBluetoothAdapter.isEnabled()) {  
    Intent enableBtIntent = new Intent(BluetoothAdapter.ACTION_REQUEST_ENABLE);  
    startActivityForResult(enableBtIntent, REQUEST_ENABLE_BT);  
}  
```

# Finding BLE Devices（搜索BLE设备）
搜索BLE设备，你可以使用startLeScan()方法。这个方法需要一个BluetoothAdapter.LeScanCallback对象作为参数。你必须实现这个callback接口，因为扫描的结果会通过这个接口返回。由于搜索设备是比较耗电的操作，你应该遵循以下指南使用：


*  一旦你找到目标设备，应该马上停止搜索。

*  不要死循环搜索，并设置搜索的最长时间。一台以前可以访问的设备可能已经移出了可检测范围，继续扫描只会消耗电量。


下面的代码片段展示了如何开始和停止搜索


```java
/**
 * 扫描和显示可访问BLE设备的Activity.
 */
public class DeviceScanActivity extends ListActivity {

    private BluetoothAdapter mBluetoothAdapter;
    private boolean mScanning;
    private Handler mHandler;

    // 10秒钟后停止扫描.
    private static final long SCAN_PERIOD = 10000;
    ...
    private void scanLeDevice(final boolean enable) {
        if (enable) {
            // 在预定义的扫描时间周期后停止扫描.
            mHandler.postDelayed(new Runnable() {
                @Override
                public void run() {
                    mScanning = false;
                    mBluetoothAdapter.stopLeScan(mLeScanCallback);
                }
            }, SCAN_PERIOD);

            mScanning = true;
            mBluetoothAdapter.startLeScan(mLeScanCallback);
        } else {
            mScanning = false;
            mBluetoothAdapter.stopLeScan(mLeScanCallback);
        }
        ...
    }
...
}
```

如果你只想搜索指定类型的外围设备，你可以替换成startLeScan(UUID[], BluetoothAdapter.LeScanCallback)方法，并提供一个你的应用程序所支持的GATT服务的UUID对象数组。


下面是一个BluetoothAdapter.LeScanCallback的实现，它是一个用于接收BLE搜索结果的接口：


```java
private LeDeviceListAdapter mLeDeviceListAdapter;
...
// 设备搜索回调接口.
private BluetoothAdapter.LeScanCallback mLeScanCallback =
        new BluetoothAdapter.LeScanCallback() {
    @Override
    public void onLeScan(final BluetoothDevice device, int rssi,
            byte[] scanRecord) {
        runOnUiThread(new Runnable() {
           @Override
           public void run() {
               mLeDeviceListAdapter.addDevice(device);
               mLeDeviceListAdapter.notifyDataSetChanged();
           }
       });
   }
};
```

注意：正如Bluetooth文档中所描述的，在同一个时间你只能搜索BLE设备或者搜索传统蓝牙设备。你不能同时搜索BLE设备和传统蓝牙设备。

# Connecting to a GATT Server（连接一个GATT服务）

与BLE设备交互的第一步就是要连接上它——更准确的说，是连接设备上的GATT服务。连接BLE设备上的GATT服务，你可以使用connectGatt()方法。这个方法需要三个参数：一个Context对象，autoConnect（一个表示是否当BLE设备可访问时马上自动连接的boolean值），还有一个BluetoothGattCallback对象:

```java
mBluetoothGatt = device.connectGatt(this, false, mGattCallback);
```

上面的代码会连接BLE设备管理的GATT服务，并返回一个BluetoothGatt实例，通过这个实例就可以执行GATT客户端的相关操作。这个调用者（Android应用程序）就是GATT客户端。里面的BluetoothGattCallback对象用于交付操作结果给客户端，例如连接状态，还有将来一些GATT客户端操作的结果。


在这个例子中，BLE应用程序提供了一个activity（DeviceControlActivity）来连接、显示数据，和显示BLE设备所支持的GATT的service以及characteristic。基于用户的输入，这个activity会和一个名为BluetoothLeService的Service通信，这个service通过Android BLE的API与BLE设备进行交互。


```java
// 一个通过Android BLE API与BLE设备进行交互的service.
public class BluetoothLeService extends Service {
    private final static String TAG = BluetoothLeService.class.getSimpleName();

    private BluetoothManager mBluetoothManager;
    private BluetoothAdapter mBluetoothAdapter;
    private String mBluetoothDeviceAddress;
    private BluetoothGatt mBluetoothGatt;
    private int mConnectionState = STATE_DISCONNECTED;

    private static final int STATE_DISCONNECTED = 0;
    private static final int STATE_CONNECTING = 1;
    private static final int STATE_CONNECTED = 2;

    public final static String ACTION_GATT_CONNECTED =
            "com.example.bluetooth.le.ACTION_GATT_CONNECTED";
    public final static String ACTION_GATT_DISCONNECTED =
            "com.example.bluetooth.le.ACTION_GATT_DISCONNECTED";
    public final static String ACTION_GATT_SERVICES_DISCOVERED =
            "com.example.bluetooth.le.ACTION_GATT_SERVICES_DISCOVERED";
    public final static String ACTION_DATA_AVAILABLE =
            "com.example.bluetooth.le.ACTION_DATA_AVAILABLE";
    public final static String EXTRA_DATA =
            "com.example.bluetooth.le.EXTRA_DATA";

    public final static UUID UUID_HEART_RATE_MEASUREMENT =
            UUID.fromString(SampleGattAttributes.HEART_RATE_MEASUREMENT);

    // BLE API定义的各个回调方法.
    private final BluetoothGattCallback mGattCallback =
            new BluetoothGattCallback() {
        @Override
        public void onConnectionStateChange(BluetoothGatt gatt, int status,
                int newState) {
            String intentAction;
            if (newState == BluetoothProfile.STATE_CONNECTED) {
                intentAction = ACTION_GATT_CONNECTED;
                mConnectionState = STATE_CONNECTED;
                broadcastUpdate(intentAction);
                Log.i(TAG, "连接GATT服务.");
                Log.i(TAG, "尝试开始service搜索:" +
                        mBluetoothGatt.discoverServices());

            } else if (newState == BluetoothProfile.STATE_DISCONNECTED) {
                intentAction = ACTION_GATT_DISCONNECTED;
                mConnectionState = STATE_DISCONNECTED;
                Log.i(TAG, "断开GATT server连接.");
                broadcastUpdate(intentAction);
            }
        }

        @Override
        // New services discovered
        public void onServicesDiscovered(BluetoothGatt gatt, int status) {
            if (status == BluetoothGatt.GATT_SUCCESS) {
                broadcastUpdate(ACTION_GATT_SERVICES_DISCOVERED);
            } else {
                Log.w(TAG, "onServicesDiscovered received: " + status);
            }
        }

        @Override
        // Result of a characteristic read operation
        public void onCharacteristicRead(BluetoothGatt gatt,
                BluetoothGattCharacteristic characteristic,
                int status) {
            if (status == BluetoothGatt.GATT_SUCCESS) {
                broadcastUpdate(ACTION_DATA_AVAILABLE, characteristic);
            }
        }
     ...
    };
...
}
```

当一个特定的的回调方法被调用的时候，它就会适当调用broadcastUpdate()帮助方法并传递一个操作标识。注意本节中的数据是根据蓝牙心率测量的profile规范解析的


```java
private void broadcastUpdate(final String action) {
    final Intent intent = new Intent(action);
    sendBroadcast(intent);
}

private void broadcastUpdate(final String action,
                             final BluetoothGattCharacteristic characteristic) {
    final Intent intent = new Intent(action);

    // 按照心率测量的profile进行的特定处理.
    // 按照么一个profile规范进行数据解析.
    if (UUID_HEART_RATE_MEASUREMENT.equals(characteristic.getUuid())) {
        int flag = characteristic.getProperties();
        int format = -1;
        if ((flag & 0x01) != 0) {
            format = BluetoothGattCharacteristic.FORMAT_UINT16;
            Log.d(TAG, "Heart rate format UINT16.");
        } else {
            format = BluetoothGattCharacteristic.FORMAT_UINT8;
            Log.d(TAG, "Heart rate format UINT8.");
        }
        final int heartRate = characteristic.getIntValue(format, 1);
        Log.d(TAG, String.format("Received heart rate: %d", heartRate));
        intent.putExtra(EXTRA_DATA, String.valueOf(heartRate));
    } else {
        // 针对其他profiles, 将数据格式化为16禁止数据.
        final byte[] data = characteristic.getValue();
        if (data != null && data.length > 0) {
            final StringBuilder stringBuilder = new StringBuilder(data.length);
            for(byte byteChar : data)
                stringBuilder.append(String.format("%02X ", byteChar));
            intent.putExtra(EXTRA_DATA, new String(data) + "\n" +
                    stringBuilder.toString());
        }
    }
    sendBroadcast(intent);
}
```
回到DeviceControlActivity，下面的事件通过一个BroadcaseReceiver处理：


```java
// 处理Service发送过来的各种时间.
// ACTION_GATT_CONNECTED: 连接上了一个GATT服务.
// ACTION_GATT_DISCONNECTED: 断开了一个GATT服务.
// ACTION_GATT_SERVICES_DISCOVERED: 发现了GATT服务.
// ACTION_DATA_AVAILABLE: 从设备接收到数据. 这里可能是一个读取或者通知操作的结果。
private final BroadcastReceiver mGattUpdateReceiver = new BroadcastReceiver() {
    @Override
    public void onReceive(Context context, Intent intent) {
        final String action = intent.getAction();
        if (BluetoothLeService.ACTION_GATT_CONNECTED.equals(action)) {
            mConnected = true;
            updateConnectionState(R.string.connected);
            invalidateOptionsMenu();
        } else if (BluetoothLeService.ACTION_GATT_DISCONNECTED.equals(action)) {
            mConnected = false;
            updateConnectionState(R.string.disconnected);
            invalidateOptionsMenu();
            clearUI();
        } else if (BluetoothLeService.
                ACTION_GATT_SERVICES_DISCOVERED.equals(action)) {
            // 显示所有支持的service和characteristic。
            displayGattServices(mBluetoothLeService.getSupportedGattServices());
        } else if (BluetoothLeService.ACTION_DATA_AVAILABLE.equals(action)) {
            displayData(intent.getStringExtra(BluetoothLeService.EXTRA_DATA));
        }
    }
};
```

# Reading BLE Attribute（读取BLE属性）


一旦你的Android应用程序连接上了一个GATT服务并且发现了设备上的service，就可以在支持读写的地方读写属性。例如，下面的代码片段通过迭代服务的service和characteristic，并将它们显示在界面上：


```java
public class DeviceControlActivity extends Activity {
    ...
    // 演示如何迭代所支持的GATT Services/Characteristics.
    // 在这个例子中，我们填充绑定到ExpandableListView的数据结构。
    private void displayGattServices(List<BluetoothGattService> gattServices) {
        if (gattServices == null) return;
        String uuid = null;
        String unknownServiceString = getResources().
                getString(R.string.unknown_service);
        String unknownCharaString = getResources().
                getString(R.string.unknown_characteristic);
        ArrayList<HashMap<String, String>> gattServiceData =
                new ArrayList<HashMap<String, String>>();
        ArrayList<ArrayList<HashMap<String, String>>> gattCharacteristicData
                = new ArrayList<ArrayList<HashMap<String, String>>>();
        mGattCharacteristics =
                new ArrayList<ArrayList<BluetoothGattCharacteristic>>();

        // 循环迭代可访问的GATT Services.
        for (BluetoothGattService gattService : gattServices) {
            HashMap<String, String> currentServiceData =
                    new HashMap<String, String>();
            uuid = gattService.getUuid().toString();
            currentServiceData.put(
                    LIST_NAME, SampleGattAttributes.
                            lookup(uuid, unknownServiceString));
            currentServiceData.put(LIST_UUID, uuid);
            gattServiceData.add(currentServiceData);

            ArrayList<HashMap<String, String>> gattCharacteristicGroupData =
                    new ArrayList<HashMap<String, String>>();
            List<BluetoothGattCharacteristic> gattCharacteristics =
                    gattService.getCharacteristics();
            ArrayList<BluetoothGattCharacteristic> charas =
                    new ArrayList<BluetoothGattCharacteristic>();
           // 循环迭代可访问的Characteristics.
            for (BluetoothGattCharacteristic gattCharacteristic :
                    gattCharacteristics) {
                charas.add(gattCharacteristic);
                HashMap<String, String> currentCharaData =
                        new HashMap<String, String>();
                uuid = gattCharacteristic.getUuid().toString();
                currentCharaData.put(
                        LIST_NAME, SampleGattAttributes.lookup(uuid,
                                unknownCharaString));
                currentCharaData.put(LIST_UUID, uuid);
                gattCharacteristicGroupData.add(currentCharaData);
            }
            mGattCharacteristics.add(charas);
            gattCharacteristicData.add(gattCharacteristicGroupData);
         }
    ...
    }
...
}
```

# Receiving GATT Notification（接收GATT通知）

BLE应用程序要求在设备的某个指定characteristic改变的时候接收到通知是很常见。下面的代码片段展示了如何通过使用setCharacteristicNotification()为一个characteristic设置通知：


```java
private BluetoothGatt mBluetoothGatt;
BluetoothGattCharacteristic characteristic;
boolean enabled;
...
mBluetoothGatt.setCharacteristicNotification(characteristic, enabled);
...
BluetoothGattDescriptor descriptor = characteristic.getDescriptor(
        UUID.fromString(SampleGattAttributes.CLIENT_CHARACTERISTIC_CONFIG));
descriptor.setValue(BluetoothGattDescriptor.ENABLE_NOTIFICATION_VALUE);
mBluetoothGatt.writeDescriptor(descriptor);
```

一旦为一个characteristic启用了通知，当远程设备上的characteristic改变的时候就会触发onCharacteristicChanged()方法：

```java
@Override
// Characteristic notification
public void onCharacteristicChanged(BluetoothGatt gatt,
        BluetoothGattCharacteristic characteristic) {
    broadcastUpdate(ACTION_DATA_AVAILABLE, characteristic);
}
```

# Closing the Client App（关闭客户端应用程序）
一旦你的应用程序使用完BLE设备，你应该调用close()方法，这样系统才能适当释放占用的资源：


```java
public void close() {
    if (mBluetoothGatt == null) {
        return;
    }
    mBluetoothGatt.close();
    mBluetoothGatt = null;
}
```

原文地址:http://developer.android.com/guide/topics/connectivity/bluetooth-le.html

