# 扫码应用 Java 开发指导

本文说明第三方 Java Android 应用或服务端如何与本扫码应用集成。扫码应用提供三种独立能力：广播触发扫码、广播接收结果和 Webhook 输出。

## 1. 集成前配置

在扫码应用的“设置 -> 输出转发”中按需开启对应输出通道。

| 能力 | 必要设置 | 默认值 |
| --- | --- | --- |
| 广播触发扫码 | 后台服务已运行，按键触发功能开启 | `com.system.key_f1`、`com.system.key_f2`、`scan_action` |
| 广播接收结果 | 开启“广播输出”，确认 Action 与 Extra Key | Action：`com.android.scanner.broadcast`；Key：`barcode` |

## 2. Java 触发扫码

扫码应用默认监听下列广播 Action。第三方应用可在持有 `Context` 时发送其中之一：

```java
import android.content.Context;
import android.content.Intent;

public final class ScanTrigger {
    public static final String ACTION_SCAN = "com.system.key_f1";

    private ScanTrigger() {}

    public static void trigger(Context context) {
        context.sendBroadcast(new Intent(ACTION_SCAN));
    }
}
```

也可通过 ADB 验证设备侧监听：

```sh
adb shell am broadcast -a com.system.key_f1
adb shell am broadcast -a com.system.key_f2
adb shell am broadcast -a scan_action
```

触发广播只表示已经请求设备开始识读，并不代表已经读取到条码。应通过下一节的结果广播，或 Webhook 服务端记录，确认实际结果。

## 3. Java 接收扫码结果广播

接收方与扫码应用是不同应用，动态注册时必须标记为 `RECEIVER_EXPORTED`，否则 Android 13+ 会拒绝来自其他应用的广播。

### 3.1 依赖与导入

Java 文件导入：

```java
import android.content.BroadcastReceiver;
import android.content.Context;
import android.content.Intent;
import android.content.IntentFilter;
import android.util.Log;
import androidx.core.content.ContextCompat;
```

### 3.2 Activity 示例

```java
public final class ScanActivity extends AppCompatActivity {
    private static final String SCAN_ACTION =
            "com.android.scanner.broadcast";
    private static final String EXTRA_BARCODE = "barcode";

    private final BroadcastReceiver scanReceiver = new BroadcastReceiver() {
        @Override
        public void onReceive(Context context, Intent intent) {
            if (!SCAN_ACTION.equals(intent.getAction())) {
                return;
            }
            String barcode = intent.getStringExtra(EXTRA_BARCODE);
            if (barcode == null || barcode.isEmpty()) {
                return;
            }
            onBarcodeScanned(barcode);
        }
    };

    @Override
    protected void onStart() {
        super.onStart();
        ContextCompat.registerReceiver(
                this,
                scanReceiver,
                new IntentFilter(SCAN_ACTION),
                ContextCompat.RECEIVER_EXPORTED
        );
    }

    @Override
    protected void onStop() {
        unregisterReceiver(scanReceiver);
        super.onStop();
    }

    private void onBarcodeScanned(String barcode) {
        Log.i("ScanIntegration", "barcode=" + barcode);
        // 更新页面、查询商品或提交本应用业务逻辑。
    }
}
```

注意！使用上述API时要打开扫码App设置里面的“按键广播触发扫码”和“广播输出”两个开关。

