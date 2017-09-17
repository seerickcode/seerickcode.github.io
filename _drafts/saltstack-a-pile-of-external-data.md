---
layout: post
title: Saltstack - a Pile of External Data
modified:
categories: devops python 
description:
tags: [saltstack pillar external]
image:
  feature:
  credit:
  creditlink:
comments:
share:
---

Write about :
Getting external data from Confluence Wiki

curl -v -u duhmyusername:isthemagicnumber "http://wiki.puttinarealurlheredude.ca/rest/api/content/33033033?expand=body.storage.value" | python -mjson.tool

content = "the above data"
soup = BeautifulSoup(body,"html.parser")
s = soup.findAll("ac:structured-macro", {'ac:name':'code'})
for x in s:
  title=x.find("ac:parameter", {'ac:name':'title'})
  if title.text == 'monitoring':
    yaml = x.find('ac:plain-text-body').text
    print yaml


