# Awesome JCEF [![Awesome](https://awesome.re/badge.svg)](https://awesome.re)

> 一个关于 **JCEF（Java Chromium Embedded Framework）** 的精选资源列表。

[Java Chromium Embedded Framework](https://bitbucket.org/chromiumembedded/java-cef)（JCEF）是 Chromium Embedded Framework (CEF) 的 Java 语言绑定。CEF 是一个开源项目，允许开发者在应用程序中嵌入 Chromium 浏览器。

本项目旨在收集和分享与 JCEF 相关的优秀资源、教程和示例，帮助开发者更好地理解和使用 JCEF。

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