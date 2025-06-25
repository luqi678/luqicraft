# 基于EasyExcel+线程池+定时轮询服务解决POI文件导出大量数据时的内存溢出及超时问题


# 基于EasyExcel+线程池+定时轮询服务解决POI文件导出大量数据时的内存溢出及超时问题

## 背景
在一个全部合同导出功能中，需要导出 Excel, 但是全部合同的数量比较大，用原本的 Apache POI 库导出数据时因其内存占用较高会导致内存溢出问题。同时，数据处理过程比较耗时，导致用户等待时间过长甚至有可能请求超时。造成用户体验非常差，为解决这些问题，我采用了基于 EasyExcel 和线程池的解决方案。

### 原实现
原系统使用的是POI的XSSFWorkbook,采用的方式也是同步执行。点击导出后等待文件导出完成。

## 技术选型
针对数据量大导出会有内存溢出的问题可以选用，SXSSFWorkbook和EasyExcel方式。

但是考虑到不影响原来的导出功能模块，以及POI的代码复杂度高。因此采用EasyExcel的方式。

针对导出时间长的问题，采用线程池+定时服务轮询的方式异步导出excel，上传至小文件系统。之后再导出列表中提供下载连接。

## 具体实现
controller层 接受用户的文件导出请求

```plain
@Override
public Integer exportBcontractJoinAndContractGx(BcontractJoinPageQryVo paramVo) {
    //写入轮询记录表中等待轮询任务执行
    SchdulerStrategyConfigQueryVo param = new SchdulerStrategyConfigQueryVo();
    Optional.of(SalGxCommonEnum.SCHDULER_STRATEGY_CONFIG_OBJ_TYPE_2400.getCode()).ifPresent(param::setObjType);
    Optional.of(SalGxCommonEnum.STATE_70A.getCode()).ifPresent(param::setSchedulerStatusCd);
    param.setCreateStaff(BaseContext.getStaffId());
    if(CollectionUtils.isNotEmpty(SchdulerStrategyConfig.repository().complexQuery(param))){
       throw new BaseAppException("后台正在导出广西合同到期清单_全部合同数据,请后台未导出或导出成功后再试！");
    }
    JSONObject jsonObject = PmsBeanUtils.mapToBean(JSONObject.class, paramVo);
    jsonObject.put("info",BaseContext.getBaseContextInfo());
    String paramStr = Optional.of(JSONObject.toJSONString(jsonObject)).filter(StringUtils::isNotBlank).orElse(null);
    return SalCommonUtils.saveSchdulerStrategyConfigEntity(null,SalGxCommonEnum.SCHDULER_STRATEGY_CONFIG_OBJ_TYPE_2400.getCode(),null,
          SalGxCommonEnum.SCHDULER_STRATEGY_CONFIG_OBJ_TYPE_2400_KEY.getCode(),paramStr);
}
```

具体导出实现

```java

/**
 * 导出excel
 *
 * @param fileName excel文件名称
 * @param sheetName excel sheet名称
 * @param list 数据
 * @param clazz
 * @param response
 */
public static void exportExcel(String fileName, String sheetName, List list, Class clazz, HttpServletResponse response){
    ServletOutputStream outputStream;
    try {
        response.setContentType("application/vnd.ms-excel");
        response.setCharacterEncoding("utf8");
        response.setHeader("Content-Disposition", "attachment; filename=" + fileName + ".xlsx");
        outputStream = response.getOutputStream();
        EasyExcel.write(outputStream)
                .head(clazz)
                .excelType(ExcelTypeEnum.XLSX)
                .sheet(sheetName)
                .doWrite(list);
        outputStream.flush();
    } catch (Exception e) {
        log.error("导出excel异常：{}",e.getMessage());
        e.printStackTrace();
    }


    ByteArrayOutputStream outputSteam = new ByteArrayOutputStream();
    EasyExcel.write(outputStream)
        .head(clazz)
        .excelType(ExcelTypeEnum.XLSX)
        .sheet(sheetName)
        .doWrite(list);
    outputSteam.toByteArray();
    
}

```

线程代码

