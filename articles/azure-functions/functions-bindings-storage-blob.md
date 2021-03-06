---
title: Azure Blob storage bindings for Azure Functions
description: Understand how to use Azure Blob storage triggers and bindings in Azure Functions.
services: functions
documentationcenter: na
author: ggailey777
manager: cfowler
editor: ''
tags: ''
keywords: azure functions, functions, event processing, dynamic compute, serverless architecture

ms.service: functions
ms.devlang: multiple
ms.topic: reference
ms.tgt_pltfrm: multiple
ms.workload: na
ms.date: 02/12/2018
ms.author: glenga
---

# Azure Blob storage bindings for Azure Functions

This article explains how to work with Azure Blob storage bindings in Azure Functions. Azure Functions supports trigger, input, and output bindings for blobs. The article includes a section for each binding:

* [Blob trigger](#trigger)
* [Blob input binding](#input)
* [Blob output binding](#output)

[!INCLUDE [intro](../../includes/functions-bindings-intro.md)]

> [!NOTE]
> [Blob-only storage accounts](../storage/common/storage-create-storage-account.md#blob-storage-accounts) are not supported for blob triggers. Blob storage triggers require a general-purpose storage account. For input and output bindings you can use blob-only storage accounts.

## Packages

The Blob storage bindings are provided in the [Microsoft.Azure.WebJobs](http://www.nuget.org/packages/Microsoft.Azure.WebJobs) NuGet package. Source code for the package is in the [azure-webjobs-sdk](https://github.com/Azure/azure-webjobs-sdk/tree/master/src) GitHub repository.

[!INCLUDE [functions-package-auto](../../includes/functions-package-auto.md)]

## Trigger

Use a Blob storage trigger to start a function when a new or updated blob is detected. The blob contents are provided as input to the function.

> [!NOTE]
> When you're using a blob trigger on a Consumption plan, there can be up to a 10-minute delay in processing new blobs after a function app has gone idle. After the function app is running, blobs are processed immediately. To avoid this initial delay, consider one of the following options:
> - Use an App Service plan with Always On enabled.
> - Use another mechanism to trigger the blob processing, such as a queue message that contains the blob name. For an example, see the [blob input bindings example later in this article](#input---example).

## Trigger - example

See the language-specific example:

* [C#](#trigger---c-example)
* [C# script (.csx)](#trigger---c-script-example)
* [JavaScript](#trigger---javascript-example)

### Trigger - C# example

The following example shows a [C# function](functions-dotnet-class-library.md) that writes a log when a blob is added or updated in the `samples-workitems` container.

```csharp
[FunctionName("BlobTriggerCSharp")]        
public static void Run([BlobTrigger("samples-workitems/{name}")] Stream myBlob, string name, TraceWriter log)
{
    log.Info($"C# Blob trigger function Processed blob\n Name:{name} \n Size: {myBlob.Length} Bytes");
}
```

The string `{name}` in the blob trigger path `samples-workitems/{name}` creates a [binding expression](functions-triggers-bindings.md#binding-expressions-and-patterns) that you can use in function code to access the file name of the triggering blob. For more information, see [Blob name patterns](#trigger---blob-name-patterns) later in this article.

For more information about the `BlobTrigger` attribute, see [Trigger - attributes](#trigger---attributes).

### Trigger - C# script example

The following example shows a blob trigger binding in a *function.json* file and [C# script (.csx)](functions-reference-csharp.md) code that uses the binding. The function writes a log when a blob is added or updated in the `samples-workitems` container.

Here's the binding data in the *function.json* file:

```json
{
    "disabled": false,
    "bindings": [
        {
            "name": "myBlob",
            "type": "blobTrigger",
            "direction": "in",
            "path": "samples-workitems/{name}",
            "connection":"MyStorageAccountAppSetting"
        }
    ]
}
```

The string `{name}` in the blob trigger path `samples-workitems/{name}` creates a [binding expression](functions-triggers-bindings.md#binding-expressions-and-patterns) that you can use in function code to access the file name of the triggering blob. For more information, see [Blob name patterns](#trigger---blob-name-patterns) later in this article.

For more information about *function.json* file properties, see the [Configuration](#trigger---configuration) section explains these properties.

Here's C# script code that binds to a `Stream`:

```cs
public static void Run(Stream myBlob, TraceWriter log)
{
   log.Info($"C# Blob trigger function Processed blob\n Name:{name} \n Size: {myBlob.Length} Bytes");
}
```

Here's C# script code that binds to a `CloudBlockBlob`:

```cs
#r "Microsoft.WindowsAzure.Storage"

using Microsoft.WindowsAzure.Storage.Blob;

public static void Run(CloudBlockBlob myBlob, string name, TraceWriter log)
{
    log.Info($"C# Blob trigger function Processed blob\n Name:{name}\nURI:{myBlob.StorageUri}");
}
```

### Trigger - JavaScript example

The following example shows a blob trigger binding in a *function.json* file and [JavaScript code](functions-reference-node.md) that uses the binding. The function writes a log when a blob is added or updated in the `samples-workitems` container.

Here's the *function.json* file:

```json
{
    "disabled": false,
    "bindings": [
        {
            "name": "myBlob",
            "type": "blobTrigger",
            "direction": "in",
            "path": "samples-workitems/{name}",
            "connection":"MyStorageAccountAppSetting"
        }
    ]
}
```

The string `{name}` in the blob trigger path `samples-workitems/{name}` creates a [binding expression](functions-triggers-bindings.md#binding-expressions-and-patterns) that you can use in function code to access the file name of the triggering blob. For more information, see [Blob name patterns](#trigger---blob-name-patterns) later in this article.

For more information about *function.json* file properties, see the [Configuration](#trigger---configuration) section explains these properties.

Here's the JavaScript code:

```javascript
module.exports = function(context) {
    context.log('Node.js Blob trigger function processed', context.bindings.myBlob);
    context.done();
};
```

## Trigger - attributes

In [C# class libraries](functions-dotnet-class-library.md), use the following attributes to configure a blob trigger:

* [BlobTriggerAttribute](https://github.com/Azure/azure-webjobs-sdk/blob/master/src/Microsoft.Azure.WebJobs/BlobTriggerAttribute.cs)

  The attribute's constructor takes a path string that indicates the container to watch and optionally a [blob name pattern](#trigger---blob-name-patterns). Here's an example:

  ```csharp
  [FunctionName("ResizeImage")]
  public static void Run(
      [BlobTrigger("sample-images/{name}")] Stream image, 
      [Blob("sample-images-md/{name}", FileAccess.Write)] Stream imageSmall)
  {
      ....
  }
  ```

  You can set the `Connection` property to specify the storage account to use, as shown in the following example:

   ```csharp
  [FunctionName("ResizeImage")]
  public static void Run(
      [BlobTrigger("sample-images/{name}", Connection = "StorageConnectionAppSetting")] Stream image, 
      [Blob("sample-images-md/{name}", FileAccess.Write)] Stream imageSmall)
  {
      ....
  }
  ```

  For a complete example, see [Trigger - C# example](#trigger---c-example).

* [StorageAccountAttribute](https://github.com/Azure/azure-webjobs-sdk/blob/master/src/Microsoft.Azure.WebJobs/StorageAccountAttribute.cs)

  Provides another way to specify the storage account to use. The constructor takes the name of an app setting that contains a storage connection string. The attribute can be applied at the parameter, method, or class level. The following example shows class level and method level:

  ```csharp
  [StorageAccount("ClassLevelStorageAppSetting")]
  public static class AzureFunctions
  {
      [FunctionName("BlobTrigger")]
      [StorageAccount("FunctionLevelStorageAppSetting")]
      public static void Run( //...
  {
      ....
  }
  ```

The storage account to use is determined in the following order:

* The `BlobTrigger` attribute's `Connection` property.
* The `StorageAccount` attribute applied to the same parameter as the `BlobTrigger` attribute.
* The `StorageAccount` attribute applied to the function.
* The `StorageAccount` attribute applied to the class.
* The default storage account for the function app ("AzureWebJobsStorage" app setting).

## Trigger - configuration

The following table explains the binding configuration properties that you set in the *function.json* file and the `BlobTrigger` attribute.

|function.json property | Attribute property |Description|
|---------|---------|----------------------|
|**type** | n/a | Must be set to `blobTrigger`. This property is set automatically when you create the trigger in the Azure portal.|
|**direction** | n/a | Must be set to `in`. This property is set automatically when you create the trigger in the Azure portal. Exceptions are noted in the [usage](#trigger---usage) section. |
|**name** | n/a | The name of the variable that represents the blob in function code. | 
|**path** | **BlobPath** |The container to monitor.  May be a [blob name pattern](#trigger-blob-name-patterns). | 
|**connection** | **Connection** | The name of an app setting that contains the Storage connection string to use for this binding. If the app setting name begins with "AzureWebJobs", you can specify only the remainder of the name here. For example, if you set `connection` to "MyStorage", the Functions runtime looks for an app setting that is named "AzureWebJobsMyStorage." If you leave `connection` empty, the Functions runtime uses the default Storage connection string in the app setting that is named `AzureWebJobsStorage`.<br><br>The connection string must be for a general-purpose storage account, not a [blob-only storage account](../storage/common/storage-create-storage-account.md#blob-storage-accounts).|

[!INCLUDE [app settings to local.settings.json](../../includes/functions-app-settings-local.md)]

## Trigger - usage

In C# and C# script, you can use the following parameter types for the triggering blob:

* `Stream`
* `TextReader`
* `string`
* `Byte[]`
* A POCO serializable as JSON
* `ICloudBlob`<sup>1</sup>
* `CloudBlockBlob`<sup>1</sup>
* `CloudPageBlob`<sup>1</sup>
* `CloudAppendBlob`<sup>1</sup>

<sup>1</sup> Requires "inout" binding `direction` in *function.json* or `FileAccess.ReadWrite` in a C# class library.

Binding to `string`, `Byte[]`, or POCO is only recommended if the blob size is small, as the entire blob contents are loaded into memory. Generally, it is preferable to use a `Stream` or `CloudBlockBlob` type. For more information, see [Concurrency and memory usage](#trigger---concurrency-and-memory-usage) later in this article.

In JavaScript, access the input blob data using `context.bindings.<name from function.json>`.

## Trigger - blob name patterns

You can specify a blob name pattern in the `path` property in *function.json* or in the `BlobTrigger` attribute constructor. The name pattern can be a [filter or binding expression](functions-triggers-bindings.md#binding-expressions-and-patterns). The following sections provide examples.

### Get file name and extension

The following example shows how to bind to the blob file name and extension separately:

```json
"path": "input/{blobname}.{blobextension}",
```
If the blob is named *original-Blob1.txt*, the values of the `blobname` and `blobextension` variables in function code are *original-Blob1* and *txt*.

### Filter on blob name

The following example triggers only on blobs in the `input` container that start with the string "original-":

```json
"path": "input/original-{name}",
```
 
If the blob name is *original-Blob1.txt*, the value of the `name` variable in function code is `Blob1`.

### Filter on file type

The following example triggers only on *.png* files:

```json
"path": "samples/{name}.png",
```

### Filter on curly braces in file names

To look for curly braces in file names, escape the braces by using two braces. The following example filters for blobs that have curly braces in the name:

```json
"path": "images/{{20140101}}-{name}",
```

If the blob is named *{20140101}-soundfile.mp3*, the `name` variable value in the function code is *soundfile.mp3*. 

## Trigger - metadata

The blob trigger provides several metadata properties. These properties can be used as part of binding expressions in other bindings or as parameters in your code. These values have the same semantics as the [Cloud​Blob](https://docs.microsoft.com/dotnet/api/microsoft.windowsazure.storage.blob.cloudblob?view=azure-dotnet) type.

|Property  |Type  |Description  |
|---------|---------|---------|
|`BlobTrigger`|`string`|The path to the triggering blob.|
|`Uri`|`System.Uri`|The blob's URI for the primary location.|
|`Properties` |[BlobProperties](https://docs.microsoft.com/dotnet/api/microsoft.windowsazure.storage.blob.blobproperties)|The blob's system properties. |
|`Metadata` |`IDictionary<string,string>`|The user-defined metadata for the blob.|

For example, the following C# script and JavaScript examples log the path to the triggering blob, including the container:

```csharp
public static void Run(string myBlob, string blobTrigger, TraceWriter log)
{
    log.Info($"Full blob path: {blobTrigger}");
} 
```

```javascript
module.exports = function (context, myBlob) {
    context.log("Full blob path:", context.bindingData.blobTrigger);
    context.done();
};
```

## Trigger - blob receipts

The Azure Functions runtime ensures that no blob trigger function gets called more than once for the same new or updated blob. To determine if a given blob version has been processed, it maintains *blob receipts*.

Azure Functions stores blob receipts in a container named *azure-webjobs-hosts* in the Azure storage account for your function app (defined by the app setting `AzureWebJobsStorage`). A blob receipt has the following information:

* The triggered function ("*&lt;function app name>*.Functions.*&lt;function name>*", for example: "MyFunctionApp.Functions.CopyBlob")
* The container name
* The blob type ("BlockBlob" or "PageBlob")
* The blob name
* The ETag (a blob version identifier, for example: "0x8D1DC6E70A277EF")

To force reprocessing of a blob, delete the blob receipt for that blob from the *azure-webjobs-hosts* container manually.

## Trigger - poison blobs

When a blob trigger function fails for a given blob, Azure Functions retries that function a total of 5 times by default. 

If all 5 tries fail, Azure Functions adds a message to a Storage queue named *webjobs-blobtrigger-poison*. The queue message for poison blobs is a JSON object that contains the following properties:

* FunctionId (in the format *&lt;function app name>*.Functions.*&lt;function name>*)
* BlobType ("BlockBlob" or "PageBlob")
* ContainerName
* BlobName
* ETag (a blob version identifier, for example: "0x8D1DC6E70A277EF")

## Trigger - concurrency and memory usage

The blob trigger uses a queue internally, so the maximum number of concurrent function invocations is controlled by the [queues configuration in host.json](functions-host-json.md#queues). The default settings limit concurrency to 24 invocations. This limit applies separately to each function that uses a blob trigger.

[The consumption plan](functions-scale.md#how-the-consumption-plan-works) limits a function app on one virtual machine (VM) to 1.5 GB of memory. Memory is used by each concurrently executing function instance and by the Functions runtime itself. If a blob-triggered function loads the entire blob into memory, the maximum memory used by that function just for blobs is 24 * maximum blob size. For example, a function app with three blob-triggered functions and the default settings would have a maximum per-VM concurrency of 3*24 = 72 function invocations.

JavaScript functions load the entire blob into memory, and C# functions do that if you bind to `string`, `Byte[]`, or POCO.

## Trigger - polling

If the blob container being monitored contains more than 10,000 blobs, the Functions runtime scans log files to watch 
for new or changed blobs. This process can result in delays. A function might not get triggered until several minutes or longer 
after the blob is created. In addition, [storage logs are created on a "best effort"](/rest/api/storageservices/About-Storage-Analytics-Logging) 
basis. There's no guarantee that all events are captured. Under some conditions, logs may be missed. If you require faster or more reliable blob processing, consider creating a [queue message](../storage/queues/storage-dotnet-how-to-use-queues.md) 
 when you create the blob. Then use a [queue trigger](functions-bindings-storage-queue.md) instead of a blob trigger to process the blob. Another option is to use Event Grid; see the tutorial [Automate resizing uploaded images using Event Grid](../event-grid/resize-images-on-storage-blob-upload-event.md).

## Input

Use a Blob storage input binding to read blobs.

## Input - example

See the language-specific example:

* [C#](#input---c-example)
* [C# script (.csx)](#input---c-script-example)
* [JavaScript](#input---javascript-example)

### Input - C# example

The following example is a [C# function](functions-dotnet-class-library.md) that uses a queue trigger and an input blob binding. The queue message contains the name of the blob, and the function logs the size of the blob.

```csharp
[FunctionName("BlobInput")]
public static void Run(
    [QueueTrigger("myqueue-items")] string myQueueItem,
    [Blob("samples-workitems/{queueTrigger}", FileAccess.Read)] Stream myBlob,
    TraceWriter log)
{
    log.Info($"BlobInput processed blob\n Name:{myQueueItem} \n Size: {myBlob.Length} bytes");
}
```        

### Input - C# script example

<!--Same example for input and output. -->

The following example shows blob input and output bindings in a *function.json* file and [C# script (.csx)](functions-reference-csharp.md) code that uses the bindings. The function makes a copy of a text blob. The function is triggered by a queue message that contains the name of the blob to copy. The new blob is named *{originalblobname}-Copy*.

In the *function.json* file, the `queueTrigger` metadata property is used to specify the blob name in the `path` properties:

```json
{
  "bindings": [
    {
      "queueName": "myqueue-items",
      "connection": "MyStorageConnectionAppSetting",
      "name": "myQueueItem",
      "type": "queueTrigger",
      "direction": "in"
    },
    {
      "name": "myInputBlob",
      "type": "blob",
      "path": "samples-workitems/{queueTrigger}",
      "connection": "MyStorageConnectionAppSetting",
      "direction": "in"
    },
    {
      "name": "myOutputBlob",
      "type": "blob",
      "path": "samples-workitems/{queueTrigger}-Copy",
      "connection": "MyStorageConnectionAppSetting",
      "direction": "out"
    }
  ],
  "disabled": false
}
``` 

The [configuration](#input---configuration) section explains these properties.

Here's the C# script code:

```cs
public static void Run(string myQueueItem, string myInputBlob, out string myOutputBlob, TraceWriter log)
{
    log.Info($"C# Queue trigger function processed: {myQueueItem}");
    myOutputBlob = myInputBlob;
}
```

### Input - JavaScript example

<!--Same example for input and output. -->

The following example shows blob input and output bindings in a *function.json* file and [JavaScript code](functions-reference-node.md) that uses the bindings. The function makes a copy of a blob. The function is triggered by a queue message that contains the name of the blob to copy. The new blob is named *{originalblobname}-Copy*.

In the *function.json* file, the `queueTrigger` metadata property is used to specify the blob name in the `path` properties:

```json
{
  "bindings": [
    {
      "queueName": "myqueue-items",
      "connection": "MyStorageConnectionAppSetting",
      "name": "myQueueItem",
      "type": "queueTrigger",
      "direction": "in"
    },
    {
      "name": "myInputBlob",
      "type": "blob",
      "path": "samples-workitems/{queueTrigger}",
      "connection": "MyStorageConnectionAppSetting",
      "direction": "in"
    },
    {
      "name": "myOutputBlob",
      "type": "blob",
      "path": "samples-workitems/{queueTrigger}-Copy",
      "connection": "MyStorageConnectionAppSetting",
      "direction": "out"
    }
  ],
  "disabled": false
}
``` 

The [configuration](#input---configuration) section explains these properties.

Here's the JavaScript code:

```javascript
module.exports = function(context) {
    context.log('Node.js Queue trigger function processed', context.bindings.myQueueItem);
    context.bindings.myOutputBlob = context.bindings.myInputBlob;
    context.done();
};
```

## Input - attributes

In [C# class libraries](functions-dotnet-class-library.md), use the [BlobAttribute](https://github.com/Azure/azure-webjobs-sdk/blob/master/src/Microsoft.Azure.WebJobs/BlobAttribute.cs).

The attribute's constructor takes the path to the blob and a `FileAccess` parameter indicating read or write, as shown in the following example:

```csharp
[FunctionName("BlobInput")]
public static void Run(
    [QueueTrigger("myqueue-items")] string myQueueItem,
    [Blob("samples-workitems/{queueTrigger}", FileAccess.Read)] Stream myBlob,
    TraceWriter log)
{
    log.Info($"BlobInput processed blob\n Name:{myQueueItem} \n Size: {myBlob.Length} bytes");
}

```

You can set the `Connection` property to specify the storage account to use, as shown in the following example:

```csharp
[FunctionName("BlobInput")]
public static void Run(
    [QueueTrigger("myqueue-items")] string myQueueItem,
    [Blob("samples-workitems/{queueTrigger}", FileAccess.Read, Connection = "StorageConnectionAppSetting")] Stream myBlob,
    TraceWriter log)
{
    log.Info($"BlobInput processed blob\n Name:{myQueueItem} \n Size: {myBlob.Length} bytes");
}
```

You can use the `StorageAccount` attribute to specify the storage account at class, method, or parameter level. For more information, see [Trigger - attributes](#trigger---attributes).

## Input - configuration

The following table explains the binding configuration properties that you set in the *function.json* file and the `Blob` attribute.

|function.json property | Attribute property |Description|
|---------|---------|----------------------|
|**type** | n/a | Must be set to `blob`. |
|**direction** | n/a | Must be set to `in`. Exceptions are noted in the [usage](#input---usage) section. |
|**name** | n/a | The name of the variable that represents the blob in function code.|
|**path** |**BlobPath** | The path to the blob. | 
|**connection** |**Connection**| The name of an app setting that contains the Storage connection string to use for this binding. If the app setting name begins with "AzureWebJobs", you can specify only the remainder of the name here. For example, if you set `connection` to "MyStorage", the Functions runtime looks for an app setting that is named "AzureWebJobsMyStorage." If you leave `connection` empty, the Functions runtime uses the default Storage connection string in the app setting that is named `AzureWebJobsStorage`.<br><br>The connection string must be for a general-purpose storage account, not a [blob-only storage account](../storage/common/storage-create-storage-account.md#blob-storage-accounts).|
|n/a | **Access** | Indicates whether you will be reading or writing. |

[!INCLUDE [app settings to local.settings.json](../../includes/functions-app-settings-local.md)]

## Input - usage

In C# and C# script, you can use the following parameter types for the blob input binding:

* `Stream`
* `TextReader`
* `string`
* `Byte[]`
* `CloudBlobContainer`
* `CloudBlobDirectory`
* `ICloudBlob`<sup>1</sup>
* `CloudBlockBlob`<sup>1</sup>
* `CloudPageBlob`<sup>1</sup>
* `CloudAppendBlob`<sup>1</sup>

<sup>1</sup> Requires "inout" binding `direction` in *function.json* or `FileAccess.ReadWrite` in a C# class library.

Binding to `string` or `Byte[]` is only recommended if the blob size is small, as the entire blob contents are loaded into memory. Generally, it is preferable to use a `Stream` or `CloudBlockBlob` type. For more information, see [Concurrency and memory usage](#trigger---concurrency-and-memory-usage) earlier in this article.

In JavaScript, access the blob data using `context.bindings.<name from function.json>`.

## Output

Use Blob storage output bindings to write blobs.

## Output - example

See the language-specific example:

* [C#](#output---c-example)
* [C# script (.csx)](#output---c-script-example)
* [JavaScript](#output---javascript-example)

### Output - C# example

The following example is a [C# function](functions-dotnet-class-library.md) that uses a blob trigger and two output blob bindings. The function is triggered by the creation of an image blob in the *sample-images* container. It creates small and medium size copies of the image blob. 

```csharp
[FunctionName("ResizeImage")]
public static void Run(
    [BlobTrigger("sample-images/{name}")] Stream image, 
    [Blob("sample-images-sm/{name}", FileAccess.Write)] Stream imageSmall, 
    [Blob("sample-images-md/{name}", FileAccess.Write)] Stream imageMedium)
{
    var imageBuilder = ImageResizer.ImageBuilder.Current;
    var size = imageDimensionsTable[ImageSize.Small];

    imageBuilder.Build(image, imageSmall,
        new ResizeSettings(size.Item1, size.Item2, FitMode.Max, null), false);

    image.Position = 0;
    size = imageDimensionsTable[ImageSize.Medium];

    imageBuilder.Build(image, imageMedium,
        new ResizeSettings(size.Item1, size.Item2, FitMode.Max, null), false);
}

public enum ImageSize { ExtraSmall, Small, Medium }

private static Dictionary<ImageSize, (int, int)> imageDimensionsTable = new Dictionary<ImageSize, (int, int)>() {
    { ImageSize.ExtraSmall, (320, 200) },
    { ImageSize.Small,      (640, 400) },
    { ImageSize.Medium,     (800, 600) }
};
```        

### Output - C# script example

<!--Same example for input and output. -->

The following example shows blob input and output bindings in a *function.json* file and [C# script (.csx)](functions-reference-csharp.md) code that uses the bindings. The function makes a copy of a text blob. The function is triggered by a queue message that contains the name of the blob to copy. The new blob is named *{originalblobname}-Copy*.

In the *function.json* file, the `queueTrigger` metadata property is used to specify the blob name in the `path` properties:

```json
{
  "bindings": [
    {
      "queueName": "myqueue-items",
      "connection": "MyStorageConnectionAppSetting",
      "name": "myQueueItem",
      "type": "queueTrigger",
      "direction": "in"
    },
    {
      "name": "myInputBlob",
      "type": "blob",
      "path": "samples-workitems/{queueTrigger}",
      "connection": "MyStorageConnectionAppSetting",
      "direction": "in"
    },
    {
      "name": "myOutputBlob",
      "type": "blob",
      "path": "samples-workitems/{queueTrigger}-Copy",
      "connection": "MyStorageConnectionAppSetting",
      "direction": "out"
    }
  ],
  "disabled": false
}
``` 

The [configuration](#output---configuration) section explains these properties.

Here's the C# script code:

```cs
public static void Run(string myQueueItem, string myInputBlob, out string myOutputBlob, TraceWriter log)
{
    log.Info($"C# Queue trigger function processed: {myQueueItem}");
    myOutputBlob = myInputBlob;
}
```

### Output - JavaScript example

<!--Same example for input and output. -->

The following example shows blob input and output bindings in a *function.json* file and [JavaScript code] (functions-reference-node.md) that uses the bindings. The function makes a copy of a blob. The function is triggered by a queue message that contains the name of the blob to copy. The new blob is named *{originalblobname}-Copy*.

In the *function.json* file, the `queueTrigger` metadata property is used to specify the blob name in the `path` properties:

```json
{
  "bindings": [
    {
      "queueName": "myqueue-items",
      "connection": "MyStorageConnectionAppSetting",
      "name": "myQueueItem",
      "type": "queueTrigger",
      "direction": "in"
    },
    {
      "name": "myInputBlob",
      "type": "blob",
      "path": "samples-workitems/{queueTrigger}",
      "connection": "MyStorageConnectionAppSetting",
      "direction": "in"
    },
    {
      "name": "myOutputBlob",
      "type": "blob",
      "path": "samples-workitems/{queueTrigger}-Copy",
      "connection": "MyStorageConnectionAppSetting",
      "direction": "out"
    }
  ],
  "disabled": false
}
``` 

The [configuration](#output---configuration) section explains these properties.

Here's the JavaScript code:

```javascript
module.exports = function(context) {
    context.log('Node.js Queue trigger function processed', context.bindings.myQueueItem);
    context.bindings.myOutputBlob = context.bindings.myInputBlob;
    context.done();
};
```

## Output - attributes

In [C# class libraries](functions-dotnet-class-library.md), use the [BlobAttribute](https://github.com/Azure/azure-webjobs-sdk/blob/master/src/Microsoft.Azure.WebJobs/BlobAttribute.cs).

The attribute's constructor takes the path to the blob and a `FileAccess` parameter indicating read or write, as shown in the following example:

```csharp
[FunctionName("ResizeImage")]
public static void Run(
    [BlobTrigger("sample-images/{name}")] Stream image, 
    [Blob("sample-images-md/{name}", FileAccess.Write)] Stream imageSmall)
{
    ...
}
```

You can set the `Connection` property to specify the storage account to use, as shown in the following example:

```csharp
[FunctionName("ResizeImage")]
public static void Run(
    [BlobTrigger("sample-images/{name}")] Stream image, 
    [Blob("sample-images-md/{name}", FileAccess.Write, Connection = "StorageConnectionAppSetting")] Stream imageSmall)
{
    ...
}
```

For a complete example, see [Output - C# example](#output---c-example).

You can use the `StorageAccount` attribute to specify the storage account at class, method, or parameter level. For more information, see [Trigger - attributes](#trigger---attributes).

## Output - configuration

The following table explains the binding configuration properties that you set in the *function.json* file and the `Blob` attribute.

|function.json property | Attribute property |Description|
|---------|---------|----------------------|
|**type** | n/a | Must be set to `blob`. |
|**direction** | n/a | Must be set to `out` for an output binding. Exceptions are noted in the [usage](#output---usage) section. |
|**name** | n/a | The name of the variable that represents the blob in function code.  Set to `$return` to reference the function return value.|
|**path** |**BlobPath** | The path to the blob. | 
|**connection** |**Connection**| The name of an app setting that contains the Storage connection string to use for this binding. If the app setting name begins with "AzureWebJobs", you can specify only the remainder of the name here. For example, if you set `connection` to "MyStorage", the Functions runtime looks for an app setting that is named "AzureWebJobsMyStorage." If you leave `connection` empty, the Functions runtime uses the default Storage connection string in the app setting that is named `AzureWebJobsStorage`.<br><br>The connection string must be for a general-purpose storage account, not a [blob-only storage account](../storage/common/storage-create-storage-account.md#blob-storage-accounts).|
|n/a | **Access** | Indicates whether you will be reading or writing. |

[!INCLUDE [app settings to local.settings.json](../../includes/functions-app-settings-local.md)]

## Output - usage

In C# and C# script, you can bind to the following types to write blobs:

* `TextWriter`
* `out string`
* `out Byte[]`
* `CloudBlobStream`
* `Stream`
* `CloudBlobContainer`<sup>1</sup>
* `CloudBlobDirectory`
* `ICloudBlob`<sup>2</sup>
* `CloudBlockBlob`<sup>2</sup>
* `CloudPageBlob`<sup>2</sup>
* `CloudAppendBlob`<sup>2</sup>

<sup>1</sup> Requires "in" binding `direction` in *function.json* or `FileAccess.Read` in a C# class library.

<sup>2</sup> Requires "inout" binding `direction` in *function.json* or `FileAccess.ReadWrite` in a C# class library.

In async functions, use the return value or `IAsyncCollector` instead of an `out` parameter.

Binding to `string` or `Byte[]` is only recommended if the blob size is small, as the entire blob contents are loaded into memory. Generally, it is preferable to use a `Stream` or `CloudBlockBlob` type. For more information, see [Concurrency and memory usage](#trigger---concurrency-and-memory-usage) earlier in this article.


In JavaScript, access the blob data using `context.bindings.<name from function.json>`.

## Exceptions and return codes

| Binding |  Reference |
|---|---|
| Blob | [Blob Error Codes](https://docs.microsoft.com/rest/api/storageservices/fileservices/blob-service-error-codes) |
| Blob, Table, Queue |  [Storage Error Codes](https://docs.microsoft.com/rest/api/storageservices/fileservices/common-rest-api-error-codes) |
| Blob, Table, Queue |  [Troubleshooting](https://docs.microsoft.com/rest/api/storageservices/fileservices/troubleshooting-api-operations) |

## Next steps

> [!div class="nextstepaction"]
> [Go to a quickstart that uses a Blob storage trigger](functions-create-storage-blob-triggered-function.md)

> [!div class="nextstepaction"]
> [Learn more about Azure functions triggers and bindings](functions-triggers-bindings.md)
