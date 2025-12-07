---
description: 本篇文档系统复盘了 Remark 项目在真实生产数据量级下遇到的严重报表性能问题（N+1 查询、内存爆炸、ES 查询不当、大数据量 Excel 导出导致 OOM 等），并详细阐述了优化过程中的一系列工程实践：包括宽表设计、预计算、批量查询、ES 聚合、流式处理、缓存策略、数据下沉和架构调整。通过两轮优化，全量报表生成从“小时级/OOM 崩溃”成功降至“数分钟内完成”，实现稳定性与性能的全面提升。文章为大数据量系统性能治理提供了可复用的最佳实践与方法论指引。
title:  从崩溃到秒级响应：Remark 报表系统性能优化实战总结
tag:
  - 工作
  - JAVA
sidebar: true
comment: true
recommend: 1
---
# 从“崩溃报表”到“秒级响应”：Remark项目报表系统优化实践

## 项目背景

- Remark项目包含约20个核心业务报表。在开发与测试阶段，由于数据量极小，所有报表功能均表现正常。然而，在接入生产环境的全量数据（即数据迁移后）后，系统遭遇了严重的性能瓶颈，**超过60%的报表生成时间超过1小时，部分复杂报表直接导致应用崩溃。**

### 问题发现

- **性能极度低下**：多个核心报表生成耗时长达数小时
- **系统资源耗尽**：并发报表生成时，系统直接崩溃，资源被迅速榨干
- **用户体验恶劣**：用户请求长期处于等待状态

## 根本原因分析

- 技术层面
  1. **缺乏缓存策略**：每次查询均直接访问底层数据库，未能复用先前结果，造成大量重复的I/O开销与数据库连接。
  2. **存在N+1查询问题**：数据组装逻辑低效，为构造单行报告数据，需循环发起多次离散的数据库查询，导致查询次数增加。
  3. **内存计算过载**：将大量原始数据拉取至应用层进行聚合与对象封装，产生海量瞬时对象，导致应用服务器内存飙升，引发OOM（内存溢出）。
  4. **全量数据导出**：采用“全量查询→内存组装→一次性写入”的Excel生成策略，数据量过大时，极易耗尽JVM堆内存，引发OOM（内存溢出）服务崩溃。
- 流程与管理层面
  1. **流程保障缺失**：在“快”的业务压力下，一天要完成两个报表的KT和提测，导致对非功能需求（如大数据量下的性能与响应时间）的考量被完全牺牲。

## 案例审视

### **Case 1：保单明细报表 - N+1查询与内存爆炸**

**问题代码逻辑：**

```
// 伪代码逻辑
//1.查询并转换数据为Report元数据
listAndConvert2ReportItemDTO()
//2.将数据插入到Excel报表中并生成PgReport
createReport()
//3.发送邮件
sendEmail()

List<Policy> policies = selectPolicies(); // ES查询后再查询保单，获取30万条保单
List<ReportDTO> dtos = policies.collect(policy -> {
    BcpPolicyBills bills = bcpBizInterface.getPolicyBills(policy.policyId);     // 第1次N+1查询
    List<BankFileRec> bankRecs = bankFileService.getBankFileRecsInfoByRefNo1(...); 
    // 第2次N+1查询
    Package aPackage = packageService.findByPackageCode(...); 
    PackageMarketing packageMarketing = packageMarketingService.findByPackageId(...); 
    // 第3次N+1查询
    Date cancelEndoDate = endoService.getCancelEndo(...);;       // 第4次N+1查询
    // ...
    // 最后，组装一个DTO
    return assembleDTO(policy, bills, bankRecs, aPackage, endoList, ...);
});
```

**性能瓶颈：**

- **内存对象爆炸**：在内存中同时持有30万个`Policy`对象，极易引发OOM。
- **查询放大效应**：假设主查询返回30万条`Policy`记录，没有引发OOM，上述代码最少会产生 `30W * 4 ≈ 120W` 次数据库查询。

#### 第一次**解决方案与优化实践**：

**优化策略：数据分层、空间换时间、减少交互**

- **流式处理**：对于大数据量导出，摒弃全量加载，一行一行地写入Excel，保持内存中只有少量数据。

