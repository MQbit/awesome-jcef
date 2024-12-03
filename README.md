# Awesome JCEF [![Awesome](https://awesome.re/badge.svg)](https://awesome.re)

> 一个关于 **JCEF（Java Chromium Embedded Framework）** 的精选资源列表。

[Java Chromium Embedded Framework](https://bitbucket.org/chromiumembedded/java-cef)（JCEF）是 Chromium Embedded Framework (CEF) 的 Java 语言绑定。CEF 是一个开源项目，允许开发者在应用程序中嵌入 Chromium 浏览器。

本项目旨在收集和分享与 JCEF 相关的优秀资源、教程和示例，帮助开发者更好地理解和使用 JCEF。

![image](https://github.com/user-attachments/assets/64c6a546-494e-48ef-b0c5-fc47dd75526a)

### Java 与 JCEF 的交互流程与原理解析

JCEF（Java Chromium Embedded Framework）是 Chromium 嵌入式框架（CEF）的 Java 封装。它允许在 Java 应用程序中嵌入一个功能完整的 Chromium 浏览器。以下是根据流程图对 Java 与 JCEF 的交互流程和原理的详细讲解：

---

### 核心组件说明

#### **Java 进程部分**
1. **CefClient**：
   - `CefClient` 是 Java 应用和 JCEF 交互的核心入口。
   - 它负责管理多个浏览器实例（`CefBrowser`）以及与 CEF 渲染进程的通信。

2. **CefBrowser**：
   - 每个 `CefBrowser` 实例对应一个浏览器窗口或标签页。
   - 它是 Java 端与浏览器 DOM 和 JavaScript 交互的主要桥梁。

3. **JNI Bridge**：
   - 使用 Java Native Interface (JNI)，实现 Java 和本地 C++ 代码之间的通信。
   - Java 方法可以通过 JNI 调用底层 C++ 的 CEF 接口，而 C++ 的 CEF 回调也可以触发 Java 方法。

#### **Chromium Embedded Framework (CEF)**
1. **Client**：
   - `Client` 是 CEF 的核心组件，它负责与 Java 进程通信。
   - 通过 IPC（进程间通信）与 Chromium Renderer 进程交互。

2. **Browser**：
   - `Browser` 是 CEF 内部浏览器的表示，与 `CefBrowser` 一一对应。
   - 处理浏览器相关的操作，如页面加载、JavaScript 执行等。

#### **Chromium Renderer**
1. **Frame**：
   - `Frame` 是每个网页文档的呈现单位，包含 DOM 和 JavaScript 执行环境。
   - 通过 Chromium IPC 通信，处理渲染和 JavaScript 的执行请求。

2. **DOM 和 JS**：
   - DOM 是页面内容的结构化表示，JS 是页面交互的执行环境。
   - Java 应用程序可以通过 JCEF 操作 DOM 元素或执行 JS 代码。

#### **Chromium GPU**
- GPU 进程负责硬件加速任务，比如页面绘制、视频渲染等。
- 虽然与 JCEF 的交互有限，但对渲染性能至关重要。

---

### Java 与 JCEF 的交互流程

#### 1. **浏览器创建与初始化**
   - Java 应用创建一个 `CefApp` 实例，用于管理整个 JCEF 生命周期。
   - 调用 `CefClient` 创建一个 `CefBrowser` 实例，并加载指定的 URL。

   **代码示例**：
   ```java
   CefApp cefApp = CefApp.getInstance(new CefSettings());
   CefClient client = cefApp.createClient();
   CefBrowser browser = client.createBrowser("https://example.com", false, false);
   ```

#### 2. **页面加载与事件处理**
   - 当 `CefBrowser` 加载页面时，触发 `onLoadStart` 和 `onLoadEnd` 事件。
   - Java 应用可以通过事件处理器监听加载状态，并执行 JavaScript 操作或 DOM 操作。

   **示例：监听页面加载完成**：
   ```java
   client.addLoadHandler(new CefLoadHandlerAdapter() {
       @Override
       public void onLoadEnd(CefBrowser browser, int frameId, int httpStatusCode) {
           if (frameId == 0) { // 主框架加载完成
               System.out.println("Page Loaded: " + browser.getURL());
           }
       }
   });
   ```

#### 3. **JavaScript 执行与结果返回**
   - Java 应用通过 `CefBrowser.executeJavaScript()` 注入 JavaScript 脚本。
   - 脚本可以直接操作 DOM 或调用内嵌的 Java 方法。

   **示例：执行 JavaScript**：
   ```java
   browser.executeJavaScript("document.querySelector('h1').innerText", browser.getURL(), 0);
   ```

#### 4. **Java 和 JS 之间的数据通信**
   - 通过 `CefMessageRouter` 实现双向通信：
     - **从 Java 调用 JS**：直接通过 `executeJavaScript`。
     - **从 JS 调用 Java**：通过 `cefQuery` 方法发送请求到 Java。

   **Java 注册回调方法**：
   ```java
   CefMessageRouter router = CefMessageRouter.create();
   router.addHandler(new CefMessageRouterHandlerAdapter() {
       @Override
       public boolean onQuery(CefBrowser browser, long queryId, String request, boolean persistent, CefQueryCallback callback) {
           System.out.println("Received from JS: " + request);
           callback.success("Data received!");
           return true;
       }
   }, true);
   client.addMessageRouter(router);
   ```

   **JavaScript 调用 Java**：
   ```javascript
   let data = "Hello from JS!";
   window.cefQuery({
       request: data,
       onSuccess: function(response) { console.log(response); },
       onFailure: function(error_code, error_message) { console.error(error_message); }
   });
   ```

#### 5. **DOM 操作与用户模拟**
   - Java 应用可以通过 JS 操作 DOM，例如获取元素、修改值、模拟点击等。
   - 使用 JCEF 的键盘或鼠标事件接口模拟用户操作。

---

### 关键原理解析

1. **JNI Bridge 的作用**
   - JCEF 使用 JNI 在 Java 和 C++ 之间建立桥梁：
     - Java 调用 C++：通过 JNI 接口发送浏览器指令。
     - C++ 回调 Java：通过 JNI 将浏览器事件通知到 Java。

2. **Chromium IPC**
   - `CefClient` 和 `Renderer` 之间通过 IPC 进行通信，处理页面渲染、JavaScript 执行等任务。

3. **线程模型**
   - JCEF 运行在多线程环境：
     - Java 进程处理用户交互和应用逻辑。
     - Renderer 进程负责渲染页面和执行 JavaScript。
     - 通过 IPC 实现线程间的通信。

4. **DOM 操作与 JS 交互**
   - DOM 和 JS 是运行在 Renderer 进程中的，Java 通过 `CefBrowser.executeJavaScript` 将指令发送到 Renderer，由 Renderer 执行并返回结果。

---

### 总结

JCEF 的 Java 与 Chromium 的交互主要通过以下几个关键点实现：
1. **CefClient 和 CefBrowser**：Java 和 CEF 的桥梁，负责管理浏览器实例。
2. **JNI Bridge**：连接 Java 和 C++，实现双向通信。
3. **Chromium IPC**：在 CEF 的 Client 和 Renderer 进程之间进行通信，处理渲染和 JS 执行。
4. **双向通信**：Java 可以通过 `executeJavaScript` 操作网页，JS 可以通过 `cefQuery` 调用 Java 方法。

这套机制使得 JCEF 成为强大的嵌入式浏览器解决方案，非常适合用于 RPA、爬虫和自动化测试等场景。


## 目录

- [特性](#特性)
- [安装](#安装)
- [快速开始](#快速开始)
- [示例代码](#示例代码)
- [资源链接](#资源链接)
- [贡献指南](#贡献指南)
- [许可证](#许可证)

## 特性

- **嵌入式浏览器**：在 Java 应用程序中嵌入 Chromium 浏览器，实现丰富的浏览器功能。
- **JavaScript 交互**：支持 Java 与 JavaScript 之间的双向通信。
- **高度可定制**：通过各种处理器（Handler）和回调函数，自定义浏览器行为。
- **跨平台支持**：支持 Windows、macOS 和 Linux 平台。

## 安装

在您的 Maven 项目中添加以下依赖：

```xml
<dependencies>
    <!-- JCEF Maven Wrapper -->
    <dependency>
        <groupId>me.friwi</groupId>
        <artifactId>jcefmaven</artifactId>
        <version>127.3.1</version>
    </dependency>
    <!-- JSON 处理库 -->
    <dependency>
        <groupId>com.google.code.gson</groupId>
        <artifactId>gson</artifactId>
        <version>2.8.9</version>
    </dependency>
</dependencies>
```

> **注意**：请根据您的需求选择合适的版本，最新版本请参考 [jcefmaven](https://github.com/jcefmaven/jcefmaven) 的官方文档。

## 快速开始

以下是一个基本的示例，展示如何在 Java 应用程序中嵌入浏览器并加载网页：

```java
import org.cef.CefApp;
import org.cef.CefClient;
import org.cef.CefSettings;
import org.cef.browser.CefBrowser;

import javax.swing.*;
import java.awt.*;

public class SimpleBrowser {
    public static void main(String[] args) {
        // 初始化 JCEF
        CefApp.startup();
        CefSettings settings = new CefSettings();
        CefApp cefApp = CefApp.getInstance(settings);
        CefClient client = cefApp.createClient();

        // 创建浏览器实例
        CefBrowser browser = client.createBrowser("https://www.baidu.com", false, false);

        // 创建 Swing 窗口并嵌入浏览器
        JFrame frame = new JFrame("Simple JCEF Browser");
        frame.setSize(1024, 768);
        frame.setDefaultCloseOperation(JFrame.EXIT_ON_CLOSE);
        frame.add(browser.getUIComponent(), BorderLayout.CENTER);
        frame.setVisible(true);
    }
}
```

运行上述代码，将会打开一个包含百度首页的浏览器窗口。

## 示例代码

### 自动化百度搜索并提取结果

以下示例展示了如何使用 JCEF 自动执行百度搜索，并提取搜索结果：

```java
import com.google.gson.Gson;
import com.google.gson.reflect.TypeToken;
import org.cef.CefApp;
import org.cef.CefClient;
import org.cef.CefSettings;
import org.cef.browser.CefBrowser;
import org.cef.handler.CefDisplayHandlerAdapter;
import org.cef.handler.CefLoadHandlerAdapter;

import javax.swing.*;
import java.awt.*;
import java.lang.reflect.Type;
import java.util.List;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.TimeUnit;

public class BaiduSearchExample {
    private CefBrowser browser;
    private CompletableFuture<List<SearchResult>> resultFuture;
    private static final Gson gson = new Gson();

    public List<SearchResult> performBaiduSearch(String keyword) throws Exception {
        resultFuture = new CompletableFuture<>();

        // 初始化 JCEF
        CefApp.startup(new String[]{});
        CefSettings settings = new CefSettings();
        CefApp cefApp = CefApp.getInstance(settings);
        CefClient client = cefApp.createClient();

        // 创建浏览器窗口
        JFrame frame = new JFrame("Baidu Search");
        frame.setSize(1024, 768);
        frame.setDefaultCloseOperation(JFrame.EXIT_ON_CLOSE);

        browser = client.createBrowser("https://www.baidu.com", false, false);
        frame.add(browser.getUIComponent(), BorderLayout.CENTER);
        frame.setVisible(true);

        // 添加加载监听器
        client.addLoadHandler(new CefLoadHandlerAdapter() {
            private boolean isSearchPerformed = false;

            @Override
            public void onLoadingStateChange(CefBrowser browser, boolean isLoading, boolean canGoBack, boolean canGoForward) {
                if (!isLoading && !isSearchPerformed) {
                    // 首页加载完成，执行搜索
                    performSearch(keyword);
                    isSearchPerformed = true;
                }
            }

            @Override
            public void onLoadEnd(CefBrowser cefBrowser, org.cef.browser.CefFrame cefFrame, int httpStatusCode) {
                // 搜索结果页加载完成后提取数据
                if (cefBrowser.getURL().contains("baidu.com/s")) {
                    extractSearchResults();
                }
            }
        });

        // 添加显示处理器，监听控制台消息
        client.addDisplayHandler(new CefDisplayHandlerAdapter() {
            @Override
            public boolean onConsoleMessage(CefBrowser browser, String message, String source, int line) {
                if (message.startsWith("EXTRACTED_RESULTS:")) {
                    String jsonData = message.substring("EXTRACTED_RESULTS:".length());
                    try {
                        List<SearchResult> results = parseSearchResults(jsonData);
                        resultFuture.complete(results);
                    } catch (Exception ex) {
                        resultFuture.completeExceptionally(ex);
                    }
                    return true;
                }
                return false;
            }
        });

        // 等待结果，设置超时
        try {
            List<SearchResult> results = resultFuture.get(30, TimeUnit.SECONDS);

            // 关闭浏览器
            browser.close(true);
            cefApp.dispose();

            return results;
        } catch (Exception e) {
            throw new Exception("搜索超时或发生错误: " + e.getMessage());
        }
    }

    private void performSearch(String keyword) {
        // 执行搜索的 JavaScript
        String searchScript =
                "var input = document.getElementById('kw');" +
                        "var searchBtn = document.getElementById('su');" +
                        "if (input && searchBtn) {" +
                        "   input.value = '" + keyword + "';" +
                        "   searchBtn.click();" +
                        "} else {" +
                        "   console.error('无法找到搜索输入框或按钮');" +
                        "}";

        SwingUtilities.invokeLater(() -> {
            browser.executeJavaScript(searchScript, browser.getURL(), 0);
        });
    }

    private void extractSearchResults() {
        String extractScript =
                "setTimeout(function() {" +
                        "   var results = [];" +
                        "   var searchResults = document.querySelectorAll('div.result, div.result-op');" +
                        "   searchResults.forEach(function(result) {" +
                        "       var titleElement = result.querySelector('h3 a');" +
                        "       if (titleElement) {" +
                        "           results.push({" +
                        "               title: titleElement.innerText.trim()," +
                        "               link: titleElement.href" +
                        "           });" +
                        "       }" +
                        "   });" +
                        "   console.log('EXTRACTED_RESULTS:' + JSON.stringify(results));" +
                        "}, 1000);"; // 延迟 1 秒，确保内容加载完成

        SwingUtilities.invokeLater(() -> {
            browser.executeJavaScript(extractScript, browser.getURL(), 0);
        });
    }

    private List<SearchResult> parseSearchResults(String jsonResult) {
        try {
            if (jsonResult == null || jsonResult.equals("null")) {
                return List.of();
            }

            Type listType = new TypeToken<List<SearchResult>>(){}.getType();
            return gson.fromJson(jsonResult, listType);
        } catch (Exception e) {
            System.err.println("解析搜索结果失败: " + e.getMessage());
            return List.of();
        }
    }

    // 搜索结果内部类
    public static class SearchResult {
        public String title;
        public String link;

        @Override
        public String toString() {
            return "标题: " + title + "\n链接: " + link;
        }
    }

    // 主函数
    public static void main(String[] args) {
        BaiduSearchExample example = new BaiduSearchExample();
        try {
            List<SearchResult> results = example.performBaiduSearch("人工智能");
            for (SearchResult result : results) {
                System.out.println(result);
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

运行上述代码，将自动在百度搜索“人工智能”，并在控制台输出搜索结果的标题和链接。

## 资源链接

- **官方项目和文档**
  - [JCEF 官方仓库](https://bitbucket.org/chromiumembedded/java-cef)：JCEF 的官方代码库。
  - [JCEF Wiki](https://bitbucket.org/chromiumembedded/java-cef/wiki/Home)：JCEF 的使用文档和指南。
  - [CEF 官方网站](https://bitbucket.org/chromiumembedded/cef)：CEF 的官方资源。
  - [jcefmaven](https://github.com/jcefmaven/jcefmaven)：JCEF 的 Maven 包装器，简化了 JCEF 的集成。

- **社区和教程**
  - [JCEF 示例代码](https://github.com/jcefmaven/jcefmaven/tree/master/examples)：各种使用 JCEF 的示例程序。
  - [JCEF 开发者讨论](https://magpcss.org/ceforum/viewforum.php?f=17)：CEF 论坛的 Java 部分。

- **相关项目**
  - [JavaFX WebView](https://openjfx.io)：JavaFX 提供的嵌入式浏览器组件，支持基本的浏览功能。
  - [Electron](https://www.electronjs.org/)：基于 Chromium 和 Node.js，用于构建跨平台桌面应用程序。
  - [jcef-rpa-demo](https://github.com/MQbit/jcef-rpa)

## 贡献指南

非常欢迎大家为本项目贡献力量！如果您有以下内容，欢迎提交 Pull Request：

- 新的示例代码或教程。
- 有用的资源链接或工具。
- 对现有内容的改进或纠正。

请确保您的贡献遵循以下原则：

- **高质量**：提供清晰、易懂、有用的内容。
- **合法合规**：不得包含侵权或违反法律的内容。
- **格式规范**：遵循 Markdown 格式和本项目的结构。

如有任何问题或建议，请提交 Issue。

## 许可证

本项目采用 [MIT License](LICENSE) 许可证。

---

感谢您的关注和支持！希望本项目能对您有所帮助。
