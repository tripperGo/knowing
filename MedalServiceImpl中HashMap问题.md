MedalServiceImpl类中HashMap一共有6个地方出现。

 

#### 第一处 382行

​	这里是将该家庭下本周所有收到的勋章记录进行散列处理，目的是为了提升按照日期（周几）查询勋章的效率。	

​	HashMap长度最大为7，以本周为例，周一是20200427，键为：日期（20200427、20200428、20200429、20200430、20200501、20200502、20200503）。值为当天收到的勋章集合，每天的勋章集合长度最大为6（全部一共6枚勋章，一天最多收到6枚）。

```java
// 使用in操作，一次查出本家庭本周所有勋章记录。有序列表，最终结果按照这个List排序
List<FamilyMedal> familyMedalList = medalDao.listFamilyMedalInSendDate(familyId, dateList);

Map<String, List<FamilyMedal>> familyMedalMap = new HashMap<>();

// 相关勋章信息
List<Integer> medalIds = new ArrayList<>();
if (familyMedalList != null) {
    familyMedalList.forEach(familyMedal -> {
        String sendDate = familyMedal.getSendDate();
        medalIds.add(familyMedal.getMedalId());
        if (StringUtils.isNotEmpty(sendDate)) {
            List<FamilyMedal> familyMedalListTemp = familyMedalMap.get(sendDate);
            if (familyMedalListTemp == null) {
                familyMedalListTemp = new ArrayList<>();
                familyMedalListTemp.add(familyMedal);
                familyMedalMap.put(sendDate, familyMedalListTemp);
            } else {
                familyMedalListTemp.add(familyMedal);
                familyMedalMap.put(sendDate, familyMedalListTemp);
            }
        }
    });
}
```

#### 第二处 403行

这里是把全部勋章（6枚）进行散列存储，键为勋章ID，值为勋章对象，长度最大为6。

```java
Map<Integer, MedalVO> medalVOMap = new HashMap<>();
if (!medalIds.isEmpty()) {
    List<MedalVO> medalList = medalDao.listMedalInIds(medalIds);
    if (medalList != null && !medalList.isEmpty()) {
        medalList.forEach(medal -> medalVOMap.put(medal.getId(), medal));
    }
}
```

#### 第三处 411行

这里将本周的抽奖记录进行散列存储，键为日期（周几），值为当日抽奖记录对象，长度最大为7。

```java
Map<String, MedalAward> medalAwardMap = new HashMap<>();
List<MedalAward> medalAwardList = medalDao.listMedalAwardInAwardDates(familyId, dateList);
if (medalAwardList != null && !medalAwardList.isEmpty()) {
    medalAwardList.forEach(medalAward -> {
        String awardDate = medalAward.getAwardDate();
        if (StringUtils.isNotEmpty(awardDate)) {
            medalAwardMap.put(awardDate, medalAward);
        }
    });
}
```

#### 第四处 481行

这里是返回值的格式化处理，长度固定为4，这里不会出现问题。

```java
Map<String, Object> resultMap = new HashMap<>(4);
resultMap.put("medals", resultList);

// 宝宝头像
Map<String, Object> babyMap = userService.findBabyInfo(familyId);
if (babyMap != null) {
    resultMap.put("babyImg", babyMap.get("baby_img") == null ? "" : babyMap.get("baby_img"));
    resultMap.put("babySex", babyMap.get("baby_sex"));
} else {
    resultMap.put("babyImg", "");
    resultMap.put("babySex", "");
}

result.setData(resultMap);
return result;
```

#### 第五处 581行

这是我自己的测试代码，有一个抽奖算法，在后台可以配置奖品的权重，比如三个奖品（奖品A：0.3；奖品B：0.5；奖品C：0.2），那么这个算法每次抽奖应该有50%概率抽到奖品B。

这段测试代码就是模拟1000或者2000次抽奖，记录下抽到奖品A、奖品B、奖品C的次数，然后汇总看看权重是否生效。这段代码除了我用过其他地方不会有使用，这里不会出现问题。

```java
@Override
public Result test(Integer count) {
    Result result = new Result();
    List<Award> awardList = medalDao.listAwardByType("XZCJ");
    Map<Integer, Integer> countMap = new HashMap<>(2048);
    for (int i = 0; i < count; i++) {
        Award award = weightRandom(awardList);
        Integer id = award.getId();
        Integer countId = countMap.get(id);
        if (countId == null) {
            countMap.put(id, 1);
        } else {
            countMap.put(id, countId + 1);
        }
    }
    result.setData(countMap);
    return result;
}
```

#### 第六处 661行

这里和第四处一样，将每天的返回值格式化一下，长度固定为4，不会出现问题。

```java
Map<String, Object> mondayMap = new HashMap<>(4);
mondayMap.put(CommonConstants.MEDAL_AWARD_STATUS, 0);
List<MedalVO> oneDayMedalList = new ArrayList<>(8);

if (familyMedalList != null) {
    // 获取抽奖资格
    familyMedalList.forEach(familyMedal -> {
        String sendDate = familyMedal.getSendDate();
        Integer medalId = familyMedal.getMedalId();
        if (StringUtils.isNotEmpty(sendDate) && sendDate.equals(DateUtils.getOneDayByWeek(CommonConstants.DATE_FORMAT_MEDAL, weekNum))) {
            MedalVO medal = medalVOMap.get(medalId);
            if (medal != null) {
                MedalVO medalVOTemp = new MedalVO();
                BeanUtils.copyProperties(medal, medalVOTemp);
                if (medalAward != null && medalAward.getStatus() != null && canAward(medalAward.getFamilyId(), medalAward.getAwardDate())) {
                    mondayMap.put(CommonConstants.MEDAL_AWARD_STATUS, medalAward.getStatus());
                }
                oneDayMedalList.add(medalVOTemp);
            }
        }
    });
}
mondayMap.put(CommonConstants.LIST, oneDayMedalList);
mondayMap.put(CommonConstants.MEDAL_DAY, weekStr);
return mondayMap;
```

