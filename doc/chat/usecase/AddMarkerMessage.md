# 添加标记消息

## 概述

添加标记消息功能是My Assistant插件中的一个特性，它允许用户在对话中插入可视化的分隔点。这个看似简单的功能实际上为用户提供了强大的对话管理工具，标记消息允许用户在长对话中创建清晰的分隔点，这对于组织和管理长期对话特别有用，可以将对话分成不同的主题或阶段。

## 核心实现

```javascript
class AddMarkerAgent {
    async executeTask(task, thread) {
        if (task.name === "AddMarker") {
            const { threadRepository, postMessage } = task.host_utils;
            
            // 核心代码开始
            // 1. 添加标记到thread
            const markerId = threadRepository.addMarker(thread);
            
            // 2. 通知前端更新UI
            postMessage({
                type: 'markerAdded',
                thread: thread,
                markerId
            });
            // 核心代码结束


            return new Response("Marker added successfully.");
        }
    }
}
```

核心机制解释：

1. 添加标记：
   ```javascript
   const markerId = threadRepository.addMarker(thread);
   ```
   这行代码是添加标记的核心。它调用 `threadRepository` 的 `addMarker` 方法，将标记添加到当前线程中。这个方法会在线程的消息列表中插入一个特殊的标记消息。

2. 通知前端：
   ```javascript
   postMessage({
       type: 'markerAdded',
       thread: thread,
       markerId
   });
   ```
   这段代码通知前端UI标记已被添加。前端可以根据这个消息更新UI，例如在聊天界面中显示一条分隔线。

这两步构成了添加标记功能的核心机制：
- 在后端数据中添加标记
- 通知前端更新UI以反映这个变化