- **构建宽表**：将关联逻辑和聚合逻辑下沉到宽表中，报表直接查询高度优化后的宽表，查询复杂度从O(n)降至O(1)。

- **引入缓存**：对静态数据（配置数据，码表）使用内存缓存，设置合理的过期时间。

- **批量查询**：将循环内的单个查询，改为先收集所有ID，然后通过`IN`语句进行批量查询。

  

优化后的代码：

```
// 伪代码逻辑
SearchResponse searchResponse = BfTransQueryBuilder.newInstance().executeScrollQuery(new Scroll(TimeValue.timeValueMinutes(5L)))
boolean satisfyConditions = //满足查询条件
while(satisfyConditions){
    //游标查询获取TransIndexESDoc
    List<TransIndexESDoc> docs = searchHits.collect {MyJsonUtils.fromJson(it.sourceAsString, TransIndexESDoc, false)}
    processDocsPutInWorkBook(docs)
}

List<ReportDTO> dtos = docs.collect(doc -> {

    PolicyPremium policyPremium = MyJsonUtils.fromJson(doc.policyPremium, PolicyPremium, false)
    Address address = MyJsonUtils.fromJson(doc.policyholderAddress, Address, false)
    BankAccount bankAccount = MyJsonUtils.fromJson(doc.bankAccount, BankAccount, false)
    BcpPolicyBills bill = MyJsonUtils.fromJson(doc.bcpPolicyBill, BcpPolicyBills, false)
    List<BankFileRec> bankFileRecs = MyJsonUtils.fromJsonAsList(doc.bankFileRec, BankFileRec, false)
    MasterTable table = getByCache()
    Package aPackage = getByCache()
    String endoType = doc.endoType
    String endoCancelDate = doc.endoCancelDate
    // ...
    // 最后，组装一个DTO
    return assembleDTO(policy, bills, bankRecs, aPackage, endoList, ...);
});
```

优化结果：

|                          |         |            |           |
| ------------------------ | ------- | ---------- | --------- |
| **保单明细报表生成时间** | OOM失败 | **< 30分钟 | **> 99%** |

#### **第二次解决方案与优化实践：**

- **引入预计算**：在Trans_index索引中增加一个字段，用来预存储这个保单在这个report中的行数据，实现“空间换时间”。为什么要引入预计算？因为这个Report需要的字段比较多，需要再中间表（ES索引）中增加比较多的中间数据，很多字段从es拿出来还要做二次加工(Json -> Object)，所以选择了预计算，直接生产出“最终产品”。

**旧流程（宽表）：** `业务数据 -> 宽表（预计算了关联）-> 报表服务（序列化/二次计算）-> 报表展示`

**新流程（展示预计算）：** `业务数据 -> 预计算服务（直接生成报表所需的数据结构）-> 报表展示`

优化后的代码：

```
// 伪代码逻辑
List<PortfolioListingReportDTO> reportDTOList = docs?.collect {
    PortfolioListingReportDTO reportDTO = MyJsonUtils.fromJson(it.portfolioReport, PortfolioListingReportDTO, false)
}
excelWriter.fill(reportDTOList, writeSheet)
```

优化后的结果：

|                          |           |           |           |
| ------------------------ | --------- | --------- | --------- |
| **保单明细报表生成时间** | *< 30分钟 | **< 7分钟 | **> 80%** |



### **Case 2：账单聚合报表 - 内存爆炸**

- **ES聚合计算**：利用Elasticsearch天然支持大数据量聚合的优势，将计算下沉到ES层

问题代码逻辑：

```
// 伪代码 - 循环每个月直到最后一个月
while (!isLastMonth) {
    // 获取月份的开始日期和结束日期
    startDate = 月份第一天
    endDate = 月份最后一天

    // 获取该月数据
    List<PremPostingReportDTO> premPostingDTOList = listCurrentMonthData(startDate, endDate)

    //聚合该月数据
    aggsQueryBills(premPostingDTOList)

    // 跳到上一个月
    当前月份 = 上一个月

    // 检查是否是最后一个月
    if (当前月份 < 截止月份) {
        isLastMonth = true
    }
}
```