```java
package com.ffcs.gewp.scheduler.sal.service.manager.Impl;

import com.ffcs.crmd.platform.base.utils.type.CollectionUtils;
import com.ffcs.gewp.kernel.pub.context.BaseContext;
import com.ffcs.gewp.kernel.pub.context.BaseContextInfo;
import com.ffcs.gewp.kernel.pub.enums.DataMergeTypeEnum;
import com.ffcs.gewp.kernel.pub.utils.SpringUtil;
import com.ffcs.gewp.kernel.pub.utils.ThreadPoolUtil;
import com.ffcs.gewp.kernel.service.strategy.BaseSchedulerStrategy;
import com.ffcs.gewp.kernel.service.vo.RetBaseVo;
import com.ffcs.gewp.kernel.service.vo.SchdulerStrategyConfigBaseVo;
import com.ffcs.gewp.sal.constants.enums.SalCommonEnum;
import com.ffcs.gewp.sal.service.convert.DTO2VoConvert;
import com.ffcs.gewp.sal.service.util.SalCommonUtils;
import com.ffcs.gewp.sal.visitmanage.entity.SchdulerStrategyConfig;
import com.ffcs.gewp.sal.visitmanage.manager.ISchdulerStrategyConfigManager;
import com.ffcs.gewp.sal.visitmanage.vo.SchdulerStrategyConfigVo;
import com.ffcs.gewp.sal.visitmanage.vo.query.SchdulerStrategyConfigQueryVo;
import com.ffcs.gewp.scheduler.sal.service.manager.ISchedulerStrategyManager;
import com.ffcs.gewp.scheduler.sal.service.param.SyncDataInParam;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import java.util.Arrays;
import java.util.List;
import java.util.Optional;
import java.util.concurrent.ThreadPoolExecutor;

/**
 * 通用轮询任务实现类
 * @author taowy
 */
@Service
public class SchedulerStrategyManagerImpl implements ISchedulerStrategyManager {

    private static final Logger log = LoggerFactory.getLogger(SchedulerStrategyManagerImpl.class);

    @Autowired
    private ISchdulerStrategyConfigManager schdulerStrategyConfigManager;

    /**
     * @description 轮询多线程执行任务,注意控制线程安全和死锁
     * @param param 入参
     */
    @Override
    public void run(SyncDataInParam param) {
        log.info("[轮询任务策略模式]:开始准备执行轮询策略任务...");
        SchdulerStrategyConfigQueryVo paramVo = new SchdulerStrategyConfigQueryVo();
        //查找待执行状态
        Optional.of(SalCommonEnum.STATE_70A.getCode()).ifPresent(paramVo::setSchedulerStatusCd);
        paramVo.setAppModule(SalCommonEnum.APP_MODULE_SAL.getCode());
        //未处理轮询任务列表
        List<SchdulerStrategyConfig> noDealTaskList = SchdulerStrategyConfig.repository().complexQuery(paramVo);
        if(CollectionUtils.isEmpty(noDealTaskList)){
            log.info("[轮询任务策略模式]:未查询到可执行的策略轮询任务...");
            return;
        }
        //设置多线程的本地地线程变量
        BaseContextInfo baseContext = BaseContext.getBaseContextInfo();
        ThreadPoolExecutor threadPoolExecutor = ThreadPoolUtil.standardThreadPool(4, 8, 2000L);
        log.info("[轮询任务策略模式]:线程池创建成功,当前线程池核心线程数为4,最大线程数为8");
        noDealTaskList.forEach(entity -> {
            entity.setSchedulerStatusCd(SalCommonEnum.STATE_70B.getCode());
            entity.update();
            threadPoolExecutor.execute(() -> {
                if(Optional.ofNullable(entity).isPresent()){
                    //执行策略
                    this.execute(entity,baseContext);
                }
            });
        });
        //线程池不再接收线程任务
        threadPoolExecutor.shutdown();
        log.info("[轮询任务策略模式]:当前线程池任务已添加完成");
    }

    public void execute(SchdulerStrategyConfig entity,BaseContextInfo baseContext){
        try {
            SchdulerStrategyConfigBaseVo baseVo = DTO2VoConvert.INSTANCE.toVo(entity);
            //设置线程名称
            Optional.of("thread-pool-thread-" + entity.getStrategyName()).ifPresent(Thread.currentThread()::setName);
            //设置本地线程变量
            Optional.ofNullable(baseContext).ifPresent(BaseContext::setBaseContextInfo);
            //获取策略类
            BaseSchedulerStrategy schedulerStrategy = SpringUtil.getBean(entity.getStrategyBeanId(), BaseSchedulerStrategy.class);
            //执行轮询任务策略
            RetBaseVo retDto = Optional.of(baseVo).map(schedulerStrategy::execute).orElse(null);
            Optional.ofNullable(retDto).map(RetBaseVo::getDetailMsg).map(o->SalCommonUtils.cutString(o,4000)).ifPresent(entity::setRemark);
            Optional.of(SalCommonEnum.STATE_70C.getCode()).ifPresent(entity::setSchedulerStatusCd);
        } catch (Exception e){
            e.printStackTrace();
            log.error(e.getMessage());
            Optional.ofNullable(SalCommonUtils.cutString(e.getMessage(),4000)).ifPresent(entity::setRemark);
            Optional.of(SalCommonEnum.STATE_70E.getCode()).ifPresent(entity::setSchedulerStatusCd);
        } finally {
            updateStrategyConfig(entity);
        }
    }

    /**
     * 修改轮询任务策略配置
     * @param entity
     * @return
     */
    private Integer updateStrategyConfig(SchdulerStrategyConfig entity){
        SchdulerStrategyConfigVo param = new SchdulerStrategyConfigVo();
        param.setSchedulerConfigId(entity.getSchedulerConfigId());
        param.setSchedulerStatusCd(entity.getSchedulerStatusCd());
        param.setRemark(entity.getRemark());
        param.setMergeType(DataMergeTypeEnum.MOD.getCode());
        param.setMergeProperties(Arrays.asList(SalCommonEnum.SCHEDULER_STATUS_CD.getCode()
                ,SalCommonEnum.REMARK.getCode()));
        return schdulerStrategyConfigManager.update(entity.getSchedulerConfigId(),null,param);
    }
}

```



