+++
title = 'selenium爬虫小记'
tags = ['python', 'crawler']
date =  '2018-03-16T14:22:08+08:00'
+++


刚刚学代码的时候就想写个爬虫，爬一点有意思的东西，也试着爬豆瓣知乎的一些内容，然而在内容值钱的年代，爬虫越来越难，最近接触到selenium自动化测试，想着用这个做爬虫岂不美哉。
<!--more-->

# 知乎话题结构

![zhihu_topic.png](/images/zhihu_topic.png)
首先看一下知乎的话题结构，典型的网状结构，一个话题节点，有多个父节点多个子节点。

# 思路

手动给程序一个启动节点，登录后，程序打开 `https://www.zhihu.com/topic/:id/organize` 页面，记录下该页面的所有的父节点子节点id和title信息，并且标记isDone:0, 方便程序意外中断后，手动恢复后可以继续爬取数据，接着把该话题的父节点已数组的形式存好，标记isDone:1完成该节点的信息爬取，关闭tab，继续下一个话题的爬取

mongoDB数据Model：

``` python
class Topic(Document):
  topicId = IntField(max_length=32,required=True,unique=True)
  title = StringField(max_length=200)
  fatherId = ListField(IntField())
  focusPeople = IntField()
  isDone = IntField(required=True)
  sameTopic = ListField(IntField())
  createdTime = DateTimeField(defualt = datetime.datetime.utcnow())
  updatedTime = DateTimeField(defualt = datetime.datetime.utcnow())
```

# 代码

``` python
# import
from selenium import webdriver 
# from selenium.webdriver.common.keys import Keys
import time
import datetime
from sqlalchemy import Column, String, create_engine
from sqlalchemy.orm import sessionmaker
from sqlalchemy.ext.declarative import declarative_base
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
from mongoengine import *
from Model.Topics import Topic
from selenium.webdriver.common.desired_capabilities import DesiredCapabilities

import sys
sys.setrecursionlimit(1000000) #例如这里设置为一百万 

def getTopicOrg(id):
  return 'https://www.zhihu.com/topic/' + str(id) + '/organize'

def login(driver):
  driver.get("https://www.zhihu.com")
  time.sleep(3)
  driver.find_element_by_xpath('//*[@id="root"]/div/main/div/div/div/div[2]/div[2]/span').click()
  time.sleep(2)

  driver.find_element_by_name('username').send_keys('账户')
  time.sleep(2)
  driver.find_element_by_name('password').send_keys('密码')
  time.sleep(2)

  driver.find_element_by_xpath('//*[@id="root"]/div/main/div/div/div/div[2]/div[1]/form/button').click()
  time.sleep(3)

def findWork():
  topic = Topic.objects(isDone = 0).order_by('createdTime')[0]
  return topic

def startCrawl(driver):
  nowTopic = findWork()
  href = getTopicOrg(nowTopic.topicId)

  # 获取当前窗口句柄
  topics_handle = driver.current_window_handle

  new_tab = 'window.open("' + href + '")'
  driver.execute_script(new_tab)
  now_handles = driver.window_handles
  driver.switch_to.window(now_handles[-1])

  time.sleep(4)

  current_url = driver.current_url

  # 对于已经合并的话题，页面会跳转新的话题
  if current_url.endswith("hot"):
    # get sameTopicId
    sameTopicId = int(current_url.split('/')[4])
    # 标记完成
    nowTopic.isDone = 2

    if sameTopicId not in nowTopic.sameTopic:
      nowTopic.sameTopic.append(sameTopicId)
    nowTopic.updatedTime = datetime.datetime.utcnow()
    nowTopic.save()

    #记录
    newTopicTitle = driver.find_element_by_xpath('//*[@id="root"]/div/main/div/div[1]/div[1]/div[1]/div[1]/div/div[2]/h2/div/h1').get_attribute('innerHTML')
    newTopic = Topic(
      topicId = sameTopicId,
      title = newTopicTitle,
      isDone = 0,
      createdTime = datetime.datetime.utcnow()
    )
    try:
      newTopic.save()
    except:
      print("Error: 已经存在")
    else:
      print("内容写入文件成功")
  else:
    # XPATH
    focusPeople_xpath = '//*[@id="zh-topic-side-head"]/div/a/strong'
    fatherId_xpath = '//*[@id="zh-topic-organize-parent-editor"]/div/a'
    childId_xpath = '//*[@id="zh-topic-organize-child-editor"]/div/a'

    try:
      focusPeople = driver.find_element_by_xpath(focusPeople_xpath).get_attribute('innerHTML')
    except:
      print("当前话题关注人数为0")
      focusPeople = 0
    else:
      print("当前话题关注人数")
      print(focusPeople)

    fatherId_List = driver.find_elements_by_xpath(fatherId_xpath)
    childId_List = driver.find_elements_by_xpath(childId_xpath)

    fatherlist = []
    for fatherId in fatherId_List:
      fatherIdText = fatherId.text
      fatherIdData = fatherId.get_attribute('data-token')
      print(fatherIdText)
      print(fatherIdData)
      # save shuju
      topic = Topic(
        topicId = fatherIdData,
        title = fatherIdText,
        isDone = 0,
        createdTime = datetime.datetime.utcnow()
      )
      try:
        topic.save()
      except:
          print("Error: 已经存在")
      else:
          print("内容写入文件成功")
      fatherlist.append(fatherIdData)

    for childId in childId_List:
      childIdText = childId.text
      childIdData = childId.get_attribute('data-token')
      print(childIdText)
      print(childIdData)

      # save shuju
      topic = Topic(
        topicId = childIdData,
        title = childIdText,
        isDone = 0,
        createdTime = datetime.datetime.utcnow()
      )
      try:
        topic.save()
      except:
          print("Error: 已经存在")
      else:
          print("内容写入文件成功")

    # tag db isDone
    print(fatherlist)
    nowTopic.fatherId = fatherlist
    nowTopic.focusPeople = focusPeople
    nowTopic.isDone = 1
    nowTopic.updatedTime = datetime.datetime.utcnow()
    nowTopic.save()

  # close tag
  driver.close()
  driver.switch_to.window(topics_handle)
  time.sleep(2)

  # iteration operation
  return startCrawl(driver)


def main():
  connect('zhihu', host='localhost', port=27017)

  options = webdriver.ChromeOptions()
  options.add_argument('--disable-infobars')
  driver = webdriver.Chrome('/usr/local/bin/chromedriver', chrome_options=options)
  # Mac下全屏
  driver.fullscreen_window()

  # 登录
  login(driver)

  # 爬虫topic
  startCrawl(driver)
  time.sleep(10)
  options.close()
```

