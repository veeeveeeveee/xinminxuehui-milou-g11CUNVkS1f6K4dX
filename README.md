
### **背景：**


我们需求是显示Date Time类型的Time信息，比如我们想要在report中基于Hour Of Created Date进行分组，从而想要了解到一段时间内什么时间是数据创建的高峰期，不同的running user可能时区不同，比如中国时区是GMT\+8，日本的时区是GMT\+9，美国可能不同的州对应的时区也不同，而且涉及到冬令时夏令时问题。目前的痛点是formula中的函数只能显示GMT时间，无法做到显示 user locale的时间，那么应该如何设计才可以保证不同运行的人可以显示不同时区的时间信息呢？


这里我们需要考虑几个问题点：


1. 如何获取当前用户的时区以及相对GMT的偏移量？比如中国时区是GMT\+8，代码上如何知道他想对GMT时间是\+8的？
2. 如果你的运行用户有欧美国家的时区，需要考虑冬令时和夏令时。欧美很多国家都有都有冬令时和夏令时的概念，我们以纽约为例，中国大陆所有地方的时区是GMT \+ 8, 纽约的夏令时时区是GMT\-4，也就是说在每年的3月到11月，如果GMT时间是 12:00:00，则如果你的user时区是China Standard Time，则时间显示 20:00:00， 如果你的user时区是America/New\_York，则时间显示8:00:00。然而等到了11月的某个时间节点以后，New\_York的GMT便会变成GMT\-5，同样的GMT时间，则时间显示成 7:00:00。
3. 我们看report的数据可能跨年。举个例子，当前的时间如果是 25年1月，我所看的数据除了25年的数据，还可能看到24年的数据甚至23年的数据，如何显示不同年的数据？


### **分析：**


针对这种需求情况，我们可能需要去看一下我们的org，然后确定几个点：


1\. 当前的org针对当前需求的user有几个TimeZone。可以简单run一个sql，根据需求增加相关的filter：比如下面的结果，我们有两个时区。Asis/Shanghai 以及 America/New\_York




```
SELECT TimeZoneSidKey, count(Id)
from user
where TimeZoneSidKey != null
and isactive = true
and Profile.UserLicense.Name != 'Guest User License'
group by TimeZoneSidKey
```


