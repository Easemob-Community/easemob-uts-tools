# easemob-uts-tools

环信（Easemob）IM 场景的 UTS 插件集。面向 uni-app / uni-app x 项目，覆盖 Android、iOS、HarmonyOS Next 三端，为 IM 开发中常见的原生能力缺口提供统一、跨端、无权限负担的插件实现。

## 目录结构

```
easemob-uts-tools/
└── packages/                  # 所有插件统一放在 packages 下，一个插件一个目录
    └── easemob-file-picker/   # 跨平台文件选择器
```

## 插件列表

| 插件 | 说明 | 平台 | 文档 |
|---|---|---|---|
| [easemob-file-picker](packages/easemob-file-picker) | 系统文档选择器选文件、元数据查询、URI 复制为本地缓存副本（供 IM SDK 上传）、临时文件清理 | Android / iOS / HarmonyOS | [readme](packages/easemob-file-picker/readme.md) |

### easemob-file-picker 简介

为环信 IM 发文件消息场景设计：

- 三端均走系统文档选择器（SAF / UIDocumentPicker / @ohos.file.picker），**无需申请任何存储权限**，符合各平台隐私合规要求
- 提供 `chooseFile` / `getFileInfo` / `prepareFile` / `removeTempFile` 四个 API
- `prepareFile` 把 provider URI 复制为本地缓存文件，返回真实路径供 IM SDK 上传（HarmonyOS 端采用 fd 级复制，兼容 `file://docs/...` 跨沙箱 URI）
- 统一错误码（9010001-9010008），实现 `IUniError`

## 在项目中使用

**方式一：拷贝接入（使用方）**

把 `packages/` 下对应插件目录整个拷贝到项目的 `uni_modules/` 目录即可，HBuilderX 会自动识别编译。

**方式二：软链接联调（插件开发者）**

插件源码只维护本仓库一份，demo/业务项目通过相对软链接引用：

```bash
# 假设 easemob-uts-tools 与你的项目同级，都在 ~/easemob/ 下
cd your-project/uni_modules
ln -s ../../easemob-uts-tools/packages/easemob-file-picker easemob-file-picker
```

改插件只在本仓库改，业务项目实时生效。git 会提交软链接本身，只要两个仓库保持同级目录结构，克隆后链接依然有效。

## 环境要求

- HBuilderX >= 5.26
- uni-app >= 3.1.0 / uni-app x >= 5.26.0
- Android minSdk 24+ / iOS 16+ / HarmonyOS API 12+

## 新增插件约定

1. 在 `packages/` 下新建插件目录，结构与 uni_modules 插件规范一致（`package.json` + `utssdk/`）
2. 公共接口定义放 `utssdk/interface.uts`，错误码放 `utssdk/unierror.uts`，各端实现放 `utssdk/app-<platform>/`
3. 每个插件自带 readme.md，写明平台要求、API、错误码和完整调用示例
4. 三端行为保持一致：同样的输入给同样的回调结构，平台差异在文档中显式标注

## License

MIT
