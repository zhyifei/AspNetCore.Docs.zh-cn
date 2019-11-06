---
title: ASP.NET Core 中的内存管理和模式
author: rick-anderson
description: 了解 ASP.NET Core 如何管理内存，以及垃圾回收器（GC）的工作方式。
ms.author: riande
ms.custom: mvc
ms.date: 11/05/2019
uid: performance/memory
ms.openlocfilehash: 8f6b47ecde6f265bfb9437234b89f11f7d235869
ms.sourcegitcommit: 6628cd23793b66e4ce88788db641a5bbf470c3c1
ms.translationtype: MT
ms.contentlocale: zh-CN
ms.lasthandoff: 11/06/2019
ms.locfileid: "73660018"
---
# <a name="memory-management-and-garbage-collection-gc-in-aspnet-core"></a><span data-ttu-id="77abd-103">ASP.NET Core 中的内存管理和垃圾回收（GC）</span><span class="sxs-lookup"><span data-stu-id="77abd-103">Memory management and garbage collection (GC) in ASP.NET Core</span></span>

<span data-ttu-id="77abd-104">作者： [Sébastien Ros](https://github.com/sebastienros)和[Rick Anderson](https://twitter.com/RickAndMSFT)</span><span class="sxs-lookup"><span data-stu-id="77abd-104">By [Sébastien Ros](https://github.com/sebastienros) and [Rick Anderson](https://twitter.com/RickAndMSFT)</span></span>

<span data-ttu-id="77abd-105">内存管理是很复杂的，即使在 .NET 等托管框架中也是如此。</span><span class="sxs-lookup"><span data-stu-id="77abd-105">Memory management is complex, even in a managed framework like .NET.</span></span> <span data-ttu-id="77abd-106">分析和了解内存问题可能非常困难。</span><span class="sxs-lookup"><span data-stu-id="77abd-106">Analyzing and understanding memory issues can be challenging.</span></span> <span data-ttu-id="77abd-107">本文内容：</span><span class="sxs-lookup"><span data-stu-id="77abd-107">This article:</span></span>

* <span data-ttu-id="77abd-108">有很多*内存泄漏*和*GC 不起作用*的问题。</span><span class="sxs-lookup"><span data-stu-id="77abd-108">Was motivated by many *memory leak* and *GC not working* issues.</span></span> <span data-ttu-id="77abd-109">其中的大多数问题都是由不了解 .NET Core 中的内存使用情况或不了解其测量方式导致的。</span><span class="sxs-lookup"><span data-stu-id="77abd-109">Most of these issues were caused by not understanding how memory consumption works in .NET Core, or not understanding how it's measured.</span></span>
* <span data-ttu-id="77abd-110">演示内存使用情况，并提出替代方法。</span><span class="sxs-lookup"><span data-stu-id="77abd-110">Demonstrates problematic memory use, and suggests alternative approaches.</span></span>

## <a name="how-garbage-collection-gc-works-in-net-core"></a><span data-ttu-id="77abd-111">如何在 .NET Core 中使用垃圾回收（GC）</span><span class="sxs-lookup"><span data-stu-id="77abd-111">How garbage collection (GC) works in .NET Core</span></span>

<span data-ttu-id="77abd-112">GC 分配堆段，其中每个段都是一系列连续的内存。</span><span class="sxs-lookup"><span data-stu-id="77abd-112">The GC allocates heap segments where each segment is a contiguous range of memory.</span></span> <span data-ttu-id="77abd-113">位于堆中的对象归类为三个代之一：0、1或2。</span><span class="sxs-lookup"><span data-stu-id="77abd-113">Objects placed in the heap are categorized into one of 3 generations: 0, 1, or 2.</span></span> <span data-ttu-id="77abd-114">该代确定 GC 尝试释放应用程序不再引用的托管对象上内存的频率。</span><span class="sxs-lookup"><span data-stu-id="77abd-114">The generation determines the frequency the GC attempts to release memory on managed objects that are no longer referenced by the app.</span></span> <span data-ttu-id="77abd-115">较低编号的生成更为频繁。</span><span class="sxs-lookup"><span data-stu-id="77abd-115">Lower numbered generations are GC'd more frequently.</span></span>

<span data-ttu-id="77abd-116">根据对象的生存期，将对象从一代移到另一代。</span><span class="sxs-lookup"><span data-stu-id="77abd-116">Objects are moved from one generation to another based on their lifetime.</span></span> <span data-ttu-id="77abd-117">随着对象的运行时间较长，它们会移到较高的代中。</span><span class="sxs-lookup"><span data-stu-id="77abd-117">As objects live longer, they are moved into a higher generation.</span></span> <span data-ttu-id="77abd-118">如前所述，较高的版本是不太常见的垃圾回收。</span><span class="sxs-lookup"><span data-stu-id="77abd-118">As mentioned previously, higher generations are GC'd less often.</span></span> <span data-ttu-id="77abd-119">短期生存期的对象始终保留在第0代中。</span><span class="sxs-lookup"><span data-stu-id="77abd-119">Short term lived objects always remain in generation 0.</span></span> <span data-ttu-id="77abd-120">例如，在 web 请求过程中引用的对象的生存期很短。</span><span class="sxs-lookup"><span data-stu-id="77abd-120">For example, objects that are referenced during the life of a web request are short lived.</span></span> <span data-ttu-id="77abd-121">应用程序级别[单一实例](xref:fundamentals/dependency-injection#service-lifetimes)通常迁移到第2代。</span><span class="sxs-lookup"><span data-stu-id="77abd-121">Application level [singletons](xref:fundamentals/dependency-injection#service-lifetimes) generally migrate to generation 2.</span></span>

<span data-ttu-id="77abd-122">当 ASP.NET Core 应用启动时，GC：</span><span class="sxs-lookup"><span data-stu-id="77abd-122">When an ASP.NET Core app starts, the GC:</span></span>

* <span data-ttu-id="77abd-123">为初始堆段保留一些内存。</span><span class="sxs-lookup"><span data-stu-id="77abd-123">Reserves some memory for the initial heap segments.</span></span>
* <span data-ttu-id="77abd-124">加载运行时，提交一小部分内存。</span><span class="sxs-lookup"><span data-stu-id="77abd-124">Commits a small portion of memory when the runtime is loaded.</span></span>

<span data-ttu-id="77abd-125">出于性能方面的原因，上述内存分配已完成。</span><span class="sxs-lookup"><span data-stu-id="77abd-125">The preceding memory allocations are done for performance reasons.</span></span> <span data-ttu-id="77abd-126">性能优势来自连续内存中的堆段。</span><span class="sxs-lookup"><span data-stu-id="77abd-126">The performance benefit comes from heap segments in contiguous memory.</span></span>

### <a name="call-gccollect"></a><span data-ttu-id="77abd-127">调用 GC。收集</span><span class="sxs-lookup"><span data-stu-id="77abd-127">Call GC.Collect</span></span>

<span data-ttu-id="77abd-128">调用[GC。显式收集](xref:System.GC.Collect*)：</span><span class="sxs-lookup"><span data-stu-id="77abd-128">Calling [GC.Collect](xref:System.GC.Collect*) explicitly:</span></span>

* <span data-ttu-id="77abd-129">**不**应由生产 ASP.NET Core 应用完成。</span><span class="sxs-lookup"><span data-stu-id="77abd-129">Should **not** be done by production ASP.NET Core apps.</span></span>
* <span data-ttu-id="77abd-130">调查内存泄漏时非常有用。</span><span class="sxs-lookup"><span data-stu-id="77abd-130">Is useful when investigating memory leaks.</span></span>
* <span data-ttu-id="77abd-131">调查时，验证 GC 是否已从内存中删除所有无关联的对象，以便可以测量内存。</span><span class="sxs-lookup"><span data-stu-id="77abd-131">When investigating, verifies the GC has removed all dangling objects from memory so memory can be measured.</span></span>

## <a name="analyzing-the-memory-usage-of-an-app"></a><span data-ttu-id="77abd-132">分析应用的内存使用情况</span><span class="sxs-lookup"><span data-stu-id="77abd-132">Analyzing the memory usage of an app</span></span>

<span data-ttu-id="77abd-133">专用工具可帮助分析内存使用量：</span><span class="sxs-lookup"><span data-stu-id="77abd-133">Dedicated tools can help analyzing memory usage:</span></span>

- <span data-ttu-id="77abd-134">计算对象引用数</span><span class="sxs-lookup"><span data-stu-id="77abd-134">Counting object references</span></span>
- <span data-ttu-id="77abd-135">度量 GC 对 CPU 使用的影响程度</span><span class="sxs-lookup"><span data-stu-id="77abd-135">Measuring how much impact the GC has on CPU usage</span></span>
- <span data-ttu-id="77abd-136">测量每代使用的内存空间</span><span class="sxs-lookup"><span data-stu-id="77abd-136">Measuring memory space used for each generation</span></span>

<span data-ttu-id="77abd-137">使用以下工具分析内存使用量：</span><span class="sxs-lookup"><span data-stu-id="77abd-137">Use the following tools to analyze memory usage:</span></span>

* <span data-ttu-id="77abd-138">[dotnet](/dotnet/core/diagnostics/dotnet-trace)：可在生产计算机上使用。</span><span class="sxs-lookup"><span data-stu-id="77abd-138">[dotnet-trace](/dotnet/core/diagnostics/dotnet-trace): Can be  used on production machines.</span></span>
* [<span data-ttu-id="77abd-139">在不使用 Visual Studio 调试器的情况下分析内存使用情况</span><span class="sxs-lookup"><span data-stu-id="77abd-139">Analyze memory usage without the Visual Studio debugger</span></span>](/visualstudio/profiling/memory-usage-without-debugging2)
* [<span data-ttu-id="77abd-140">Visual Studio 中的配置文件内存使用</span><span class="sxs-lookup"><span data-stu-id="77abd-140">Profile memory usage in Visual Studio</span></span>](/visualstudio/profiling/memory-usage)

### <a name="detecting-memory-issues"></a><span data-ttu-id="77abd-141">检测内存问题</span><span class="sxs-lookup"><span data-stu-id="77abd-141">Detecting memory issues</span></span>

<span data-ttu-id="77abd-142">任务管理器可用于了解 ASP.NET 应用正在使用的内存量。</span><span class="sxs-lookup"><span data-stu-id="77abd-142">Task Manager can be used to get an idea of how much memory an ASP.NET app is using.</span></span> <span data-ttu-id="77abd-143">任务管理器内存值：</span><span class="sxs-lookup"><span data-stu-id="77abd-143">The Task Manager memory value:</span></span>

* <span data-ttu-id="77abd-144">表示 ASP.NET 进程使用的内存量。</span><span class="sxs-lookup"><span data-stu-id="77abd-144">Represents the amount of memory that is used by the ASP.NET process.</span></span>
* <span data-ttu-id="77abd-145">包括应用的活对象和其他内存使用者（如本机内存使用情况）。</span><span class="sxs-lookup"><span data-stu-id="77abd-145">Includes the app's living objects and other memory consumers such as native memory usage.</span></span>

<span data-ttu-id="77abd-146">如果任务管理器内存值无限增加且从未平展，则应用程序的内存泄漏。</span><span class="sxs-lookup"><span data-stu-id="77abd-146">If the Task Manager memory value increases indefinitely and never flattens out, the app has a memory leak.</span></span> <span data-ttu-id="77abd-147">以下部分演示并解释了几种内存使用模式。</span><span class="sxs-lookup"><span data-stu-id="77abd-147">The following sections demonstrate and explain several memory usage patterns.</span></span>

## <a name="sample-display-memory-usage-app"></a><span data-ttu-id="77abd-148">示例显示内存使用情况应用</span><span class="sxs-lookup"><span data-stu-id="77abd-148">Sample display memory usage app</span></span>

<span data-ttu-id="77abd-149">GitHub 上提供了[MemoryLeak 示例应用](https://github.com/sebastienros/memoryleak)。</span><span class="sxs-lookup"><span data-stu-id="77abd-149">The [MemoryLeak sample app](https://github.com/sebastienros/memoryleak) is available on GitHub.</span></span> <span data-ttu-id="77abd-150">MemoryLeak 应用：</span><span class="sxs-lookup"><span data-stu-id="77abd-150">The MemoryLeak app:</span></span>

* <span data-ttu-id="77abd-151">包括一个收集应用程序的实时内存和 GC 数据的诊断控制器。</span><span class="sxs-lookup"><span data-stu-id="77abd-151">Includes a diagnostic controller that gathers real-time memory and GC data for the app.</span></span>
* <span data-ttu-id="77abd-152">具有显示内存和 GC 数据的索引页。</span><span class="sxs-lookup"><span data-stu-id="77abd-152">Has an Index page that displays the memory and GC data.</span></span> <span data-ttu-id="77abd-153">索引页每秒刷新一次。</span><span class="sxs-lookup"><span data-stu-id="77abd-153">The Index page is refreshed every second.</span></span>
* <span data-ttu-id="77abd-154">包含提供各种内存负载模式的 API 控制器。</span><span class="sxs-lookup"><span data-stu-id="77abd-154">Contains an API controller that provides various memory load patterns.</span></span>
* <span data-ttu-id="77abd-155">不是受支持的工具，但它可用于显示 ASP.NET Core 应用的内存使用模式。</span><span class="sxs-lookup"><span data-stu-id="77abd-155">Is not a supported tool, however, it can be used to display memory usage patterns of ASP.NET Core apps.</span></span>

<span data-ttu-id="77abd-156">运行 MemoryLeak。</span><span class="sxs-lookup"><span data-stu-id="77abd-156">Run MemoryLeak.</span></span> <span data-ttu-id="77abd-157">分配的内存缓慢增加，直到 GC 发生。</span><span class="sxs-lookup"><span data-stu-id="77abd-157">Allocated memory slowly increases until a GC occurs.</span></span> <span data-ttu-id="77abd-158">内存增加是因为该工具分配自定义对象来捕获数据。</span><span class="sxs-lookup"><span data-stu-id="77abd-158">Memory increases because the tool allocates custom object to capture data.</span></span> <span data-ttu-id="77abd-159">下图显示了 Gen 0 GC 发生时的 MemoryLeak 索引页。</span><span class="sxs-lookup"><span data-stu-id="77abd-159">The following image shows the MemoryLeak Index page when a Gen 0 GC occurs.</span></span> <span data-ttu-id="77abd-160">此图表显示 0 RPS （每秒请求数），因为未调用 API 控制器中的任何 API 终结点。</span><span class="sxs-lookup"><span data-stu-id="77abd-160">The chart shows 0 RPS (Requests per second) because no API endpoints from the API controller have been called.</span></span>

![上图](memory/_static/0RPS.png)

<span data-ttu-id="77abd-162">此图表显示内存使用量的两个值：</span><span class="sxs-lookup"><span data-stu-id="77abd-162">The chart displays two values for the memory usage:</span></span>

- <span data-ttu-id="77abd-163">已分配：托管对象占用的内存量</span><span class="sxs-lookup"><span data-stu-id="77abd-163">Allocated: the amount of memory occupied by managed objects</span></span>
- <span data-ttu-id="77abd-164">工作集：进程使用的总物理内存（RAM）。</span><span class="sxs-lookup"><span data-stu-id="77abd-164">Working set: the total physical memory (RAM) used by the process.</span></span> <span data-ttu-id="77abd-165">显示的工作集是任务管理器可以显示的相同值。</span><span class="sxs-lookup"><span data-stu-id="77abd-165">The working set shown is the same value Task Manager can display.</span></span>

### <a name="transient-objects"></a><span data-ttu-id="77abd-166">暂时性对象</span><span class="sxs-lookup"><span data-stu-id="77abd-166">Transient objects</span></span>

<span data-ttu-id="77abd-167">以下 API 创建一个 10 KB 的字符串实例，并将其返回给客户端。</span><span class="sxs-lookup"><span data-stu-id="77abd-167">The following API creates a 10-KB String instance and returns it to the client.</span></span> <span data-ttu-id="77abd-168">对于每个请求，将在内存中分配一个新的对象，并将其写入响应中。</span><span class="sxs-lookup"><span data-stu-id="77abd-168">On each request, a new object is allocated in memory and written to the response.</span></span> <span data-ttu-id="77abd-169">字符串作为 UTF-16 字符存储在 .NET 中，因此每个字符需要2个字节的内存。</span><span class="sxs-lookup"><span data-stu-id="77abd-169">Strings are stored as UTF-16 characters in .NET so each character takes 2 bytes in memory.</span></span>

```csharp
[HttpGet("bigstring")]
public ActionResult<string> GetBigString()
{
    return new String('x', 10 * 1024);
}
```

<span data-ttu-id="77abd-170">下面的关系图是使用相对较小的负载生成的，用于显示 GC 如何影响内存分配。</span><span class="sxs-lookup"><span data-stu-id="77abd-170">The following graph is generated with a relatively small load in to show how memory allocations are impacted by the GC.</span></span>

![上图](memory/_static/bigstring.png)

<span data-ttu-id="77abd-172">上面的图表显示：</span><span class="sxs-lookup"><span data-stu-id="77abd-172">The preceding chart shows:</span></span>

* <span data-ttu-id="77abd-173">4K RPS （每秒请求数）。</span><span class="sxs-lookup"><span data-stu-id="77abd-173">4K RPS (Requests per second).</span></span>
* <span data-ttu-id="77abd-174">第0代垃圾回收大约每两秒发生一次。</span><span class="sxs-lookup"><span data-stu-id="77abd-174">Generation 0 GC collections occur about every two seconds.</span></span>
* <span data-ttu-id="77abd-175">工作集的大小约为 500 MB。</span><span class="sxs-lookup"><span data-stu-id="77abd-175">The working set is constant at approximately 500 MB.</span></span>
* <span data-ttu-id="77abd-176">CPU 为12%。</span><span class="sxs-lookup"><span data-stu-id="77abd-176">CPU is 12%.</span></span>
* <span data-ttu-id="77abd-177">内存消耗和发布（通过 GC）是稳定的。</span><span class="sxs-lookup"><span data-stu-id="77abd-177">The memory consumption and release (through GC) is stable.</span></span>

<span data-ttu-id="77abd-178">以下图表采用可由计算机处理的最大吞吐量。</span><span class="sxs-lookup"><span data-stu-id="77abd-178">The following chart is taken at the max throughput that can be handled by the machine.</span></span>

![上图](memory/_static/bigstring2.png)

<span data-ttu-id="77abd-180">上面的图表显示：</span><span class="sxs-lookup"><span data-stu-id="77abd-180">The preceding chart shows:</span></span>

* <span data-ttu-id="77abd-181">22K RPS</span><span class="sxs-lookup"><span data-stu-id="77abd-181">22K RPS</span></span>
* <span data-ttu-id="77abd-182">第0代垃圾回收每秒发生多次。</span><span class="sxs-lookup"><span data-stu-id="77abd-182">Generation 0 GC collections occur several times per second.</span></span>
* <span data-ttu-id="77abd-183">由于每秒分配的内存量明显增加，因此将触发第1代回收。</span><span class="sxs-lookup"><span data-stu-id="77abd-183">Generation 1 collections are triggered because the app allocated significantly more memory per second.</span></span>
* <span data-ttu-id="77abd-184">工作集的大小约为 500 MB。</span><span class="sxs-lookup"><span data-stu-id="77abd-184">The working set is constant at approximately 500 MB.</span></span>
* <span data-ttu-id="77abd-185">CPU 为33%。</span><span class="sxs-lookup"><span data-stu-id="77abd-185">CPU is 33%.</span></span>
* <span data-ttu-id="77abd-186">内存消耗和发布（通过 GC）是稳定的。</span><span class="sxs-lookup"><span data-stu-id="77abd-186">The memory consumption and release (through GC) is stable.</span></span>
* <span data-ttu-id="77abd-187">CPU （33%）不会过度使用，因此垃圾回收可以跟上大量分配。</span><span class="sxs-lookup"><span data-stu-id="77abd-187">The CPU (33%) is not over-utilized, therefore the garbage collection can keep up with a high number of allocations.</span></span>

### <a name="workstation-gc-vs-server-gc"></a><span data-ttu-id="77abd-188">工作站 GC 与服务器 GC</span><span class="sxs-lookup"><span data-stu-id="77abd-188">Workstation GC vs. Server GC</span></span>

<span data-ttu-id="77abd-189">.NET 垃圾回收器具有两种不同的模式：</span><span class="sxs-lookup"><span data-stu-id="77abd-189">The .NET Garbage Collector has two different modes:</span></span>

* <span data-ttu-id="77abd-190">**工作站 GC**：针对桌面进行了优化。</span><span class="sxs-lookup"><span data-stu-id="77abd-190">**Workstation GC**: Optimized for the desktop.</span></span>
* <span data-ttu-id="77abd-191">**服务器 GC**。</span><span class="sxs-lookup"><span data-stu-id="77abd-191">**Server GC**.</span></span> <span data-ttu-id="77abd-192">ASP.NET Core 应用的默认 GC。</span><span class="sxs-lookup"><span data-stu-id="77abd-192">The default GC for ASP.NET Core apps.</span></span> <span data-ttu-id="77abd-193">针对服务器进行了优化。</span><span class="sxs-lookup"><span data-stu-id="77abd-193">Optimized for the server.</span></span>

<span data-ttu-id="77abd-194">GC 模式可以在项目文件中或在已发布应用的*runtimeconfig.template.json*文件中显式设置。</span><span class="sxs-lookup"><span data-stu-id="77abd-194">The GC mode can be set explicitly in the project file or in the *runtimeconfig.json* file of the published app.</span></span> <span data-ttu-id="77abd-195">以下标记显示了在项目文件中设置 `ServerGarbageCollection`：</span><span class="sxs-lookup"><span data-stu-id="77abd-195">The following markup shows setting `ServerGarbageCollection` in the project file:</span></span>

```xml
<PropertyGroup>
  <ServerGarbageCollection>true</ServerGarbageCollection>
</PropertyGroup>
```

<span data-ttu-id="77abd-196">更改项目文件中的 `ServerGarbageCollection` 需要重新生成应用。</span><span class="sxs-lookup"><span data-stu-id="77abd-196">Changing `ServerGarbageCollection` in the project file requires the app to be rebuilt.</span></span>

<span data-ttu-id="77abd-197">**注意：** 服务器垃圾回收在具有单个核心的计算机上**不可用。**</span><span class="sxs-lookup"><span data-stu-id="77abd-197">**Note:** Server garbage collection is **not** available on machines with a single core.</span></span> <span data-ttu-id="77abd-198">有关更多信息，请参见<xref:System.Runtime.GCSettings.IsServerGC>。</span><span class="sxs-lookup"><span data-stu-id="77abd-198">For more information, see <xref:System.Runtime.GCSettings.IsServerGC>.</span></span>

<span data-ttu-id="77abd-199">下图显示了使用工作站 GC 的占用大量 RPS 的内存配置文件。</span><span class="sxs-lookup"><span data-stu-id="77abd-199">The following image shows the memory profile under a 5K RPS using the Workstation GC.</span></span>

![上图](memory/_static/workstation.png)

<span data-ttu-id="77abd-201">此图表与服务器版本之间的区别非常重要：</span><span class="sxs-lookup"><span data-stu-id="77abd-201">The differences between this chart and the server version are significant:</span></span>

- <span data-ttu-id="77abd-202">工作集从 500 MB 降到 70 MB。</span><span class="sxs-lookup"><span data-stu-id="77abd-202">The working set drops from 500 MB to 70 MB.</span></span>
- <span data-ttu-id="77abd-203">GC 每秒生成0次（而不是每隔两秒）回收一次。</span><span class="sxs-lookup"><span data-stu-id="77abd-203">The GC does generation 0 collections multiple times per second instead of every two seconds.</span></span>
- <span data-ttu-id="77abd-204">GC 从 300 MB 降到 10 MB。</span><span class="sxs-lookup"><span data-stu-id="77abd-204">GC drops from 300 MB to 10 MB.</span></span>

<span data-ttu-id="77abd-205">在典型的 web 服务器环境中，CPU 使用率比内存更重要，因此服务器 GC 更好。</span><span class="sxs-lookup"><span data-stu-id="77abd-205">On a typical web server environment, CPU usage is more important than memory, therefore the Server GC is better.</span></span> <span data-ttu-id="77abd-206">如果内存使用率很高且 CPU 使用率相对较低，则工作站 GC 可能会更高的性能。</span><span class="sxs-lookup"><span data-stu-id="77abd-206">If memory utilization is high and CPU usage is relatively low, the Workstation GC might be more performant.</span></span> <span data-ttu-id="77abd-207">例如，在内存不足的情况下承载几个 web 应用的高密度。</span><span class="sxs-lookup"><span data-stu-id="77abd-207">For example, high density hosting several web apps where memory is scarce.</span></span>

### <a name="persistent-object-references"></a><span data-ttu-id="77abd-208">持久性对象引用</span><span class="sxs-lookup"><span data-stu-id="77abd-208">Persistent object references</span></span>

<span data-ttu-id="77abd-209">GC 无法释放所引用的对象。</span><span class="sxs-lookup"><span data-stu-id="77abd-209">The GC cannot free objects that are referenced.</span></span> <span data-ttu-id="77abd-210">引用但不再需要的对象将导致内存泄露。</span><span class="sxs-lookup"><span data-stu-id="77abd-210">Objects that are referenced but no longer needed result in a memory leak.</span></span> <span data-ttu-id="77abd-211">如果应用经常分配对象，但在不再需要对象之后无法释放它们，则内存使用量将随着时间的推移而增加。</span><span class="sxs-lookup"><span data-stu-id="77abd-211">If the app frequently allocates objects and fails to free them after they are no longer needed, memory usage will increase over time.</span></span>

<span data-ttu-id="77abd-212">以下 API 创建一个 10 KB 的字符串实例，并将其返回给客户端。</span><span class="sxs-lookup"><span data-stu-id="77abd-212">The following API creates a 10-KB String instance and returns it to the client.</span></span> <span data-ttu-id="77abd-213">与上一示例的不同之处在于，此实例由静态成员引用，这意味着它不能用于收集。</span><span class="sxs-lookup"><span data-stu-id="77abd-213">The difference with the previous example is that this instance is referenced by a static member, which means it's never available for collection.</span></span>

```csharp
private static ConcurrentBag<string> _staticStrings = new ConcurrentBag<string>();

[HttpGet("staticstring")]
public ActionResult<string> GetStaticString()
{
    var bigString = new String('x', 10 * 1024);
    _staticStrings.Add(bigString);
    return bigString;
}
```

<span data-ttu-id="77abd-214">前面的代码：</span><span class="sxs-lookup"><span data-stu-id="77abd-214">The preceding code:</span></span>

* <span data-ttu-id="77abd-215">典型内存泄漏的示例。</span><span class="sxs-lookup"><span data-stu-id="77abd-215">Is an example of a typical memory leak.</span></span>
* <span data-ttu-id="77abd-216">如果频繁调用，会导致应用内存增加，直到进程因 `OutOfMemory` 异常而崩溃。</span><span class="sxs-lookup"><span data-stu-id="77abd-216">With frequent calls, causes app memory to increases until the process crashes with an `OutOfMemory` exception.</span></span>

![上图](memory/_static/eternal.png)

<span data-ttu-id="77abd-218">在上图中：</span><span class="sxs-lookup"><span data-stu-id="77abd-218">In the preceding image:</span></span>

* <span data-ttu-id="77abd-219">负载测试 `/api/staticstring` 终结点会导致内存线性增加。</span><span class="sxs-lookup"><span data-stu-id="77abd-219">Load testing the `/api/staticstring` endpoint causes a linear increase in memory.</span></span>
* <span data-ttu-id="77abd-220">GC 在内存压力增加时，通过调用第2代回收来尝试释放内存。</span><span class="sxs-lookup"><span data-stu-id="77abd-220">The GC tries to free memory as the memory pressure grows, by calling a generation 2 collection.</span></span>
* <span data-ttu-id="77abd-221">GC 无法释放泄漏的内存。</span><span class="sxs-lookup"><span data-stu-id="77abd-221">The GC cannot free the leaked memory.</span></span> <span data-ttu-id="77abd-222">已分配和工作集增加了时间。</span><span class="sxs-lookup"><span data-stu-id="77abd-222">Allocated and working set increase with time.</span></span>

<span data-ttu-id="77abd-223">某些方案（如缓存）需要保留对象引用，直到内存压力强制释放它们。</span><span class="sxs-lookup"><span data-stu-id="77abd-223">Some scenarios, such as caching, require object references to be held until memory pressure forces them to be released.</span></span> <span data-ttu-id="77abd-224"><xref:System.WeakReference> 类可用于这种类型的缓存代码。</span><span class="sxs-lookup"><span data-stu-id="77abd-224">The <xref:System.WeakReference> class can be used for this type of caching code.</span></span> <span data-ttu-id="77abd-225">内存压力下将收集一个 `WeakReference` 对象。</span><span class="sxs-lookup"><span data-stu-id="77abd-225">A `WeakReference` object is collected under memory pressures.</span></span> <span data-ttu-id="77abd-226"><xref:Microsoft.Extensions.Caching.Memory.IMemoryCache> 的默认实现使用 `WeakReference`。</span><span class="sxs-lookup"><span data-stu-id="77abd-226">The default implementation of <xref:Microsoft.Extensions.Caching.Memory.IMemoryCache> uses `WeakReference`.</span></span>

### <a name="native-memory"></a><span data-ttu-id="77abd-227">本机内存</span><span class="sxs-lookup"><span data-stu-id="77abd-227">Native memory</span></span>

<span data-ttu-id="77abd-228">某些 .NET Core 对象依赖本机内存。</span><span class="sxs-lookup"><span data-stu-id="77abd-228">Some .NET Core objects rely on native memory.</span></span> <span data-ttu-id="77abd-229">GC**无法**收集本机内存。</span><span class="sxs-lookup"><span data-stu-id="77abd-229">Native memory can **not** be collected by the GC.</span></span> <span data-ttu-id="77abd-230">使用本机内存的 .NET 对象必须使用本机代码释放它。</span><span class="sxs-lookup"><span data-stu-id="77abd-230">The .NET object using native memory must free it using native code.</span></span>

<span data-ttu-id="77abd-231">.NET 提供了 <xref:System.IDisposable> 界面，使开发人员能够释放本机内存。</span><span class="sxs-lookup"><span data-stu-id="77abd-231">.NET provides the <xref:System.IDisposable> interface to let developers release native memory.</span></span> <span data-ttu-id="77abd-232">即使未调用 <xref:System.IDisposable.Dispose*>，正确实现的类也会在[终结器](/dotnet/csharp/programming-guide/classes-and-structs/destructors)运行时调用 `Dispose`。</span><span class="sxs-lookup"><span data-stu-id="77abd-232">Even if <xref:System.IDisposable.Dispose*> is not called, correctly implemented classes call `Dispose` when the [finalizer](/dotnet/csharp/programming-guide/classes-and-structs/destructors) runs.</span></span>

<span data-ttu-id="77abd-233">考虑下列代码：</span><span class="sxs-lookup"><span data-stu-id="77abd-233">Consider the following code:</span></span>

```csharp
[HttpGet("fileprovider")]
public void GetFileProvider()
{
    var fp = new PhysicalFileProvider(TempPath);
    fp.Watch("*.*");
}
```

<span data-ttu-id="77abd-234">[PhysicaFileProvider](/dotnet/api/microsoft.extensions.fileproviders.physicalfileprovider?view=dotnet-plat-ext-3.0)是托管类，因此将在请求结束时收集任何实例。</span><span class="sxs-lookup"><span data-stu-id="77abd-234">[PhysicaFileProvider](/dotnet/api/microsoft.extensions.fileproviders.physicalfileprovider?view=dotnet-plat-ext-3.0) is a managed class, so any instance will be collected at the end of the request.</span></span>

<span data-ttu-id="77abd-235">下图显示了连续调用 `fileprovider` API 时的内存配置文件。</span><span class="sxs-lookup"><span data-stu-id="77abd-235">The following image shows the memory profile while invoking the `fileprovider` API continuously.</span></span>

![上图](memory/_static/fileprovider.png)

<span data-ttu-id="77abd-237">上面的图表显示了此类的实现的一个明显问题，因为它会不断增加内存使用量。</span><span class="sxs-lookup"><span data-stu-id="77abd-237">The preceding chart shows an obvious issue with the implementation of this class, as it keeps increasing memory usage.</span></span> <span data-ttu-id="77abd-238">这是[此问题](https://github.com/aspnet/Home/issues/3110)中正在跟踪的已知问题。</span><span class="sxs-lookup"><span data-stu-id="77abd-238">This is a known problem that is being tracked in [this issue](https://github.com/aspnet/Home/issues/3110).</span></span>

<span data-ttu-id="77abd-239">可以通过以下方式之一在用户代码中发生相同的泄漏：</span><span class="sxs-lookup"><span data-stu-id="77abd-239">The same leak could be happen in user code, by one of the following:</span></span>

* <span data-ttu-id="77abd-240">不能正确释放类。</span><span class="sxs-lookup"><span data-stu-id="77abd-240">Not releasing the class correctly.</span></span>
* <span data-ttu-id="77abd-241">忘记调用应释放的依赖对象的 `Dispose`方法。</span><span class="sxs-lookup"><span data-stu-id="77abd-241">Forgetting to invoke the `Dispose`method of the dependent objects that should be disposed.</span></span>

### <a name="large-objects-heap"></a><span data-ttu-id="77abd-242">大型对象堆</span><span class="sxs-lookup"><span data-stu-id="77abd-242">Large objects heap</span></span>

<span data-ttu-id="77abd-243">频繁的内存分配/空闲周期可以分段内存，尤其是在分配大块内存时。</span><span class="sxs-lookup"><span data-stu-id="77abd-243">Frequent memory allocation/free cycles can fragment memory, especially when allocating large chunks of memory.</span></span> <span data-ttu-id="77abd-244">对象在连续内存块中分配。</span><span class="sxs-lookup"><span data-stu-id="77abd-244">Objects are allocated in contiguous blocks of memory.</span></span> <span data-ttu-id="77abd-245">为了缓解碎片，当 GC 释放内存时，它会 trys 对内存进行碎片整理。</span><span class="sxs-lookup"><span data-stu-id="77abd-245">To mitigate fragmentation, when the GC frees memory, it trys to defragment it.</span></span> <span data-ttu-id="77abd-246">此过程称为**压缩**。</span><span class="sxs-lookup"><span data-stu-id="77abd-246">This process is called **compaction**.</span></span> <span data-ttu-id="77abd-247">压缩涉及移动对象。</span><span class="sxs-lookup"><span data-stu-id="77abd-247">Compaction involves moving objects.</span></span> <span data-ttu-id="77abd-248">移动大型对象会对性能产生负面影响。</span><span class="sxs-lookup"><span data-stu-id="77abd-248">Moving large objects imposes a performance penalty.</span></span> <span data-ttu-id="77abd-249">出于此原因，GC 将为_大型_对象（称为[大型对象堆](/dotnet/standard/garbage-collection/large-object-heap)（LOH））创建特殊的内存区域。</span><span class="sxs-lookup"><span data-stu-id="77abd-249">For this reason the GC creates a special memory zone for _large_ objects, called the [large object heap](/dotnet/standard/garbage-collection/large-object-heap) (LOH).</span></span> <span data-ttu-id="77abd-250">大于85000字节（大约 83 KB）的对象为：</span><span class="sxs-lookup"><span data-stu-id="77abd-250">Objects that are greater than 85,000 bytes (approximately 83 KB) are:</span></span>

* <span data-ttu-id="77abd-251">放置在 LOH 上。</span><span class="sxs-lookup"><span data-stu-id="77abd-251">Placed on the LOH.</span></span>
* <span data-ttu-id="77abd-252">未压缩。</span><span class="sxs-lookup"><span data-stu-id="77abd-252">Not compacted.</span></span>
* <span data-ttu-id="77abd-253">在第2代 Gc 期间收集。</span><span class="sxs-lookup"><span data-stu-id="77abd-253">Collected during generation 2 GCs.</span></span>

<span data-ttu-id="77abd-254">当 LOH 已满时，GC 将触发第2代回收。</span><span class="sxs-lookup"><span data-stu-id="77abd-254">When the LOH is full, the GC will trigger a generation 2 collection.</span></span> <span data-ttu-id="77abd-255">第2代回收：</span><span class="sxs-lookup"><span data-stu-id="77abd-255">Generation 2 collections:</span></span>

* <span data-ttu-id="77abd-256">的速度非常慢。</span><span class="sxs-lookup"><span data-stu-id="77abd-256">Are inherently slow.</span></span>
* <span data-ttu-id="77abd-257">此外，还会产生在所有其他代上触发集合的成本。</span><span class="sxs-lookup"><span data-stu-id="77abd-257">Additionally incur the cost of triggering a collection on all other generations.</span></span>

<span data-ttu-id="77abd-258">以下代码会立即压缩 LOH：</span><span class="sxs-lookup"><span data-stu-id="77abd-258">The following code compacts the LOH immediately:</span></span>

```csharp
GCSettings.LargeObjectHeapCompactionMode = GCLargeObjectHeapCompactionMode.CompactOnce;
GC.Collect();
```

<span data-ttu-id="77abd-259">有关压缩 LOH 的信息，请参阅 <xref:System.Runtime.GCSettings.LargeObjectHeapCompactionMode>。</span><span class="sxs-lookup"><span data-stu-id="77abd-259">See <xref:System.Runtime.GCSettings.LargeObjectHeapCompactionMode> for information on compacting the LOH.</span></span>

<span data-ttu-id="77abd-260">在使用 .NET Core 3.0 和更高版本的容器中，LOH 将自动压缩。</span><span class="sxs-lookup"><span data-stu-id="77abd-260">In containers using .NET Core 3.0 and later, the LOH is automatically compacted.</span></span>

<span data-ttu-id="77abd-261">以下 API 演示了此行为：</span><span class="sxs-lookup"><span data-stu-id="77abd-261">The following API that illustrates this behavior:</span></span>

```csharp
[HttpGet("loh/{size=85000}")]
public int GetLOH1(int size)
{
   return new byte[size].Length;
}
```

<span data-ttu-id="77abd-262">下图显示了在最大负载下调用 `/api/loh/84975` 终结点的内存配置文件：</span><span class="sxs-lookup"><span data-stu-id="77abd-262">The following chart shows the memory profile of calling the `/api/loh/84975` endpoint, under maximum load:</span></span>

![上图](memory/_static/loh1.png)

<span data-ttu-id="77abd-264">下图显示了调用 `/api/loh/84976` 终结点的内存配置文件，*只分配一个字节*：</span><span class="sxs-lookup"><span data-stu-id="77abd-264">The following chart shows the memory profile of calling the `/api/loh/84976` endpoint, allocating *just one more byte*:</span></span>

![上图](memory/_static/loh2.png)

<span data-ttu-id="77abd-266">注意： `byte[]` 结构具有开销字节。</span><span class="sxs-lookup"><span data-stu-id="77abd-266">Note: The `byte[]` structure has overhead bytes.</span></span> <span data-ttu-id="77abd-267">这就是84976字节触发85000限制的原因。</span><span class="sxs-lookup"><span data-stu-id="77abd-267">That's why 84,976 bytes triggers the 85,000 limit.</span></span>

<span data-ttu-id="77abd-268">比较上述两个图表：</span><span class="sxs-lookup"><span data-stu-id="77abd-268">Comparing the two preceding charts:</span></span>

* <span data-ttu-id="77abd-269">对于这两种方案（约 450 MB），工作集都是类似的。</span><span class="sxs-lookup"><span data-stu-id="77abd-269">The working set is similar for both scenarios, about 450 MB.</span></span>
* <span data-ttu-id="77abd-270">LOH 请求（84975字节）下面显示第0代回收。</span><span class="sxs-lookup"><span data-stu-id="77abd-270">The under LOH requests (84,975 bytes) shows mostly generation 0 collections.</span></span>
* <span data-ttu-id="77abd-271">Over LOH 请求生成常量第2代回收。</span><span class="sxs-lookup"><span data-stu-id="77abd-271">The over LOH requests generate constant generation 2 collections.</span></span> <span data-ttu-id="77abd-272">第2代回收成本高昂。</span><span class="sxs-lookup"><span data-stu-id="77abd-272">Generation 2 collections are expensive.</span></span> <span data-ttu-id="77abd-273">需要更多 CPU，吞吐量几乎会下降到50%。</span><span class="sxs-lookup"><span data-stu-id="77abd-273">More CPU is required and throughput drops almost 50%.</span></span>

<span data-ttu-id="77abd-274">临时大型对象尤其有问题，因为它们会导致 gen2 Gc。</span><span class="sxs-lookup"><span data-stu-id="77abd-274">Temporary large objects are particularly problematic because they cause gen2 GCs.</span></span>

<span data-ttu-id="77abd-275">为了获得最佳性能，应最大程度地减少使用的大型对象。</span><span class="sxs-lookup"><span data-stu-id="77abd-275">For maximum performance, large object use should be minimized.</span></span> <span data-ttu-id="77abd-276">如果可能，请拆分大型对象。</span><span class="sxs-lookup"><span data-stu-id="77abd-276">If possible, split up large objects.</span></span> <span data-ttu-id="77abd-277">例如，ASP.NET Core 中的[响应缓存](xref:performance/caching/response)中间件会将缓存项拆分为小于85000个字节的块。</span><span class="sxs-lookup"><span data-stu-id="77abd-277">For example, [Response Caching](xref:performance/caching/response) middleware in ASP.NET Core split the cache entries into blocks less than 85,000 bytes.</span></span>

<span data-ttu-id="77abd-278">以下链接显示了在 LOH 限制下保留对象的 ASP.NET Core 方法：</span><span class="sxs-lookup"><span data-stu-id="77abd-278">The following links show the ASP.NET Core approach to keeping objects under the LOH limit:</span></span>
- [<span data-ttu-id="77abd-279">ResponseCaching/StreamUtilities</span><span class="sxs-lookup"><span data-stu-id="77abd-279">ResponseCaching/Streams/StreamUtilities.cs</span></span>](https://github.com/aspnet/AspNetCore/blob/v3.0.0/src/Middleware/ResponseCaching/src/Streams/StreamUtilities.cs#L16)
- [<span data-ttu-id="77abd-280">ResponseCaching/MemoryResponseCache</span><span class="sxs-lookup"><span data-stu-id="77abd-280">ResponseCaching/MemoryResponseCache.cs</span></span>](https://github.com/aspnet/ResponseCaching/blob/c1cb7576a0b86e32aec990c22df29c780af29ca5/src/Microsoft.AspNetCore.ResponseCaching/Internal/MemoryResponseCache.cs#L55)

<span data-ttu-id="77abd-281">有关详细信息，请参见:</span><span class="sxs-lookup"><span data-stu-id="77abd-281">For more information, see:</span></span>

* [<span data-ttu-id="77abd-282">发现的大型对象堆</span><span class="sxs-lookup"><span data-stu-id="77abd-282">Large Object Heap Uncovered</span></span>](https://devblogs.microsoft.com/dotnet/large-object-heap-uncovered-from-an-old-msdn-article/)
* [<span data-ttu-id="77abd-283">大型对象堆</span><span class="sxs-lookup"><span data-stu-id="77abd-283">Large object heap</span></span>](/dotnet/standard/garbage-collection/large-object-heap)

### <a name="httpclient"></a><span data-ttu-id="77abd-284">HttpClient</span><span class="sxs-lookup"><span data-stu-id="77abd-284">HttpClient</span></span>

<span data-ttu-id="77abd-285">使用 <xref:System.Net.Http.HttpClient> 错误可能会导致资源泄漏。</span><span class="sxs-lookup"><span data-stu-id="77abd-285">Incorrectly using <xref:System.Net.Http.HttpClient> can result in a resource leak.</span></span> <span data-ttu-id="77abd-286">系统资源，如数据库连接、套接字、文件句柄等：</span><span class="sxs-lookup"><span data-stu-id="77abd-286">System resources, such as database connections, sockets, file handles, etc.:</span></span>

* <span data-ttu-id="77abd-287">比内存更稀有。</span><span class="sxs-lookup"><span data-stu-id="77abd-287">Are more scarce than memory.</span></span>
* <span data-ttu-id="77abd-288">泄漏内存时，问题更多。</span><span class="sxs-lookup"><span data-stu-id="77abd-288">Are more problematic when leaked than memory.</span></span>

<span data-ttu-id="77abd-289">有经验的 .NET 开发人员知道在实现 <xref:System.IDisposable>的对象上调用 <xref:System.IDisposable.Dispose*>。</span><span class="sxs-lookup"><span data-stu-id="77abd-289">Experienced .NET developers know to call <xref:System.IDisposable.Dispose*> on objects that implement <xref:System.IDisposable>.</span></span> <span data-ttu-id="77abd-290">不释放实现 `IDisposable` 的对象通常会导致内存泄漏或泄漏系统资源。</span><span class="sxs-lookup"><span data-stu-id="77abd-290">Not disposing objects that implement `IDisposable` typically results in leaked memory or leaked system resources.</span></span>

<span data-ttu-id="77abd-291">`HttpClient` 实现 `IDisposable`，但**不**应在每次调用时都将其释放。</span><span class="sxs-lookup"><span data-stu-id="77abd-291">`HttpClient` implements `IDisposable`, but should **not** be disposed on every invocation.</span></span> <span data-ttu-id="77abd-292">相反，应重用 `HttpClient`。</span><span class="sxs-lookup"><span data-stu-id="77abd-292">Rather, `HttpClient` should be reused.</span></span>

<span data-ttu-id="77abd-293">以下终结点针对每个请求创建并释放新的 `HttpClient` 实例：</span><span class="sxs-lookup"><span data-stu-id="77abd-293">The following endpoint creates and disposes a new  `HttpClient` instance on every request:</span></span>

```csharp
[HttpGet("httpclient1")]
public async Task<int> GetHttpClient1(string url)
{
    using (var httpClient = new HttpClient())
    {
        var result = await httpClient.GetAsync(url);
        return (int)result.StatusCode;
    }
}
```

<span data-ttu-id="77abd-294">在 "负载" 下，将记录以下错误消息：</span><span class="sxs-lookup"><span data-stu-id="77abd-294">Under load, the following error messages are logged:</span></span>

```
fail: Microsoft.AspNetCore.Server.Kestrel[13]
      Connection id "0HLG70PBE1CR1", Request id "0HLG70PBE1CR1:00000031":
      An unhandled exception was thrown by the application.
System.Net.Http.HttpRequestException: Only one usage of each socket address
    (protocol/network address/port) is normally permitted --->
    System.Net.Sockets.SocketException: Only one usage of each socket address
    (protocol/network address/port) is normally permitted
   at System.Net.Http.ConnectHelper.ConnectAsync(String host, Int32 port,
    CancellationToken cancellationToken)
```

<span data-ttu-id="77abd-295">即使 `HttpClient` 实例被释放，操作系统也需要一些时间来释放实际网络连接。</span><span class="sxs-lookup"><span data-stu-id="77abd-295">Even though the `HttpClient` instances are disposed, the actual network connection takes some time to be released by the operating system.</span></span> <span data-ttu-id="77abd-296">通过持续创建新的连接，会发生_端口耗尽_。</span><span class="sxs-lookup"><span data-stu-id="77abd-296">By continuously creating new connections, _ports exhaustion_ occurs.</span></span> <span data-ttu-id="77abd-297">每个客户端连接都需要自己的客户端端口。</span><span class="sxs-lookup"><span data-stu-id="77abd-297">Each client connection requires its own client port.</span></span>

<span data-ttu-id="77abd-298">防止端口耗尽的一种方法是重复使用同一个 `HttpClient` 实例：</span><span class="sxs-lookup"><span data-stu-id="77abd-298">One way to prevent port exhaustion is to reuse the same `HttpClient` instance:</span></span>

```csharp
private static readonly HttpClient _httpClient = new HttpClient();

[HttpGet("httpclient2")]
public async Task<int> GetHttpClient2(string url)
{
    var result = await _httpClient.GetAsync(url);
    return (int)result.StatusCode;
}
```

<span data-ttu-id="77abd-299">当应用程序停止时，将释放 `HttpClient` 的实例。</span><span class="sxs-lookup"><span data-stu-id="77abd-299">The `HttpClient` instance is released when the app stops.</span></span> <span data-ttu-id="77abd-300">此示例说明，每次使用后都不应释放每个可释放资源。</span><span class="sxs-lookup"><span data-stu-id="77abd-300">This example shows that not every disposable resource should be disposed after each use.</span></span>

<span data-ttu-id="77abd-301">请参阅以下内容，了解更好的方法来处理 `HttpClient` 实例的生存期：</span><span class="sxs-lookup"><span data-stu-id="77abd-301">See the following for a better way to handle the lifetime of an `HttpClient` instance:</span></span>

* [<span data-ttu-id="77abd-302">HttpClient 和生存期管理</span><span class="sxs-lookup"><span data-stu-id="77abd-302">HttpClient and lifetime management</span></span>](/aspnet/core/fundamentals/http-requests#httpclient-and-lifetime-management)
* [<span data-ttu-id="77abd-303">HTTPClient 工厂博客</span><span class="sxs-lookup"><span data-stu-id="77abd-303">HTTPClient factory blog</span></span>](https://devblogs.microsoft.com/aspnet/asp-net-core-2-1-preview1-introducing-httpclient-factory/)
 
### <a name="object-pooling"></a><span data-ttu-id="77abd-304">对象池</span><span class="sxs-lookup"><span data-stu-id="77abd-304">Object pooling</span></span>

<span data-ttu-id="77abd-305">前面的示例演示了如何将 `HttpClient` 实例设为静态的，并由所有请求重复使用。</span><span class="sxs-lookup"><span data-stu-id="77abd-305">The previous example showed how the `HttpClient` instance can be made static and reused by all requests.</span></span> <span data-ttu-id="77abd-306">重复使用会阻止资源耗尽。</span><span class="sxs-lookup"><span data-stu-id="77abd-306">Reuse prevents running out of resources.</span></span>

<span data-ttu-id="77abd-307">对象池：</span><span class="sxs-lookup"><span data-stu-id="77abd-307">Object pooling:</span></span>

* <span data-ttu-id="77abd-308">使用重复使用模式。</span><span class="sxs-lookup"><span data-stu-id="77abd-308">Uses the reuse pattern.</span></span>
* <span data-ttu-id="77abd-309">适用于创建成本很高的对象。</span><span class="sxs-lookup"><span data-stu-id="77abd-309">Is designed for objects that are expensive to create.</span></span>

<span data-ttu-id="77abd-310">池是预初始化对象的集合，这些对象可以在线程之间保留和释放。</span><span class="sxs-lookup"><span data-stu-id="77abd-310">A pool is a collection of pre-initialized objects that can be reserved and released across threads.</span></span> <span data-ttu-id="77abd-311">池可以定义分配规则，例如限制、预定义大小或增长速率。</span><span class="sxs-lookup"><span data-stu-id="77abd-311">Pools can define allocation rules such as limits, predefined sizes, or growth rate.</span></span>

<span data-ttu-id="77abd-312">NuGet 包[ObjectPool](https://www.nuget.org/packages/Microsoft.Extensions.ObjectPool/)包含有助于管理此类池的类。</span><span class="sxs-lookup"><span data-stu-id="77abd-312">The NuGet package [Microsoft.Extensions.ObjectPool](https://www.nuget.org/packages/Microsoft.Extensions.ObjectPool/) contains classes that help to manage such pools.</span></span>

<span data-ttu-id="77abd-313">以下 API 终结点将实例化一个 `byte` 缓冲区，该缓冲区填充了每个请求的随机数字：</span><span class="sxs-lookup"><span data-stu-id="77abd-313">The following API endpoint instantiates a `byte` buffer that is filled with random numbers on each request:</span></span>

```csharp
        [HttpGet("array/{size}")]
        public byte[] GetArray(int size)
        {
            var random = new Random();
            var array = new byte[size];
            random.NextBytes(array);

            return array;
        }
```

<span data-ttu-id="77abd-314">以下图表显示了如何通过中等负载调用前面的 API：</span><span class="sxs-lookup"><span data-stu-id="77abd-314">The following chart display calling the preceding API with moderate load:</span></span>

![上图](memory/_static/array.png)

<span data-ttu-id="77abd-316">在上图中，第0代回收大约每秒发生一次。</span><span class="sxs-lookup"><span data-stu-id="77abd-316">In the preceding chart, generation 0 collections happen approximately once per second.</span></span>

<span data-ttu-id="77abd-317">可以通过使用[`ArrayPool<T>`](xref:System.Buffers.ArrayPool`1)将 `byte` 缓冲区进行缓冲来优化前面的代码。</span><span class="sxs-lookup"><span data-stu-id="77abd-317">The preceding code can be optimized by pooling the `byte` buffer by using [`ArrayPool<T>`](xref:System.Buffers.ArrayPool`1).</span></span> <span data-ttu-id="77abd-318">静态实例可跨请求重复使用。</span><span class="sxs-lookup"><span data-stu-id="77abd-318">A static instance is reused across requests.</span></span>

<span data-ttu-id="77abd-319">此方法的不同之处在于，将从 API 返回一个共用对象。</span><span class="sxs-lookup"><span data-stu-id="77abd-319">What's different with this approach is that a pooled object is returned from the API.</span></span> <span data-ttu-id="77abd-320">这意味着：</span><span class="sxs-lookup"><span data-stu-id="77abd-320">That means:</span></span>

* <span data-ttu-id="77abd-321">从方法返回后，将立即从控件中排除对象。</span><span class="sxs-lookup"><span data-stu-id="77abd-321">The object is out of your control as soon as you return from the method.</span></span>
* <span data-ttu-id="77abd-322">不能释放对象。</span><span class="sxs-lookup"><span data-stu-id="77abd-322">You can't release the object.</span></span>

<span data-ttu-id="77abd-323">设置对象的释放：</span><span class="sxs-lookup"><span data-stu-id="77abd-323">To set up disposal of the object:</span></span>

* <span data-ttu-id="77abd-324">将池数组封装到可释放对象中。</span><span class="sxs-lookup"><span data-stu-id="77abd-324">Encapsulate the pooled array in a disposable object.</span></span>
* <span data-ttu-id="77abd-325">将此池对象注册为[RegisterForDispose](xref:Microsoft.AspNetCore.Http.HttpResponse.RegisterForDispose*)。</span><span class="sxs-lookup"><span data-stu-id="77abd-325">Register the pooled object with [HttpContext.Response.RegisterForDispose](xref:Microsoft.AspNetCore.Http.HttpResponse.RegisterForDispose*).</span></span>

<span data-ttu-id="77abd-326">`RegisterForDispose` 将负责调用目标对象 `Dispose`，以便仅当 HTTP 请求完成时才会释放该对象。</span><span class="sxs-lookup"><span data-stu-id="77abd-326">`RegisterForDispose` will take care of calling `Dispose`on the target object so that it's only released when the HTTP request is complete.</span></span>

```csharp
private static ArrayPool<byte> _arrayPool = ArrayPool<byte>.Create();

private class PooledArray : IDisposable
{
    public byte[] Array { get; private set; }

    public PooledArray(int size)
    {
        Array = _arrayPool.Rent(size);
    }

    public void Dispose()
    {
        _arrayPool.Return(Array);
    }
}

[HttpGet("pooledarray/{size}")]
public byte[] GetPooledArray(int size)
{
    var pooledArray = new PooledArray(size);

    var random = new Random();
    random.NextBytes(pooledArray.Array);

    HttpContext.Response.RegisterForDispose(pooledArray);

    return pooledArray.Array;
}
```

<span data-ttu-id="77abd-327">应用与非池版本相同的负载会导致以下图表：</span><span class="sxs-lookup"><span data-stu-id="77abd-327">Applying the same load as the non-pooled version results in the following chart:</span></span>

![上图](memory/_static/pooledarray.png)

<span data-ttu-id="77abd-329">主要区别是分配的字节数，因此产生的第0代回收量更少。</span><span class="sxs-lookup"><span data-stu-id="77abd-329">The main difference is allocated bytes, and as a consequence much fewer generation 0 collections.</span></span>

## <a name="additional-resources"></a><span data-ttu-id="77abd-330">其他资源</span><span class="sxs-lookup"><span data-stu-id="77abd-330">Additional resources</span></span>

* [<span data-ttu-id="77abd-331">垃圾回收</span><span class="sxs-lookup"><span data-stu-id="77abd-331">Garbage Collection</span></span>](/dotnet/standard/garbage-collection/)
* [<span data-ttu-id="77abd-332">利用并发可视化工具了解不同的 GC 模式</span><span class="sxs-lookup"><span data-stu-id="77abd-332">Understanding different GC modes with Concurrency Visualizer</span></span>](https://blogs.msdn.microsoft.com/seteplia/2017/01/05/understanding-different-gc-modes-with-concurrency-visualizer/)
* [<span data-ttu-id="77abd-333">发现的大型对象堆</span><span class="sxs-lookup"><span data-stu-id="77abd-333">Large Object Heap Uncovered</span></span>](https://devblogs.microsoft.com/dotnet/large-object-heap-uncovered-from-an-old-msdn-article/)
* [<span data-ttu-id="77abd-334">大型对象堆</span><span class="sxs-lookup"><span data-stu-id="77abd-334">Large object heap</span></span>](/dotnet/standard/garbage-collection/large-object-heap)