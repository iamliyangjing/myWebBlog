# 优化复杂产品BI计算时间过长问题

## 业务背景

平台系统中有一个产品HSC由于其子rider过多，计算BI时间非常长收到了用户投诉，速度优化保费计算时间。（由于其特殊属性，rider上还有子rider）

![image-20251109202949135](.\image\image-20251109202949135.png)

经过计算，可以看出如果全选，那么将会有十一个rider需要进行BI计算。

> BI计算：再保险里面有很重要的一环是利益演示 - 给投保人看购买后几年后的收益，通常以表格或图表的形式，展示未来数十年（甚至到100岁）保单的现金价值、身故保险金、生存金、分红等金额。

## 性能瓶颈分析

在选择rider过多的情况下，因为BI计算是按ride按cover Term月份计算的，**所以在特定场景下如果rider过多，BI计算时间非常长。**

## 优化思路

### 定位

首先使用了Jprofile来定位耗时代码位置。

![image-20251109204737423](.\image\image-20251109204737423.png)

### 具体实施细节

首先查看第一个耗时代码：

> ```groovy
> 第一次计算,计算不依赖其它计算项的独立计算项
> biSubItemCalcFirstLevel
> ```

分析可知，这里主要是按月按rider计算，并且是顺序执行，但是由于子rider计算后才能计算主rider，且有些rider依赖于前面rider的计算。

1. 按月按rider先后计算
2. 子rider计算后才能计算主rider
3. 有些rider依赖于前面rider的计算
4. 同一个rider计算任务相同

由于上面相同条件，我们可以抽取按rider抽取task类来进行计算。

```groovy
   private void calcCoverFirstLevel(WLBPlanCover planCover, HighSumBiResult.BiAmount biAmount, BiSubItemConf subItemConf, List<Callable<Void>> tasks) {
        String coverCode = planCover.coverCode
        int paymentTerm = getRiderPaymentTerm(coverCode)
        int coverTerm = getRiderCoverTerm(coverCode)

        def ilRegularTopUp = {
            // **
        }

        def stardardPrem = {
            // **
        }

        def loadingPrem = {
            // **
        }

        def serviceCharge = {
            // **
        }

        def lifeReward = {
            // **
        }

        def instalmentPrem = {
            // **
        }
        
        def basicComm = {
            // **
        }

        //注意：service charge每个月均有值
        def cc = {
            // **
        }
        def allocated = {
            // **
        }
        def wakalah = {
            // **
        }
        def mgmtExpense = {
            // **
        }
        def ilMgmtExpense = {
            // **
        }
        tasks.add(new Callable<Void>() {
            @Override
            Void call() throws Exception {
                UserContextUtils.run {
                    initLifeContext()
                    ilRegularTopUp()
                    stardardPrem()
                    loadingPrem()
                    serviceCharge()
                    lifeReward()
                    instalmentPrem()
                    if (isTheFirstMonthOfInstallment) {
                        if (coverCode == ProjectConst.RIDER_CODE_MY_PRIME_SAVER) {
                            HighSumBICalcCache.setInstalmentPrem(coverCode, policyMonth, biAmount.ilRegularTopUpT)
                        } else {
                            HighSumBICalcCache.setInstalmentPrem(coverCode, policyMonth, biAmount.instalmentPremT)
                            HighSumBICalcCache.setStdInstalmentPrem(coverCode, policyMonth, biAmount.stardardInstalmentPremT)
                            HighSumBICalcCache.setLoadingPrem(coverCode, policyMonth, biAmount.loadingInstalmentPremT)
                            HighSumBICalcCache.setServiceCharge(coverCode, policyMonth, biAmount.serviceChargeT)
                            HighSumBICalcCache.setLifeReword(coverCode, policyMonth, biAmount.lifeRewardT)
                        }
                    }
                    basicComm()
                    cc()
                    allocated()
                    wakalah()
                    mgmtExpense()
                    ilMgmtExpense()
                }.on(this).withLoginUser(user).go()
                return null
            }
        })
    }
```

tasks 可以分为三类

```groovy
List<Callable<Void>> childTasks = []
List<Callable<Void>> parentTasks = []
List<Callable<Void>> dependOnOtherRiderTasks = []
```

总体计算优化

