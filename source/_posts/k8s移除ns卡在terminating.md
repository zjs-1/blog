---
layout: title
title: k8s移除ns卡在terminating
date: 2021-06-16 17:01:13
tags: [运维]
categories: 技术
---

> 参考链接: [https://cloud.tencent.com/developer/article/1802531](https://cloud.tencent.com/developer/article/1802531)


1. 先排查确定是不是有资源没有删干净
2. 确定完之后，很可能是namespace的spec.finalizers不为空，可以使用 kubectl get namespace <name> -o yaml  看到原因。如果确定是这个原因，可以用下面的方法
```bash
# 导出namespace放到tmp.json中
kubectl get namespace ${namespace} -o json > tmp.json

# 编辑tmp.json，删除里面的spec.finalizers的内容

kubectl proxy &
PID=$!
curl -X PUT http://localhost:8001/api/v1/namespaces/${namespace}/finalize -H "Content-Type: application/json" --data-binary @tmp.json
kill $PID
```

