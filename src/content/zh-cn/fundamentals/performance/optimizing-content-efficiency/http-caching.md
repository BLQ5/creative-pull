project_path: /web/fundamentals/_project.yaml
book_path: /web/fundamentals/_book.yaml
description:缓存并重用之前提取的资源的能力是性能优化的一个关键方面。

{# wf_updated_on: 2019-02-06 #}
{# wf_published_on: 2013-12-31 #}
{# wf_blink_components: Blink>Network #}

# HTTP 缓存 {: .page-title }

{% include "web/_shared/contributors/ilyagrigorik.html" %}

通过网络提取内容既速度缓慢又开销巨大。 较大的响应需要在客户端与服务器之间进行多次往返通信，这会延迟浏览器获得和处理内容的时间，还会增加访问者的流量费用。 因此，缓存并重复利用之前获取的资源的能力成为性能优化的一个关键方面。


好在每个浏览器都自带了 HTTP 缓存实现功能。 您只需要确保每个服务器响应都提供正确的 HTTP 标头指令，以指示浏览器何时可以缓存响应以及可以缓存多久。

注：如果您在应用中使用 Webview 来获取和显示网页内容，可能需要提供额外的配置标志，以确保 HTTP 缓存得到启用、其大小根据用例进行了合理设置并且缓存将持久保存。 务必查看平台文档并确认您的设置！

<img src="images/http-request.png"  alt="HTTP 请求">

当服务器返回响应时，还会发出一组 HTTP 标头，用于描述响应的内容类型、长度、缓存指令、验证令牌等。 例如，在上图的交互中，服务器返回一个 1024 字节的响应，指示客户端将其缓存最多 120 秒，并提供一个验证令牌（“x234dff”），可在响应过期后用来检查资源是否被修改。


## 通过 ETag 验证缓存的响应

### TL;DR {: .hide-from-toc }
* 服务器使用 ETag HTTP 标头传递验证令牌。
* 验证令牌可实现高效的资源更新检查：资源未发生变化时不会传送任何数据。


假定在首次提取资源 120 秒后，浏览器又对该资源发起了新的请求。 首先，浏览器会检查本地缓存并找到之前的响应。 遗憾的是，该响应现已过期，浏览器无法使用。 此时，浏览器可以直接发出新的请求并获取新的完整响应。 不过，这样做效率较低，因为如果资源未发生变化，那么下载与缓存中已有的完全相同的信息就毫无道理可言！

这正是验证令牌（在 ETag 标头中指定）旨在解决的问题。 服务器生成并返回的随机令牌通常是文件内容的哈希值或某个其他指纹。 客户端不需要了解指纹是如何生成的，只需在下一次请求时将其发送至服务器。 如果指纹仍然相同，则表示资源未发生变化，您就可以跳过下载。

<img src="images/http-cache-control.png"  alt="HTTP Cache-Control 示例">

在上例中，客户端自动在“If-None-Match” HTTP 请求标头内提供 ETag 令牌。 服务器根据当前资源核对令牌。 如果它未发生变化，服务器将返回“304 Not Modified”响应，告知浏览器缓存中的响应未发生变化，可以再延用 120 秒。 请注意，您不必再次下载响应，这节约了时间和带宽。

作为网络开发者，您如何利用高效的重新验证？浏览器会替我们完成所有工作： 它会自动检测之前是否指定了验证令牌，它会将验证令牌追加到发出的请求上，并且它会根据从服务器接收的响应在必要时更新缓存时间戳。 **我们唯一要做的就是确保服务器提供必要的 ETag 令牌。 检查您的服务器文档中有无必要的配置标记。**

注：提示：HTML5 Boilerplate 项目包含所有最流行服务器的<a href='https://github.com/h5bp/server-configs'>配置文件样例</a>，其中为每个配置标志和设置都提供了详细的注解。 在列表中找到您喜爱的服务器，查找合适的设置，然后复制/确认您的服务器配置了推荐的设置。

## Cache-Control

### TL;DR {: .hide-from-toc }
* 每个资源都可通过 Cache-Control HTTP 标头定义其缓存策略
* Cache-Control 指令控制谁在什么条件下可以缓存响应以及可以缓存多久。


从性能优化的角度来说，最佳请求是无需与服务器通信的请求：您可以通过响应的本地副本消除所有网络延迟，以及避免数据传送的流量费用。 为实现此目的，HTTP 规范允许服务器返回 [Cache-Control 指令](http://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html#sec14.9)，这些指令控制浏览器和其他中间缓存如何缓存各个响应以及缓存多久。

注：Cache-Control 标头是在 HTTP/1.1 规范中定义的，取代了之前用来定义响应缓存策略的标头（例如 Expires）。 所有现代浏览器都支持 Cache-Control，因此，使用它就够了。

<img src="images/http-cache-control-highlight.png"  alt="HTTP Cache-Control 示例">

### “no-cache”和“no-store”

“no-cache”表示必须先与服务器确认返回的响应是否发生了变化，然后才能使用该响应来满足后续对同一网址的请求。 因此，如果存在合适的验证令牌 (ETag)，no-cache 会发起往返通信来验证缓存的响应，但如果资源未发生变化，则可避免下载。

相比之下，“no-store”则要简单得多。 它直接禁止浏览器以及所有中间缓存存储任何版本的返回响应，例如，包含个人隐私数据或银行业务数据的响应。 每次用户请求该资产时，都会向服务器发送请求，并下载完整的响应。

### “public”与 “private”

如果响应被标记为“public”，则即使它有关联的 HTTP 身份验证，甚至响应状态代码通常无法缓存，也可以缓存响应。 大多数情况下，“public”不是必需的，因为明确的缓存信息（例如“max-age”）已表示响应是可以缓存的。

相比之下，浏览器可以缓存“private”响应。 不过，这些响应通常只为单个用户缓存，因此不允许任何中间缓存对其进行缓存。 例如，用户的浏览器可以缓存包含用户私人信息的 HTML 网页，但 CDN 却不能缓存。

### “max-age”

指令指定从请求的时间开始，允许提取的响应被重用的最长时间（单位：秒）。 例如，“max-age=60”表示可在接下来的 60 秒缓存和重用响应。

## 定义最佳 Cache-Control 策略

<img src="images/http-cache-decision-tree.png"  alt="缓存决策树">

按照以上决策树为您的应用使用的特定资源或一组资源确定最佳缓存策略。 在理想的情况下，您的目标应该是在客户端上缓存尽可能多的响应，缓存尽可能长的时间，并且为每个响应提供验证令牌，以实现高效的重新验证。

<table class="responsive">
<thead>
  <tr>
    <th colspan="2">Cache-Control 指令和说明</th>
  </tr>
</thead>
<tr>
  <td data-th="cache-control">max-age=86400</td>
  <td data-th="explanation">浏览器以及任何中间缓存均可将响应（如果是“public”响应）缓存长达 1 天（60 秒 x 60 分钟 x 24 小时）。</td>
</tr>
<tr>
  <td data-th="cache-control">private, max-age=600</td>
  <td data-th="explanation">客户端的浏览器只能将响应缓存最长 10 分钟（60 秒 x 10 分钟）。</td>
</tr>
<tr>
  <td data-th="cache-control">no-store</td>
  <td data-th="explanation">不允许缓存响应，每次请求都必须完整获取。</td>
</tr>
</table>

根据 HTTP Archive，在排名最高的 300,000 个网站（按照 Alexa 排名）中，[所有下载的响应中几乎有半数](http://httparchive.org/trends.php#maxage0)可由浏览器缓存，这可以大量减少重复的网页浏览和访问。 当然，这并不意味着您的特定应用有 50% 的资源可以缓存。 一些网站的资源 90% 以上都可以缓存，而其他网站可能有许多私密或时效要求高的数据根本无法缓存。

**请审核您的网页，确定哪些资源可以缓存，并确保其返回正确的 Cache-Control 和 ETag 标头。**

## 废弃和更新缓存的响应

### TL;DR {: .hide-from-toc }
* 在资源“过期”之前，将一直使用本地缓存的响应。
* 您可以通过在网址中嵌入文件内容指纹，强制客户端更新到新版本的响应。
* 为获得最佳性能，每个应用都需要定义自己的缓存层次结构。


浏览器发出的所有 HTTP 请求会首先路由到浏览器缓存，以确认是否缓存了可用于满足请求的有效响应。 如果有匹配的响应，则从缓存中读取响应，这样就避免了网络延迟和传送产生的流量费用。

**不过，如果您想更新或废弃缓存的响应，该怎么办？**例如，假定您已告诉访问者将某个 CSS 样式表缓存长达 24 小时 (max-age=86400)，但设计人员刚刚提交了一个您希望所有用户都能使用的更新。 您该如何通知拥有现在“已过时”的 CSS 缓存副本的所有访问者更新其缓存？在不更改资源网址的情况下，您做不到。

浏览器缓存响应后，缓存的版本将一直使用到过期（由 max-age 或 expires 决定），或一直使用到由于某种其他原因从缓存中删除，例如用户清除了浏览器缓存。 因此，构建网页时，不同的用户可能最终使用的是文件的不同版本；刚提取了资源的用户将使用新版本的响应，而缓存了早期（但仍有效）副本的用户将使用旧版本的响应。

**如何才能鱼和熊掌兼得：客户端缓存和快速更新？**您可以在资源内容发生变化时更改其网址，强制用户下载新响应。 通常情况下，可以通过在文件名中嵌入文件的指纹或版本号来实现&mdash;例如 style.**x234dff**.css。

<img src="images/http-cache-hierarchy.png"  alt="缓存层次结构">

因为能够定义每个资源的缓存策略，所以您可以定义“缓存层次结构”，这样不但可以控制每个响应的缓存时间，还可以控制访问者看到新版本的速度。 为了进行说明，我们一起分析一下上面的示例：

* HTML 被标记为“no-cache”，这意味着浏览器在每次请求时都始终会重新验证文档，并在内容变化时提取最新版本。 此外，在 HTML 标记内，您在 CSS 和 JavaScript 资产的网址中嵌入指纹：如果这些文件的内容发生变化，网页的 HTML 也会随之改变，并会下载 HTML 响应的新副本。
* 允许浏览器和中间缓存（例如 CDN）缓存 CSS，并将 CSS 设置为 1 年后到期。 请注意，您可以放心地使用 1 年的“远期过期”，因为您在文件名中嵌入了文件的指纹：CSS 更新时网址也会随之变化。
* JavaScript 同样设置为 1 年后到期，但标记为 private，这或许是因为它包含的某些用户私人数据是 CDN 不应缓存的。
* 图像缓存时不包含版本或唯一指纹，并设置为 1 天后到期。

您可以组合使用 ETag、Cache-Control 和唯一网址来实现一举多得：较长的过期时间、控制可以缓存响应的位置以及随需更新。

## 缓存检查清单

不存在什么最佳缓存策略。 您需要根据通信模式、提供的数据类型以及应用特定的数据更新要求，为每个资源定义和配置合适的设置，以及整体的“缓存层次结构”。

在制定缓存策略时，您需要牢记下面这些技巧和方法：

* **使用一致的网址：**如果您在不同的网址上提供相同的内容，将会多次提取和存储这些内容。 提示：请注意，[网址区分大小写](http://www.w3.org/TR/WD-html40-970708/htmlweb.html)。
* **确保服务器提供验证令牌 (ETag)：**有了验证令牌，当服务器上的资源未发生变化时，就不需要传送相同的字节。
* **确定中间缓存可以缓存哪些资源：**对所有用户的响应完全相同的资源非常适合由 CDN 以及其他中间缓存进行缓存。
* **为每个资源确定最佳缓存周期：**不同的资源可能有不同的更新要求。 为每个资源审核并确定合适的 max-age。
* **确定最适合您的网站的缓存层次结构：**您可以通过为 HTML 文档组合使用包含内容指纹的资源网址和短时间或 no-cache 周期，来控制客户端获取更新的速度。
* **最大限度减少搅动：**某些资源的更新比其他资源频繁。 如果资源的特定部分（例如 JavaScript 函数或 CSS 样式集）会经常更新，可以考虑将其代码作为单独的文件提供。 这样一来，每次提取更新时，其余内容（例如变化不是很频繁的内容库代码）可以从缓存提取，从而最大限度减少下载的内容大小。

## 反馈 {: .hide-from-toc }

{% include "web/_shared/helpful.html" %}

<div class="clearfix"></div>