```groovy
    private void calcAllCoverSecondLevel(BiRiderConf riderConf, Map<String, BiSubItemConf> biSubItemConfigMap) {
        //先计算子rider
        List<Callable<Void>> childTabarruChargeTasks = []
        List<Callable<Void>> childInforceSaTasks = []
        List<Callable<Void>> parentTabarruChargeTasks = []
        List<Callable<Void>> parentInforceSaTasks = []
        String riderCode = ProjectConst.MAIN_PRODUCT_HIGH_SUM_COVERED_RIDER_CODE_DEATHTPD
        WLBPlanCover rider = getRider(riderCode)
        calcCoverSecondLevel(rider, currentBIResultItem.mainDeath, biSubItemConfigMap.get(riderCode), childTabarruChargeTasks, childInforceSaTasks)

        riderCode = ProjectConst.MAIN_PRODUCT_HIGH_SUM_COVERED_RIDER_CODE_ACCIDENTAL
        rider = getRider(riderCode)
        calcCoverSecondLevel(rider, currentBIResultItem.mainACCI, biSubItemConfigMap.get(riderCode), childTabarruChargeTasks, childInforceSaTasks)

        riderCode = ProjectConst.MAIN_PRODUCT_HIGH_SUM_COVERED_RIDER_CODE_HAJJUMRAH
        rider = getRider(riderCode)
        calcCoverSecondLevel(rider, currentBIResultItem.mainHajj, biSubItemConfigMap.get(riderCode), childTabarruChargeTasks, childInforceSaTasks)
        //再计算父rider
        riderCode = ProjectConst.MAIN_PRODUCT_HIGH_SUM_COVERED
        rider = getRider(riderCode)
        calcCoverSecondLevel(rider, currentBIResultItem.main, biSubItemConfigMap.get(riderCode), parentTabarruChargeTasks, parentInforceSaTasks)

        riderCode = ProjectConst.RIDER_CODE_HIGH_SUM_COVERED_PROTECT_DEATHTPD
        if (riderConf.isRiderEnabled(riderCode) && isRiderSelected(riderCode)) {
            rider = getRider(riderCode)
            calcCoverSecondLevel(rider, currentBIResultItem.protectDeath, biSubItemConfigMap.get(riderCode), childTabarruChargeTasks, childInforceSaTasks)
        }

        riderCode = ProjectConst.RIDER_CODE_HIGH_SUM_COVERED_PROTECT_ACCIDENTAL
        if (riderConf.isRiderEnabled(riderCode) && isRiderSelected(riderCode)) {
            rider = getRider(riderCode)
            calcCoverSecondLevel(rider, currentBIResultItem.protectACCI, biSubItemConfigMap.get(riderCode), childTabarruChargeTasks, childInforceSaTasks)
        }

        riderCode = ProjectConst.RIDER_CODE_HIGH_SUM_COVERED_PROTECT_HAJJUMRAH
        if (riderConf.isRiderEnabled(riderCode) && isRiderSelected(riderCode)) {
            rider = getRider(riderCode)
            calcCoverSecondLevel(rider, currentBIResultItem.protectHajj, biSubItemConfigMap.get(riderCode), childTabarruChargeTasks, childInforceSaTasks)
        }

        riderCode = ProjectConst.RIDER_CODE_HIGH_SUM_COVERED_PROTECT
        if (riderConf.isRiderEnabled(riderCode) && isRiderSelected(riderCode)) {
            rider = getRider(riderCode)
            calcCoverSecondLevel(rider, currentBIResultItem.protect, biSubItemConfigMap.get(riderCode), parentTabarruChargeTasks, parentInforceSaTasks)
        }

        riderCode = ProjectConst.RIDER_CODE_HIGH_SUM_COVERED_PARENT_ACCIDENTAL
        if (riderConf.isRiderEnabled(riderCode) && isRiderSelected(riderCode)) {
            rider = getRider(riderCode)
            calcCoverSecondLevel(rider, currentBIResultItem.parentACCI, biSubItemConfigMap.get(riderCode), childTabarruChargeTasks, childInforceSaTasks)
        }

        riderCode = ProjectConst.RIDER_CODE_HIGH_SUM_COVERED_PARENT_NATURAL
        if (riderConf.isRiderEnabled(riderCode) && isRiderSelected(riderCode)) {
            rider = getRider(riderCode)
            calcCoverSecondLevel(rider, currentBIResultItem.parentDeath, biSubItemConfigMap.get(riderCode), childTabarruChargeTasks, childInforceSaTasks)
        }

        riderCode = ProjectConst.RIDER_CODE_HIGH_SUM_COVERED_PARENT
        if (riderConf.isRiderEnabled(riderCode) && isRiderSelected(riderCode)) {
            rider = getRider(riderCode)
            calcCoverSecondLevel(rider, currentBIResultItem.parent, biSubItemConfigMap.get(riderCode), parentTabarruChargeTasks, parentInforceSaTasks)
        }

        riderCode = ProjectConst.RIDER_CODE_HIGH_SUM_COVERED_CI_PLUS
        if (riderConf.isRiderEnabled(riderCode) && isRiderSelected(riderCode)) {
            rider = getRider(riderCode)
            calcCoverSecondLevel(rider, currentBIResultItem.ciPlus, biSubItemConfigMap.get(riderCode), parentTabarruChargeTasks, parentInforceSaTasks)
        }

        riderCode = ProjectConst.RIDER_CODE_HIGH_SUM_COVERED_CHARITY
        if (riderConf.isRiderEnabled(riderCode) && isRiderSelected(riderCode)) {
            rider = getRider(riderCode)
            calcCoverSecondLevel(rider, currentBIResultItem.charity, biSubItemConfigMap.get(riderCode), parentTabarruChargeTasks, parentInforceSaTasks)
        }

        riderCode = ProjectConst.RIDER_CODE_MY_PRIME_SAVER
        if (riderConf.isRiderEnabled(riderCode) && isRiderSelected(riderCode)) {
            rider = getRider(riderCode)
            calcCoverSecondLevel(rider, currentBIResultItem.saver, biSubItemConfigMap.get(riderCode), childTabarruChargeTasks, childInforceSaTasks)
        }

        riderCode = ProjectConst.RIDER_CODE_MY_PRIME_PAYOR_PLUS
        if (riderConf.isRiderEnabled(riderCode) && isRiderSelected(riderCode)) {
            rider = getRider(riderCode)
            calcCoverSecondLevel(rider, currentBIResultItem.payorPlus, biSubItemConfigMap.get(riderCode), parentTabarruChargeTasks, parentInforceSaTasks)
        }

        riderCode = ProjectConst.RIDER_CODE_MY_PRIME_WAIVER
        if (riderConf.isRiderEnabled(riderCode) && isRiderSelected(riderCode)) {
            rider = getRider(riderCode)
            calcCoverSecondLevel(rider, currentBIResultItem.waiver, biSubItemConfigMap.get(riderCode), parentTabarruChargeTasks, parentInforceSaTasks)
        }
        BiCommonUtils.doCalc(childInforceSaTasks)
        BiCommonUtils.doCalc(childTabarruChargeTasks)
        BiCommonUtils.doCalc(parentInforceSaTasks)
        BiCommonUtils.doCalc(parentTabarruChargeTasks)
    }

    HighSumBiResult biCalc(BiRiderConf riderConf, Map<String, BiSubItemConf> biSubItemConfigMap) {
        long start = System.currentTimeMillis()
        int maxPolicyMonth = mainCoverTerm * 12
        user = UserUtils.loginUser
        String calcIdPre = "BI_" + RandomStringUtils.randomNumeric(12) + policy.policyId
        LifeContextUtils.lifeContext.calcIdPre = calcIdPre
        HighSumBICalcCache.initCache(calcIdPre)
        //第一次计算,计算不依赖其它计算项的独立计算项 cost 20s
        biSubItemCalcFirstLevel(riderConf, biSubItemConfigMap, maxPolicyMonth)
        //第二次计算，计算相互依赖的计算项，
        // 此产品比较特殊，所以此三者放一起计算
        // tabaru 依赖forceSA
        //forceSA依赖前一个月的pifCashValue
        biSubItemCalcSecondLevel(riderConf, biSubItemConfigMap, maxPolicyMonth)

        calcSummary()
        if (riderConf.irrEnabled) {
            calcIRR()
        }

        if (riderConf.valueTermEnabled) {
            def sumAssured = {
                def mainSa = biResult.summary.mainSumAssured
                mainSa
            }()

            calcValueTerm(new ValueTermCalcFactor(
                    sumAssured: sumAssured,
                    insuredEntryAge: lifeContext.insuredEntryAgeYears,
                    insuredGender: lifeContext.insuredGender,
                    coverTerm: mainCoverTerm,
                    paymentPlan: lifeContext.paymentPlan
            ))
        }

        biResult.monthItems = []
        // retain the data for last month of one year, and this is default option
        biResult.items = biResult.items?.findAll {
            it.policyMonth % 12 == 0
        }
        HighSumBICalcCache.removeCache(calcIdPre)
        long cost = System.currentTimeMillis() - start
        log.info("BI done, cost ${cost / 1000} s")
        return biResult
    }
```

