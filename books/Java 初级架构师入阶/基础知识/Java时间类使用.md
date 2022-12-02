---
group:
  title: 基础知识
  order: 2
order: 2
---
# Java时间类使用

## Clock
### 详解
可以取代`System.currentTimeMillis()`,时区敏感，带有时区信息
### 用法
```
Clock clock = Clock.systemDefaultZone();
long millis = clock.millis();

Instant instant = clock.instant();
Date legacyDate = Date.from(instant);   // legacy java.util.Date
```
## ZoneId
### 详解
新的时区类 `java.time.ZoneId` 是原有的 `java.util.TimeZone` 类的替代品。 ZoneId对象可以通过 `ZoneId.of()` 方法创建，也可以通过 `ZoneId.systemDefault()` 获取系统默认时区。
### 用法
```
System.out.println(ZoneId.getAvailableZoneIds());
// prints all available timezone ids

ZoneId shanghaiZoneId = ZoneId.of("Asia/Shanghai");
// ZoneRules[currentStandardOffset=+08:00]
```
有了 ZoneId，我们就可以将一个 LocalDate、LocalTime 或 LocalDateTime 对象转化为 ZonedDateTime 对象
```
LocalDateTime localDateTime = LocalDateTime.now();
ZonedDateTime zonedDateTime = ZonedDateTime.of(localDateTime, shanghaiZoneId);
```
ZonedDateTime 对象由两部分构成，LocalDateTime 和 ZoneId，其中 2018-03-03T15:26:56.147 部分为 LocalDateTime，+08:00[Asia/Shanghai] 部分为ZoneId。
## LocalTime
### 详解
LocalTime类是Java 8中日期时间功能里表示一整天中某个时间点的类，它的时间是无时区属性的（早上10点等等）
### 用法
```
LocalTime now1 = LocalTime.now(zone1);
LocalTime now2 = LocalTime.now(zone2);

System.out.println(now1.isBefore(now2));  // false

long hoursBetween = ChronoUnit.HOURS.between(now1, now2);
long minutesBetween = ChronoUnit.MINUTES.between(now1, now2);

System.out.println(hoursBetween);       // -3
System.out.println(minutesBetween);     // -239
```
## LocalDate
### 详解
LocalDate类是Java 8中日期时间功能里表示一个本地日期的类，它的日期是无时区属性的。 可以用来表示生日、节假日期等等。这个类用于表示一个确切的日期，而不是这个日期所在的时间
### 用法
```
LocalDate today = LocalDate.now();
LocalDate tomorrow = today.plus(1, ChronoUnit.DAYS);
LocalDate yesterday = tomorrow.minusDays(2);

LocalDate independenceDay = LocalDate.of(2014, Month.JULY, 4);
DayOfWeek dayOfWeek = independenceDay.getDayOfWeek();
System.out.println(dayOfWeek);    // FRIDAY
```
## LocalDateTime
### 详解
LocalDateTime类是Java 8中日期时间功能里，用于表示当地的日期与时间的类，它的值是无时区属性的。你可以将其视为Java 8中LocalDate与LocalTime两个类的结合。
### 用法
```
LocalDateTime sylvester = LocalDateTime.of(2014, Month.DECEMBER, 31, 23, 59, 59);

DayOfWeek dayOfWeek = sylvester.getDayOfWeek();
System.out.println(dayOfWeek);      // WEDNESDAY

Month month = sylvester.getMonth();
System.out.println(month);          // DECEMBER

long minuteOfDay = sylvester.getLong(ChronoField.MINUTE_OF_DAY);
System.out.println(minuteOfDay);    // 1439
```
## ZonedDateTime
### 详解
ZonedDateTime类是Java 8中日期时间功能里，用于表示带时区的日期与时间信息的类。可以用于表示一个真实事件的开始时间，如某火箭升空时间等等。
### 用法
```
ZonedDateTime dateTime = ZonedDateTime.now();
ZonedDateTime zonedDateTime = ZonedDateTime.of(LocalDateTime.now(), shanghaiZoneId);
```
转时间戳

```
ZonedDateTime zonedDateTime = ZonedDateTime.parse("2022-10-15T02:55:38.776Z");
System.out.println(zonedDateTime.toInstant().toEpochMilli());
```



## Duration

### 详解
一个Duration对象表示两个Instant间的一段时间
### 用法
```
Instant first = Instant.now();
// wait some time while something happens
Instant second = Instant.now();
Duration duration = Duration.between(first, second);
```
## DateTimeFormatter
### 详解
DateTimeFormatter类是Java 8中日期时间功能里，线程安全。用于解析和格式化日期时间的类
。类中包含如下预定义的实例：
```
BASIC_ISO_DATE

ISO_LOCAL_DATE
ISO_LOCAL_TIME
ISO_LOCAL_DATE_TIME

ISO_OFFSET_DATE
ISO_OFFSET_TIME
ISO_OFFSET_DATE_TIME

ISO_ZONED_DATE_TIME

ISO_INSTANT

ISO_DATE
ISO_TIME
ISO_DATE_TIME

ISO_ORDINAL_TIME
ISO_WEEK_DATE

RFC_1123_DATE_TIME
```
### 用法
```
DateTimeFormatter formatter = DateTimeFormatter.ISO_DATE_TIME;
//DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
String formattedDate = formatter.format(LocalDateTime.now());
```
## 参考
1. [java8-tutorial#date-api](https://github.com/winterbe/java8-tutorial#date-api)
2. [java8-datetime-api](https://github.com/biezhi/learn-java8/blob/master/java8-datetime-api/README.md)