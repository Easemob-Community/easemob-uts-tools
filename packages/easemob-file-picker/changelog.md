## 1.0.1（2026-10-22）

修复 iOS 端失败回调可能闪退的问题，并统一三端错误处理与临时文件清理行为。

- **修复（重要）**：iOS 端 `getFileInfo` / `prepareFile` 失败时可能直接闪退——内部把回调参数做了联合类型断言，编译成 Swift 的强制类型转换后一旦类型不符就会使 App 崩溃。现三端统一改为按具体回调分发，失败一律走 `fail` 回调。
- iOS：副本注册表按 uri 去重并限制 100 条，避免重复副本与临时文件残留。
- 鸿蒙：用户取消改按 `BusinessError.code` 判定，中文环境下不再把正常取消误报为 9010002。
- 三端：`removeTempFile` 在目录前缀校验外增加 `..` 校验，防止越界删除。
- 三端：`mimeTypes` 中无法识别的类型改为忽略并输出告警，不再导致"选不到任何文件"（鸿蒙端不再返回 9010008）；全部无法识别时退化为不限类型。
- 鸿蒙：`mimeType` 兜底为空字符串（此前可能返回 `null`，业务侧取 `.length` 会崩）。
- Android：`getFileInfo` 增加选择时的元数据缓存，URI 读权限丢失后仍可查询。
- 文档：补充失败分支的自测方法，修正大文件复制的耗时说明，错误码表标注保留码。

## 1.0.0（2026-10-21）

首次发布。跨平台文件选择器 UTS 插件，为环信 IM 发文件消息场景设计，已在 HBuilderX 5.07 / 5.26 完成 Android、iOS、HarmonyOS Next 三端真机验证。

- 三端系统文档选择器（SAF / UIDocumentPicker / @ohos.file.picker）：选择文件、查询元数据、复制为本地缓存副本、清理临时文件，零存储权限。
- 统一 API：`chooseFile` / `getFileInfo` / `prepareFile` / `removeTempFile`，三端回调结构与错误码一致；用户取消走 `success` 且 `cancelled: true`。
- 环境要求：HBuilderX / uni-app / uni-app-x >= 5.07；Android 7.0+（minSdk 24）、iOS 16+、HarmonyOS Next API 14+。

平台行为要点：

- iOS：picker 暂存文件在选择回调内即复制到沙箱（系统回收暂存文件后 URI 会失效），`prepareFile` 直接返回该副本；所有终止回调推迟到下一 main tick，可在回调中同步调用其他插件 API。
- Android：自动对可持久化 URI 调用 `takePersistableUriPermission`，避免重启后 URI 失效。
- HarmonyOS Next：`prepareFile` 采用 fd 级复制，兼容 `file://docs/...` 跨沙箱 URI；并发选择守卫（重复调用返回 9010002）；`mimeTypes` 仅支持内置映射表中的具体类型。