大致可以分为 我们先计算子rider，在计算父rider，最后计算需要依赖前面计算结果的rider。

查看第二个耗时代码：

同理也是可以使用多线程计算优化，但是存在依赖关系，依赖前面计算结果的rider可以后计算。

```groovy
    private void calcAllCoverSecondLevel(BiRiderConf riderConf, Map<String, BiSubItemConf> biSubItemConfigMap) {
        //先计算子rider
        List<Callable<Void>> childTabarruChargeTasks = []
        List<Callable<Void>> childInforceSaTasks = []
        List<Callable<Void>> parentTabarruChargeTasks = []
        List<Callable<Void>> parentInforceSaTasks = []
        String riderCode = ProjectConst.MAIN_PRODUCT_HIGH_SUM_COVERED_RIDER_CODE_DEATHTPD
        WLBPlanCover rider = getRider(riderCode)
        calcCoverSecondLevel(rider, currentBIResultItem.mainDeath, biSubItemConfigMap.get(riderCode), childTabarruChargeTasks, childInforceSaTasks)

        riderCode = ProjectConst.MAIN_PRODUCT_HIGH_SUM_COVERED_RIDER_CODE_ACCIDENTAL
        rider = getRider(riderCode)
        calcCoverSecondLevel(rider, currentBIResultItem.mainACCI, biSubItemConfigMap.get(riderCode), childTabarruChargeTasks, childInforceSaTasks)

        riderCode = ProjectConst.MAIN_PRODUCT_HIGH_SUM_COVERED_RIDER_CODE_HAJJUMRAH
        rider = getRider(riderCode)
        calcCoverSecondLevel(rider, currentBIResultItem.mainHajj, biSubItemConfigMap.get(riderCode), childTabarruChargeTasks, childInforceSaTasks)
        //再计算父rider
        riderCode = ProjectConst.MAIN_PRODUCT_HIGH_SUM_COVERED
        rider = getRider(riderCode)
        calcCoverSecondLevel(rider, currentBIResultItem.main, biSubItemConfigMap.get(riderCode), parentTabarruChargeTasks, parentInforceSaTasks)

        riderCode = ProjectConst.RIDER_CODE_HIGH_SUM_COVERED_PROTECT_DEATHTPD
        if (riderConf.isRiderEnabled(riderCode) && isRiderSelected(riderCode)) {
            rider = getRider(riderCode)
            calcCoverSecondLevel(rider, currentBIResultItem.protectDeath, biSubItemConfigMap.get(riderCode), childTabarruChargeTasks, childInforceSaTasks)
        }

        riderCode = ProjectConst.RIDER_CODE_HIGH_SUM_COVERED_PROTECT_ACCIDENTAL
        if (riderConf.isRiderEnabled(riderCode) && isRiderSelected(riderCode)) {
            rider = getRider(riderCode)
            calcCoverSecondLevel(rider, currentBIResultItem.protectACCI, biSubItemConfigMap.get(riderCode), childTabarruChargeTasks, childInforceSaTasks)
        }

        riderCode = ProjectConst.RIDER_CODE_HIGH_SUM_COVERED_PROTECT_HAJJUMRAH
        if (riderConf.isRiderEnabled(riderCode) && isRiderSelected(riderCode)) {
            rider = getRider(riderCode)
            calcCoverSecondLevel(rider, currentBIResultItem.protectHajj, biSubItemConfigMap.get(riderCode), childTabarruChargeTasks, childInforceSaTasks)
        }

        riderCode = ProjectConst.RIDER_CODE_HIGH_SUM_COVERED_PROTECT
        if (riderConf.isRiderEnabled(riderCode) && isRiderSelected(riderCode)) {
            rider = getRider(riderCode)
            calcCoverSecondLevel(rider, currentBIResultItem.protect, biSubItemConfigMap.get(riderCode), parentTabarruChargeTasks, parentInforceSaTasks)
        }

        riderCode = ProjectConst.RIDER_CODE_HIGH_SUM_COVERED_PARENT_ACCIDENTAL
        if (riderConf.isRiderEnabled(riderCode) && isRiderSelected(riderCode)) {
            rider = getRider(riderCode)
            calcCoverSecondLevel(rider, currentBIResultItem.parentACCI, biSubItemConfigMap.get(riderCode), childTabarruChargeTasks, childInforceSaTasks)
        }

        riderCode = ProjectConst.RIDER_CODE_HIGH_SUM_COVERED_PARENT_NATURAL
        if (riderConf.isRiderEnabled(riderCode) && isRiderSelected(riderCode)) {
            rider = getRider(riderCode)
            calcCoverSecondLevel(rider, currentBIResultItem.parentDeath, biSubItemConfigMap.get(riderCode), childTabarruChargeTasks, childInforceSaTasks)
        }

        riderCode = ProjectConst.RIDER_CODE_HIGH_SUM_COVERED_PARENT
        if (riderConf.isRiderEnabled(riderCode) && isRiderSelected(riderCode)) {
            rider = getRider(riderCode)
            calcCoverSecondLevel(rider, currentBIResultItem.parent, biSubItemConfigMap.get(riderCode), parentTabarruChargeTasks, parentInforceSaTasks)
        }

        riderCode = ProjectConst.RIDER_CODE_HIGH_SUM_COVERED_CI_PLUS
        if (riderConf.isRiderEnabled(riderCode) && isRiderSelected(riderCode)) {
            rider = getRider(riderCode)
            calcCoverSecondLevel(rider, currentBIResultItem.ciPlus, biSubItemConfigMap.get(riderCode), parentTabarruChargeTasks, parentInforceSaTasks)
        }

        riderCode = ProjectConst.RIDER_CODE_HIGH_SUM_COVERED_CHARITY
        if (riderConf.isRiderEnabled(riderCode) && isRiderSelected(riderCode)) {
            rider = getRider(riderCode)
            calcCoverSecondLevel(rider, currentBIResultItem.charity, biSubItemConfigMap.get(riderCode), parentTabarruChargeTasks, parentInforceSaTasks)
        }

        riderCode = ProjectConst.RIDER_CODE_MY_PRIME_SAVER
        if (riderConf.isRiderEnabled(riderCode) && isRiderSelected(riderCode)) {
            rider = getRider(riderCode)
            calcCoverSecondLevel(rider, currentBIResultItem.saver, biSubItemConfigMap.get(riderCode), childTabarruChargeTasks, childInforceSaTasks)
        }

        riderCode = ProjectConst.RIDER_CODE_MY_PRIME_PAYOR_PLUS
        if (riderConf.isRiderEnabled(riderCode) && isRiderSelected(riderCode)) {
            rider = getRider(riderCode)
            calcCoverSecondLevel(rider, currentBIResultItem.payorPlus, biSubItemConfigMap.get(riderCode), parentTabarruChargeTasks, parentInforceSaTasks)
        }

        riderCode = ProjectConst.RIDER_CODE_MY_PRIME_WAIVER
        if (riderConf.isRiderEnabled(riderCode) && isRiderSelected(riderCode)) {
            rider = getRider(riderCode)
            calcCoverSecondLevel(rider, currentBIResultItem.waiver, biSubItemConfigMap.get(riderCode), parentTabarruChargeTasks, parentInforceSaTasks)
        }
        BiCommonUtils.doCalc(childInforceSaTasks)
        BiCommonUtils.doCalc(childTabarruChargeTasks)
        BiCommonUtils.doCalc(parentInforceSaTasks)
        BiCommonUtils.doCalc(parentTabarruChargeTasks)
    }
```