# 洗数据

爬虫的时候因为方便，使用了Mongodb作为数据存储，为了方便接口调用，使用mysql将数据分成两张表，一张存节点的信息，一张存节点之间的关系

``` bash
CREATE TABLE `zhihu_topic` (
  `id` int(11) NOT NULL AUTO_INCREMENT COMMENT '主键id',
  `topicId` int(11) NOT NULL COMMENT '话题id',
  `title` varchar(20) COLLATE utf8mb4_unicode_ci DEFAULT NULL COMMENT '话题标题',
  `focusPeople` int(11) DEFAULT '0' COMMENT '主键id',
  `isMerge` tinyint(3) unsigned DEFAULT '0' COMMENT '是否合并',
  `mergeId` int(11) DEFAULT NULL COMMENT '新话题id',
  `created_at` datetime DEFAULT CURRENT_TIMESTAMP,
  `updated_at` datetime DEFAULT NULL ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  UNIQUE KEY `topicId_UNIQUE` (`topicId`),
  KEY `topicId` (`topicId`),
  KEY `title` (`title`)
) ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

``` bash
CREATE TABLE `zhihu_topic_link` (
  `id` int(11) NOT NULL AUTO_INCREMENT COMMENT '主键id',
  `fatherId` int(11) NOT NULL COMMENT '父话题id',
  `childId` int(11) NOT NULL COMMENT '子话题id',
  `created_at` datetime DEFAULT CURRENT_TIMESTAMP,
  `updated_at` datetime DEFAULT NULL ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  UNIQUE KEY `fatherId` (`fatherId`,`childId`),
  KEY `fatherId_2` (`fatherId`),
  KEY `childId` (`childId`)
) ENGINE=InnoDB AUTO_INCREMENT=74256 DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

# Echart数据可视化

![zhihu_echart.png](/images/zhihu_echart.png)
使用了Echart的力导向图，作为前端展示，需要注意的是每个node节点的id是不能重复的，一定要对数据做好去重工作。

# 修改默认递归深度

程序跑了一天发现地动断开了，发现python默认的递归深度是很有限的，查了资料大概是1000多的样子，当递归深度超过这个值的时候，会引发异常。

``` python
import sys
sys.setrecursionlimit(1000000)
```

# 缺点

- 理论上，知乎的话题都是类似于节点的，无论从哪一个入口爬都应该能爬完所有的话题，但是目前爬到4万多个话题就会找不到新的话题，目前正在找
- 子话题太多的情况没有考虑，一个话题如果它的子话题太多的话页面上是不显示的，所以导致部分话题没有爬完，可能导致上面的问题，需要优化。

# 感受

Selenium本来是用来自动化测试的，无奈知乎对爬虫太严厉，传统的requet爬数据太难了，Selenium被用来爬虫也是很酷的，虽然速读慢一点，但是降低了开发难度。