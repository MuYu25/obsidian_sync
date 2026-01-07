# 需要更改的地方
### 原有封禁上需要新增逻辑，主要是加上微信的封禁逻辑
#### logic/tags.go
- SetAccountTags 为用户设置标签(永久)
- SetAccountTagsAndExpirationTime 设置标签并且设置过期时间
- BatchSetAccountTags 批量为用户设置标签
- MarkHighNegativeIncomeUsers 批量标记高负收益的用户
- MarkAdCheatUsers 批量标记广告作弊的用户
- MarkAdCheatUsers2 批量标记广告作弊的用户（banner广告价值异常）
- MarkForbiddenUser 禁止用户登录
- BatchMarkForbiddenUser 批量禁止用户登录
#### 存在封禁逻辑的地方
##### ReportLowValue 客户端上报低价值用户
> if 如果用户在白名单中，不再设置标签封禁逻辑:  return
> 计算用户的盈亏，如果盈利用户就不封禁，如果亏损用户就封禁.
> 调用MarkForbiddenUser
##### EventPush 客户端事件推送
> 若属于停留事件，进行实时消费
> 若属于广告事件
> 	属于消费广告展示事件
> 		判断用户是否达到高价值活动开启阈值
> 			判断用户是否达到高价值活动开启阈值
> 			使用的广告位Id是否存在广告id库
> 				if 不是广告id库中的广告位id:
> 					MarkForbiddenUser
> 			记录用户 激励视频 广告收益到缓存中，用户cv评分的计算
##### DeviceReport 处理设备上报逻辑 // 仅查询封禁用户
> 根据Oaid判断是否为已存在用户
> 	if 查找不到用户:
> 		if 检查当天此IP注册用户数量 >= 8: return 是否超出限制，排除白名单
> 		if 账号封禁中: 
> 			if 封禁时间 <= cur: 解封
> 			else: 返回封禁时间
> 		新用户或今日首次登录的用户，活跃天数+1
> 		更新用户状态,最后登录ip、版本号、时间、渠道、token、累计活跃天数等
> 		保存登录日志
> 		按设备id保存最新上报的设备信息
> 		go: 
> 			if 是新用户，对其进行来源归因
> 				广告归因
> 				归因成功后直接进行激活回调 (通常情况下，希望用户打开APP就通知广告媒体平台该用户已经激活)
> 			// 判断是否需要上报次日留存事件（仅限应用宝、广点通渠道包）
### 解封中新增逻辑，主要是加上微信的解封逻辑
#### logic/tags.go
- RemoveAccountTags 移除用户标签
- BatchRemoveAccountTags 批量移除用户标签
- UnForbidUserByClient 定时解封属于客户端上报封禁的用
- RemoveExpiredTagsTask 定时移除有过期时间的标签