```
premPostingDTOList.groupBy {
        return it.businessPattern + "-" + it.productCode
    }?.each {
        PremPostingMetadata premPostingMetadata = it.value.find()
        String businessPattern = premPostingMetadata.businessPattern
        List<PremPostingMetadata> groupPostingDTOList = it.value
        List<PremPostingMetadata> groupCollectPostingDTOList = groupPostingDTOList.findAll {
            it.drcr == 1
        }
        List<PremPostingMetadata> groupRefundPostingDTOList = groupPostingDTOList.findAll {
            it.drcr == -1
        }
        PremPostingReportDTO reportDTO = new PremPostingReportDTO(
                businessLine: EGExternalFieldTranslator.getBusinessLine(businessPattern),
                totalDebitCount: groupCollectPostingDTOList.size(),
                totalDebitAmount: MyNumberUtils.format(MyNumberUtils.nvl(groupCollectPostingDTOList.sum { MyNumberUtils.nvl(it.amount) })),
                totalCreditCount: groupRefundPostingDTOList.size(),
                totalCreditAmount: MyNumberUtils.format(MyNumberUtils.nvl(groupRefundPostingDTOList.sum { MyNumberUtils.nvl(it.amount) })),
        )
        reportDTOS.add(reportDTO)
    }
}
```

**性能瓶颈分析：**

- **内存对象爆炸**：在内存中同时持有每个月所有的账单 * 2 * 8 ≈ 2.3亿个对象，极易引发OOM。

**解决方案与优化实践**：

- **ES聚合计算**：利用Elasticsearch天然支持大数据量聚合的优势，将计算下沉到ES层

优化后的代码：

```
Terms byBusinessPattern = response.getAggregations().get("by_businessPattern")
for (Terms.Bucket businessPatternBucket : byBusinessPattern.getBuckets()) {
    String businessPattern = businessPatternBucket.getKeyAsString()
    Terms byPackageCode = businessPatternBucket.getAggregations().get("by_packageCode")
    for (Terms.Bucket packageCodeBucket : byPackageCode.getBuckets()) {
        String packageCode = packageCodeBucket.getKeyAsString()
        Terms byBillType = packageCodeBucket.getAggregations().get("by_billType")
        PremPostingReportDTO reportDTO = new PremPostingReportDTO(
                businessLine: EGExternalFieldTranslator.getBusinessLine(businessPattern),
                billMonth: createdStr + "-" + packageCode
        )
        for (Terms.Bucket billTypeBucket : byBillType.getBuckets()) {
            String billType = billTypeBucket.getKeyAsString()
            BigDecimal amount = BigDecimal.ZERO
            ScriptedMetric preciseAgg = billTypeBucket.getAggregations().get("totalAmount")
            if (preciseAgg != null && preciseAgg.aggregation() != null) {
                amount = new BigDecimal(preciseAgg.aggregation().toString())
            }
            if (billType == BcpConst.BILL_TYPE_COLLECTION) {
                reportDTO.totalDebitAmount = MyNumberUtils.round2(amount)
                reportDTO.totalDebitCount = billTypeBucket.getDocCount()
            }
            if (billType == BcpConst.BILL_TYPE_PAYMENT) {
                reportDTO.totalCreditAmount = MyNumberUtils.round2(amount)
                reportDTO.totalCreditCount = billTypeBucket.getDocCount()
            }
        }
        postingReportDTOS.add(reportDTO)
    }
}
processDocsPutInWorkBook(postingReportDTOS, writeSheet, excelWriter, dateRange)
```

优化后的结果：

|                          |         |              |           |
| ------------------------ | ------- | ------------ | --------- |
| **保单明细报表生成时间** | OOM失败 | **< 10分钟** | **> 99%** |

## 总结与启示

通过本次优化实践，我们实现了报表系统从"崩溃"到"秒级响应"的蜕变。关键优化策略包括：

1. **查询优化**：解决N+1查询，采用批量查询和聚合查询的方式
2. **架构优化**：引入宽表设计和缓存策略
3. **处理方式优化**：采用流式处理和ES聚合计算
4. **预计算**：先计算好结果，生成报告时，直接查询这些预计算的结果
5. **流程优化**：建立了前置性能评估机制，在需求阶段就考量数据规模和性能要求；将非功能性需求纳入开发标准，避免“只重功能、忽视性能”的行为。

这些经验为后续大数据量场景下的系统设计提供了重要参考。