## 其他问题
### Apache POI
1. **HSSFWorkbook**<font style="color:rgb(13, 13, 13);">：代表一个 Excel 文件，对应于 Excel 97-2003 格式的 </font>**.xls**<font style="color:rgb(13, 13, 13);"> 文件。它使用 HSSF（Horrible Spreadsheet Format）来处理 Excel 文件，适用于较旧版本的 Excel。</font>
2. **XSSFWorkbook**<font style="color:rgb(13, 13, 13);">：代表一个 Excel 文件，对应于 Excel 2007+ 格式的 </font>**.xlsx**<font style="color:rgb(13, 13, 13);"> 文件。它使用 XSSF（XML Spreadsheet Format）来处理 Excel 文件，支持更大的数据量和更多的功能，适用于较新版本的 Excel。</font>
3. **SXSSFWorkbook**<font style="color:rgb(13, 13, 13);">：是 XSSFWorkbook 的扩展，用于处理大量数据的导出。它使用了一种流式写入（Streaming Write）的方式，可以将数据写入到临时文件中，从而避免内存溢出的问题。</font>
4. **XSSFWorkbook（Streaming）**<font style="color:rgb(13, 13, 13);">：另外还有一种基于 XML 文件的流式读取（Streaming Read）方式，可以读取大量数据的 Excel 文件而不会导致内存溢出，但是这种方式并不是 POI 提供的官方支持，而是一些第三方库提供的扩展。</font>

### EasyExcel




> 更新: 2024-05-02 21:52:13  
> 原文: <https://www.yuque.com/lujianuse/notes/xag6e59tisz88xr2>

---

> 作者:   
> URL: http://localhost:1313/%E5%9F%BA%E4%BA%8Eeasyexcel-%E7%BA%BF%E7%A8%8B%E6%B1%A0-%E5%AE%9A%E6%97%B6%E8%BD%AE%E8%AF%A2%E6%9C%8D%E5%8A%A1%E8%A7%A3%E5%86%B3poi%E6%96%87%E4%BB%B6%E5%AF%BC%E5%87%BA%E5%A4%A7%E9%87%8F%E6%95%B0%E6%8D%AE%E6%97%B6%E7%9A%84%E5%86%85%E5%AD%98%E6%BA%A2%E5%87%BA%E5%8F%8A%E8%B6%85%E6%97%B6%E9%97%AE%E9%A2%98/  

