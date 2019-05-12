---
title: AOP in Catch Error
date: 2018-2-07 16:14:10
tags:
 -Spring
categories: Spring
thumbnail: https://raw.githubusercontent.com/apollochen123/image/master/%E9%BB%98%E8%AE%A42.jpg
---

# Monitor Error Catch AOP

### Without Monitor Error Catch AOP
we don't use this AOP code:
```java
private void mainProcess(Document document, InboundTransaction transaction) throws InboundProcessException
{
    try
    {
        //convert inbound data from xml to json
        List<Element> elements = document.getRootElement().elements();
        String typeCode = document.getRootElement().getName();
        String headerCode = elements.get(0).getName();
        String itemCode = inboundNodeViewRepository.findNodeCodeByLevelAndTypeCodeAndStatusCode(1, typeCode,
            InboundConfigurationStatusCodeConstant.ACTIVE_STATUS_CODE);
        JSONObject inboundDataJson = convertInboundToJson(document, headerCode, itemCode);

        //entitlement mapping and choose generation process variant
        List<RuntimeGenerationProcessModel> runtimeModels = GenerateRunTimeGeneralProcessModels(inboundDataJson, headerCode, itemCode);

        geneContextAndInvokeWorkflow(runtimeModels, inboundDataJson, headerCode, itemCode, transaction);
    }
    catch (Exception e)
    {
        EmsLogger.error(e.getLocalizedMessage(), e);
        if (e instanceof MutiMessageException)
        {
            List<MessageResponse> messageResponseList = ((MutiMessageException) e).getMessageList();
            List<String> messageList = new ArrayList<>();
            messageResponseList.forEach(m -> {
                messageList.add(m.getMessage());
            });

            String message = Arrays.toString(messageList.toArray());
            InboundMonitorLog inboundMonitorLog = new InboundMonitorLog();
            inboundMonitorLog.setErrorInformation(Arrays.toString(e.getStackTrace()));
            inboundMonitorLog.setExecutionBy("SAPIT");
            inboundMonitorLog.setExecutionDate(new Date());
            inboundMonitorLog.setInboundTransaction(transaction);
            inboundMonitorLog.setMessage(message);
            inboundMonitorLog.setStatusCode(InboundMonitorLogStatusCodeConstant.PROCESS_ERROR_STATUS_CODE);
            inboundMonitorLogRepository.save(inboundMonitorLog);

            transaction.setCurrentInboundLog(inboundMonitorLog);
            if (transaction.getId() == 0)
            {
                transaction.setId(inboundTransactionRepository.getSequence());
            }
            inboundTransactionRepository.save(transaction);
            throw e;
        }
        else
        {
            InboundProcessException inboundProcessException = new InboundProcessException(e.getMessage());
            inboundProcessException.setErrorInfo(Arrays.toString(e.getStackTrace()));
            inboundProcessException.setExecutionBy("SAPIT");
            inboundProcessException.setExecutionDate(new Date());
            inboundProcessException.setErrorStatusCode(InboundMonitorLogStatusCodeConstant.PROCESS_ERROR_STATUS_CODE);
            inboundProcessException.setInboundTransaction(transaction);

            throw inboundProcessException;
        }
    }
}

```
try catch is so long and catch code can't reused.

### Use AOP Catch
```java
@ProcessExceptionAnnotation(InboundMonitorLogStatusCodeConstant.PROCESS_ERROR_STATUS_CODE)
private void mainProcess(Document document, InboundTransaction transaction)
{
    //convert inbound data from xml to json
    List<Element> elements = document.getRootElement().elements();
    String typeCode = document.getRootElement().getName();
    String headerCode = elements.get(0).getName();
    String itemCode = inboundNodeViewRepository.findNodeCodeByLevelAndTypeCodeAndStatusCode(1, typeCode,
        InboundConfigurationStatusCodeConstant.ACTIVE_STATUS_CODE);
    JSONObject inboundDataJson = convertInboundToJson(document, headerCode, itemCode);
    InboundDataModel inboundDataModel = new InboundDataModel(inboundDataJson, headerCode, itemCode);

    //entitlement mapping and choose generation process variant
    List<RuntimeGenerationProcessModel> runtimeModels = GenerateRunTimeGeneralProcessModels(inboundDataJson, headerCode, itemCode);

    geneContextAndInvokeWorkflow(runtimeModels, inboundDataModel, transaction);
}

```

@ProcessExceptionAnnotation(InboundMonitorLogStatusCodeConstant.PROCESS_ERROR_STATUS_CODE).value of the annotation is the monitor statusCode.
you just need add the annotation and set monitor log status code in it.

You can extend the AOP, such as adding some other Exception:
ProcessExceptionaAspect.java
```java
else if (e instanceof InboundProcessException)
{
    InboundTransaction transaction = ThreadLocalCache.transaction.get();
    InboundMonitorLog inboundMonitorLog = new InboundMonitorLog();
    inboundMonitorLog.setErrorInformation(Arrays.toString(e.getStackTrace()));
    inboundMonitorLog.setExecutionBy("SAPIT");
    inboundMonitorLog.setExecutionDate(new Date());
    inboundMonitorLog.setInboundTransaction(transaction);
    inboundMonitorLog.setMessage(e.getMessage());
    if (StringUtils.isNoneBlank(processExceptionAnnotation.value()))
    {
        inboundMonitorLog.setStatusCode(processExceptionAnnotation.value());
    }
    else
    {
        inboundMonitorLog.setStatusCode(InboundMonitorLogStatusCodeConstant.FAILED_STATUS_CODE);
    }

    inboundMonitorLogRepository.save(inboundMonitorLog);
    transaction.setCurrentInboundLog(inboundMonitorLog);
    if (transaction.getId() == 0)
    {
        transaction.setId(inboundTransactionRepository.getSequence());
    }

    inboundTransactionRepository.save(transaction);
    BaseRuntimeException ex = new BaseRuntimeException(e.getLocalizedMessage());
    ex.setStackTrace(e.getStackTrace());
    throw ex;
}
```
Add yourselves Exception after above code.
### Note
1. If you use this AOP, please make sure that :transaction must in ThreadLocalCache.
```
 ThreadLocalCache.transaction.set(transaction);
```
please remember clear ThreadLocalCache before the request is finished
2. AOP can't take effect in methods of the same kind. Because of
```
 //this code is same as
 //this.mainProcess(document, transaction)
mainProcess(document, transaction);
```
I use a ingenious way to solve it. @Autowired itself in class
```
@Service
public class TransactionDataServiceImpl implements TransactionDataService
{
    @Autowired
    private TransactionDataServiceImpl proxySelf;
}
```
and then
```
        proxySelf.mainProcess(document, transaction);
```
Now AOP can take effect.
