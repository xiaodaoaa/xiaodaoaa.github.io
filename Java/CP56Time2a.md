`CP56Time2a` 是在IEC 60870-5-104协议中使用的一种时间戳格式，表示时间的精确性为毫秒级。它通常用于传输电力系统中的时间信息。`CP56Time2a` 格式使用了7个字节来表示时间。

要将 `CP56Time2a` 格式转换为 Java 中的 `Date` 对象，你可以按照以下步骤进行解析：

1. **解析各个字段**：
   - 字节 1：毫秒的低位部分 (0-7 bit 表示毫秒，8 bit 表示是否为夏令时)
   - 字节 2：毫秒的高位部分
   - 字节 3：分钟（0-5 bit），最高位表示是否无效 (bit 7)
   - 字节 4：小时（0-4 bit），最高位表示是否无效 (bit 6)
   - 字节 5：日期（0-4 bit），最高位表示是否无效 (bit 5)
   - 字节 6：月份（0-3 bit），最高位表示是否无效 (bit 4)
   - 字节 7：年（最后两位，0-6 bit）

2. **组合字段**：
   - 解析后，将这些字段组合成一个时间戳，然后转换为 `Date` 对象。

这里是一个示例代码：

```java
public class CP56Time2aParser {
    /**
     * 时标CP56Time2a解析
     *
     * @param b 时标CP56Time2a（长度为7 的byte数组）
     * @return 解析结果
     */
    public static Date CP56Time2a2Date(byte b[]) {

        StringBuilder result = new StringBuilder();

        int year = b[6] & 0x7F;
        int month = b[5] & 0x0F;
        int day = b[4] & 0x1F;
        int week = (b[4] & 0xE0) / 32;
        int hour = b[3] & 0x1F;
        int minute = b[2] & 0x3F;
        int second = ((b[1]&0xFF) << 8) + (b[0]&0xFF);

        result.append("20");
        result.append(year).append("-");
        result.append(String.format("%02d", month)).append("-");
        result.append(String.format("%02d", day)).append(" ");
        result.append(hour).append(":").append(minute).append(":");
        result.append(second / 1000).append("\n");

        SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss");
        Date date = null;
        try { date = sdf.parse(result.toString()); } catch (Exception e) { e.printStackTrace(); }
        return date;
    }

    public static byte[] Date2CP56Time2a(Date date) {

        Calendar calendar = Calendar.getInstance();
        calendar.setTime(date);
        StringBuilder builder = new StringBuilder();
        String milliSecond = String.format("%04X", (calendar.get(Calendar.SECOND) * 1000) + calendar.get(Calendar.MILLISECOND));
        builder.append(milliSecond.substring(2, 4));
        builder.append(milliSecond.substring(0, 2));
        builder.append(String.format("%02X", calendar.get(Calendar.MINUTE) & 0x3F));
        builder.append(String.format("%02X", calendar.get(Calendar.HOUR_OF_DAY) & 0x1F));
        int week = calendar.get(Calendar.DAY_OF_WEEK);
        if (week == Calendar.SUNDAY)
            week = 7;
        else week--;
        builder.append(String.format("%02X", (week << 5) + (calendar.get(Calendar.DAY_OF_MONTH) & 0x1F)));
        builder.append(String.format("%02X", calendar.get(Calendar.MONTH) + 1));
        builder.append(String.format("%02X", calendar.get(Calendar.YEAR) - 2000));
        return HexUtil.decodeHex(builder.toString());
    }

    public static void main(String[] args) {
        byte[] time = ProtocolUtil.Date2CP56Time2a(new Date());
        log.info("{}", Arrays.toString(time));

        byte[] byteTime = HexUtil.decodeHex("A2E90D0E880818");
        Date date = ProtocolUtil.CP56Time2a2Date(byteTime);
        System.out.println(date);//Thu Aug 08 14:13:59 CST 2024
    }
}
```

### 说明：
- `millisecond` 是通过组合第一个和第二个字节得到的。
- `minute`、`hour`、`day`、`month`、`year` 都是从对应字节中提取的。
- 使用 `GregorianCalendar` 创建日期对象，然后将其转换为 `Date`。

你可以使用这个代码将 `CP56Time2a` 时间戳转换为 `Date` 对象。