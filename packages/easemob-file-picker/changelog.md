# 1.0.0

首次发布。跨平台文件选择器 UTS 插件，为环信 IM 发文件消息场景设计，已在 HBuilderX 5.07 / 5.26 完成 Android、iOS、HarmonyOS Next 三端真机验证。

- 三端系统文档选择器（SAF / UIDocumentPicker / @ohos.file.picker）：选择文件、查询元数据、复制为本地缓存副本、清理临时文件，零存储权限。
- 统一 API：`chooseFile` / `getFileInfo` / `prepareFile` / `removeTempFile`，三端回调结构与错误码一致；用户取消走 `success` 且 `cancelled: true`。
- 环境要求：HBuilderX / uni-app / uni-app-x >= 5.07；Android 7.0+（minSdk 24）、iOS 16+、HarmonyOS Next API 14+。

平台行为要点：

- iOS：picker 暂存文件在选择回调内即复制到沙箱（系统回收暂存文件后 URI 会失效），`prepareFile` 直接返回该副本；所有终止回调推迟到下一 main tick，可在回调中同步调用其他插件 API；副本注册表上限 100 条，超限自动清理最旧副本。
- Android：自动对可持久化 URI 调用 `takePersistableUriPermission`，避免重启后 URI 失效；选择时的元数据会缓存，URI 失权后 `getFileInfo` 仍可返回元数据。
- HarmonyOS Next：`prepareFile` 采用 fd 级复制，兼容 `file://docs/...` 跨沙箱 URI；并发选择守卫（重复调用返回 9010002）；用户取消按 `BusinessError.code` 判定，不依赖 message 文案。
- 三端统一的 `mimeTypes` 容错：映射表外的具体 MIME 忽略并输出告警日志，不阻断选择；全部无法识别时退化为不限类型。
- 三端 `removeTempFile` 除目录前缀校验外，额外拒绝包含 `..` 的路径，避免越界删除。