![](https://img2024.cnblogs.com/blog/910966/202412/910966-20241204231301232-1222547560.png)


2\.  判断当前Timezone的时区信息以及偏移量，这里我们可以进行很简单的方式来确定，即将admin的user切换成相关的时区，然后运行脚本就可以了，我们以一下面的代码来进行简单的确定偏移量。




```
Integer year = 2024;
Set offsetSet = new Set();
for(Integer month = 1; month<= 12; month++) {
  Date dt1 = Date.newInstance(year, month,1);
  Date dt2;
  if(month != 12) {
    dt2 = Date.newInstance(year, month + 1, 1);
  } else {
    dt2 = Date.newInstance(year + 1, 1, 1);
  }

  for(Integer day = 1; day <= dt1.daysBetween(dt2); day++) {
    Datetime gmtDate = Datetime.newInstanceGMT(year, month, day, 0,0,0);
    Datetime userLocaleDate = Datetime.newInstance(year,month,day,0,0,0);
    offsetSet.add((gmtDate.getTime() - userLocaleDate.getTime()) / (1000 * 60 * 60));
  }
}
system.debug(offsetSet);
```


我们如果以GMT \+ 8的时区来运行，则显示 8， 如果以GMT\-4的New York Time来运行，则结果为 \-4 \-5（如果超过一个，说明包含DST时间）


![](https://img2024.cnblogs.com/blog/910966/202412/910966-20241225194551378-1492959126.png)


![](https://img2024.cnblogs.com/blog/910966/202412/910966-20241225194815591-594785610.png)


3\. 确定数据需要显示的年限范围。比如我们针对当前的表，只显示当前的年和显示近两年以及显示任何历史数据的解决方案都是不同的。在出具解决方案以前，我们需要确定一下用户需要查看的年限范围从而更好的设计。


### **解决方案**


针对上面的三个分析的点，我们可以构建不同的解决方案，不同的分析结果会导向不同的解决方案，以下仅是个人的建议。




| **当前有几个timezone** | **timezone中是否包含DST** | **用户所需要看到的年限范围** | **解决方案** |
| --- | --- | --- | --- |
| 1\-4个 | 不包含 | 任意 | 在user上创建一个Number的字段用来存储偏移量，维护一下 timezone \-\> offset的关系，无论是hard code还是custom metadata / setting，然后user创建时更新当前字段或者对历史用户进行刷新。 |
| 很多，比如超过5个 | 不包含 | 任意 | 在user上创建一个Number的字段用来存储偏移量，通过login flow \+  apex class获取当前user的偏移量信息，然后存储到当前字段。后续的formula可以通过加上当前偏移量进行解决。 |
| 1\-4个 | 包含 | 1\-3年 | 创建 custom metadata，运行脚本设置timezone以及它对应的夏令时 start date 以及end date 以及冬夏的偏移量和年份，然后formula基于年份以及 start/end date去确认偏移量来动态展示。 |
| 很多，比如超过5个 | 包含 | 1\-3年 | 创建 custom metadata，通过login flow \+ apex class获取当前user的偏移量信息以及timezone，然后动态维护custom metadata信息，然后formula基于年份以及start/end date去确认偏移量来动态展示。 |
| 1\-4个 | 包含 | 任意 | 同上，但是有 formula编译失败风险，目前没想到更好的解决方案 |


 


给出解决方案以后相信开发人员可以基于解决方案进行开发，当前的博客只针对一个场景进行详细的介绍，我们选取包含 DST并且两个timezone以及用户想要看到去年和今年的数据的场景进行详细的展开。


### 实践


**需求和问题汇总**


Account上面有一个Datetime类型字段，名称为Test Date Time，还有两个formula字段，一个是使用TIMEVALUE函数返回当前的Time类型的字段，名称为 Time Of Test Date Time， 一个是使用 HOUR函数返回当前Time的Hour信息，名称为 Hour Of Test Date Time。 我们看一下数据展示。


1\. 通过GMT \+ 8中国账号演示效果，通过下方GIF我们可以看出 Test Date Time字段的Time信息 减去8小时即为 Time Of Test Date Time, Hour 和 Time 两个formula字段均显示GMT时间，8为当前user的locale time偏移量。


![](https://img2024.cnblogs.com/blog/910966/202501/910966-20250108182542934-1049204434.gif)


2\. 通过 GMT\-4/5 纽约账号登陆显示效果。我们通过下方截图可以看到，Test Date Time是8点情况下，10月的Time Of Test Date Time的差值是 \-4，然而11月确实\-5\. 原因是纽约11月初进入冬令时，user locale time从 GMT\-4 变成了 GMT\-5。


![](https://img2024.cnblogs.com/blog/910966/202501/910966-20250108194435901-1406566896.png)


 我们这个demo需要完成的就是两个：


1\. 针对中国区的客户访问report时可以显示中国区的user locale时间，针对美国区的客户访问report时可以显示美国区的user locale时间；


2\. 针对美国区的客户访问report时，需要适配Daylight Saving Time（冬令时夏令时匹配）


**实施**


**1\. 创建custom metadata并且维护数据。**


* Summer Start Date: 记录夏令时开始时间
* Summer End Date: 记录夏令时结束时间
* Summer Offset： 记录夏令时用户区域的时间偏移量
* Winter Offset：记录冬令时用户区域的时间偏移量
* Year: 记录当前数据的年限


![](https://img2024.cnblogs.com/blog/910966/202501/910966-20250108222948421-288294501.png)


 我们上图中的所维护的数据如何填写或者如何实现的呢？我们分析2中有一个脚本可以确定user的timezone有几个用户区域的偏移量，如果结果只有一个，设置当年的Start Date为1月1日并且End Date为12月31日。 如果包含两个结果，可以运行这个脚本：




```
Integer year = 2025;

String result = '\nyear: ' + String.valueOf(year);

Datetime summerDatetime = Datetime.newInstance(year, 6, 1, 12, 0 , 0);
Integer summerHourOffset = summerDatetime.Hour() - summerDatetime.hourGMT();

result += '\nSummer Offset : ' + summerHourOffset;

Datetime winterDatetime = Datetime.newInstance(year, 12, 1, 12 , 0 ,0 );
Integer winterHourOffset = winterDatetime.Hour() - winterDatetime.hourGMT();
result += ' \n winter Offset : ' + winterHourOffset;

for(Integer i = 1; i <=31; i++) {
  Datetime marchDatetime = Datetime.newInstance(year, 3, i, 12, 0, 0);
  Integer currentDateOffset = marchDatetime.Hour() - marchDatetime.hourGMT();
  if(currentDateOffset != winterHourOffset) {
    result += '\n summer start time: ' + marchDatetime.date();
    break;
  }
}
for(Integer i = 1; i <=30; i++) {
  Datetime marchDatetime = Datetime.newInstance(year, 11, i, 12, 0, 0);
  Integer currentDateOffset = marchDatetime.Hour() - marchDatetime.hourGMT();
  if(currentDateOffset != summerHourOffset) {
    result += '\n summer end time: ' + (marchDatetime.date() - 1);
    break;
  }
}

result += ' \n time zone: ' + UserInfo.getTimeZone().toString();

system.debug(result);
```



> 3月作为冬令时的结束，11月作为冬令时的开始，只需要验证这两个月份即可。


**2\. 构建Formula字段**


Offset Key: 将 User Local Time 以及Datetime字段绑定起来，获取指定模式下的字符串，用于后续逻辑使用。




```
TEXT($User.TimeZoneSidKey) + '_' + TEXT(YEAR( DATEVALUE(Test_Date_Time__c)))
```


GMT Offset: 记录 当前user locale time和GMT Time的偏移量。因为这里我们已经知道了CN没有时区，所以直接获取Summer Offset即可，好处是可以省编译的大小。




```
CASE(
  Offset_Key__c , 
  'America/New_York_2024', 
  IF(
    AND(
      DATEVALUE(Test_Date_Time__c) >=  $CustomMetadata.Offset_Mapping__mdt.NY_2024.Summer_Start_Date__c ,
      DATEVALUE(Test_Date_Time__c) <=  $CustomMetadata.Offset_Mapping__mdt.NY_2024.Summer_End_Date__c 
    ),
    $CustomMetadata.Offset_Mapping__mdt.NY_2024.Summer_Offset__c,
    $CustomMetadata.Offset_Mapping__mdt.NY_2024.Winter_Offset__c
  ),
  'America/New_York_2025', 
  IF(
    AND(
      DATEVALUE(Test_Date_Time__c) >=  $CustomMetadata.Offset_Mapping__mdt.NY_2025.Summer_Start_Date__c ,
      DATEVALUE(Test_Date_Time__c) <=  $CustomMetadata.Offset_Mapping__mdt.NY_2025.Summer_End_Date__c 
    ),
    $CustomMetadata.Offset_Mapping__mdt.NY_2025.Summer_Offset__c,
    $CustomMetadata.Offset_Mapping__mdt.NY_2025.Winter_Offset__c
  ),
  'Asia/Shanghai_2024', 
  $CustomMetadata.Offset_Mapping__mdt.CN_2024.Summer_Offset__c,
  'Asia/Shanghai_2025', 
  $CustomMetadata.Offset_Mapping__mdt.CN_2025.Summer_Offset__c,
  null
)
```


Time Of Test Date Time:  使用TimeValue函数基础上，增加偏移量从而返回User Locale Time.




```
TIMEVALUE( Test_Date_Time__c ) +  GMT_Offset__c * 60 * 60 * 1000
```


Hour Of Test Date Time: 直接使用Hour函数，返回time中的hour时间从而实现local time显示。




```
HOUR( Time_Of_Test_Date_Time__c )
```


**效果展示：**


当用户时区为中国时区，可以看到Time和Test Date Time 中的Time信息相同


**![](https://img2024.cnblogs.com/blog/910966/202501/910966-20250109175632453-77766601.gif)**


当用户为美国时区，可以看到Time和Test Date Time 中的Time信息相同


![](https://img2024.cnblogs.com/blog/910966/202501/910966-20250109175822863-213968169.gif)


**总结：**篇中主要介绍当我们使用Formula字段想要显示 Local Time并且可能涉及到多个区域以及东夏令时场景下的解决方案以及指定个例的实施。篇中有错误地方欢迎指出，有不懂欢迎留言。你们项目中遇到这类需求的时候有什么更好的解决方案吗？ 欢迎留言一起讨论和学习。


 本博客参考[milou加速器](https://jiechuangmoxing.com)。转载请注明出处！