## 出现问题

由于BI引入了多线程计算，部分BI需要依赖保费计算结果，多线程计算时，取不到保费更新的数据（MySQL事务是根据连接来的，每个线程持有一个连接）

![image-20251118001017970](.\image\image-20251118001017970.png)

方案一： 采用和之前LCA 优化流程一样的步骤， BI 和 计算保费分开接口

方案二：编程式事务 + 手动传递 Connection 

目前系统采用的方案一（考虑方案二 修改成本比较大，目前系统都是使用的**声明式事务**，而且需要考虑事务失效的情况），如果客户手动关闭游览器，可以考虑datapatch修复。

## 优化结果

±：Because the script realizes that all riders are selected under the same conditions of the insured, with different ages, occupations and genders, and incomes, and paid in a random way, the test result is added before “±”

| **NB** | **API Name**                   | **API**                                                      | **Response Time (s)** |               |                          |
| ------ | ------------------------------ | ------------------------------------------------------------ | --------------------- | ------------- | ------------------------ |
|        |                                |                                                              | **Pre-optimization**  | **Optimized** | ***\*Time Reduction\**** |
| 1      | BI Calculation                 | Request Method:getRequest URL: /api/cart/illustrate?lang=en  | HSC:±107.8s           | HSC:±23.7s    | ↑up ±70s                 |
|        |                                |                                                              | Hajj:±191.1s          | Hajj:±17.7s   | ↑up ±170s                |
| 2      | Review UW_COUNTER_OFFER_FORM   | Request Method:getRequest URL: /api/policies/refId/pdf/UW_COUNTER_OFFER_FORM?lang=en | HSC:±13.4s            | HSC:±1.9s     | ↑up ±10s                 |
|        |                                |                                                              | Hajj:±8.1s            | Hajj:±1.6s    | ↑up ±6.5s                |
| 3      | Review HSC_BI_FORM             | Request Method:getRequest URL: /api/policies/refId/pdf/HSC_BI_FORM?lang=en | ±13.8s                | ±3.5s         | ↑up ±10.3s               |
| 4      | Review HSC_FFS                 | Request Method:getRequest URL: /api/policies/refId/pdf/HSC_FFS?lang=en | ±13.3s                | ±1.9s         | ↑up ±9s                  |
| 5      | Review ADVICE_FORM             | Request Method:getRequest URL: /api/policies/refId/pdf/ADVICE_FORM?lang=en | HSC:±12.8s            | HSC:±1.8      | ↑up ±11s                 |
|        |                                |                                                              | Hajj:±7.2s            | Hajj:±1.4     | ↑up ±5.8s                |
| 6      | Review HSC_PDS_FORM            | Request Method:getRequest URL: /api/policies/refId/pdf/HSC_PDS_FORM?lang=en | ±12.7s                | ±2.3s         | ↑up ±10.4s               |
| 7      | Review HSC_PROPOSAL_FORM       | Request URL: getRequest URL: /api/policies/refId/pdf/HSC_PROPOSAL_FORM?lang=en | ±15.0s                | ±3.7s         | ↑up ±11.3s               |
| 8      | Review CFF_FORM                | Request Method:getRequest URL: /api/policies/refId/pdf/CFF_FORM?lang=en | HSC:±13.7s            | HSC:±2.3s     | HSC:±11.4s               |
|        |                                |                                                              | Hajj:±7.6s            | Hajj:±2.0s    | ↑up ±5.6s                |
| 9      | supervisor  review             | Request Method:postRequest URL:/api/uws/submit?lang=en       | ±22.5s                | ±6.3s         | ↑up±16s                  |
| 10     | proposal list enquiry          | Request Method:postRequest URL:/api/policiesq?lang=en        | ±3.2s                 | ±0.75s        | ↑up±2.4s                 |
| 11     | Review  PRIME_BI_FORM          | Request Method:getRequest URL:/api/policies/refId/pdf/PRIME_BI_FORM?lang=en | ±14.1s                | ±4.1s         | ↑up±9.7s                 |
| 12     | Review  MY_MEDIC_PDS_APDX_FORM | Request Method:getRequest URL:/api/policies/refId/pdf/MY_MEDIC_PDS_APDX_FORM?lang=en | \                     | ±1.4s         | -                        |
| 13     | Review  MYPRIME_PDS_FORM       | Request Method:getRequest URL:/api/policies/refId/pdf/MYPRIME_PDS_FORM?lang=en | ±8.3s                 | ±2.0s         | ↑up±6.3s                 |
| 14     | Review  MY_MEDIC_PDS_FORM      | Request Method:getRequest URL:/api/policies/refId/pdf/MY_MEDIC_PDS_FORM?lang=en | \                     | ±2.2s         | -                        |
| 15     | ReviewMYPRIME_PROPOSAL_FORM    | Request Method:getRequest URL:/api/policies/refId/pdf/MYPRIME_PROPOSAL_FORM?lang=en | ±8.3s                 | ±2.0s         | ↑up±6.3s                 |
| 16     | ReviewSAVER FORM               | Request Method:getRequest URL:/api/policies/refld1]pdf/SAVER FORM?lang=en | ±8.3s                 | ±1.5s         | ↑up±6.8s                 |

-------------------------------- 2025.12.7 更新

## 优化带来的问题

保费计算 + BI计算 不能在同一个事务内进行 -- > BI计算需要利用到保费计算的结果，如果同一个事务内，**且子线程里计算保费，拿不到未提交事务的数据。**