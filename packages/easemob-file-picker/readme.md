# easemob-file-picker

跨平台文件选择器 UTS 插件，为环信（Easemob）IM 发文件消息场景设计。

**解决的痛点**：系统文件选择器返回的是 provider URI（`content://` / `file://docs/...`），而环信 IM SDK 发送文件消息需要**真实本地路径**。本插件把「选文件 → 读元数据 → 复制为本地副本 → 用完清理」整条链路封装成 4 个 API，三端行为完全一致。

- 插件类型：UTS 插件（uni_modules）
- 支持平台：Android / iOS / HarmonyOS Next（uni-app 与 uni-app x）
- 仓库位置：[easemob-uts-tools](https://github.com/Easemob-Community/easemob-uts-tools) → `packages/easemob-file-picker`

## 特性

- **零存储权限**：三端均走系统文档选择器（SAF / UIDocumentPicker / @ohos.file.picker），不申请任何存储权限，满足各平台隐私合规与上架审核要求
- **三端统一**：同一套 TypeScript 调用代码，回调结构、错误码完全一致
- **安全复制**：`prepareFile` 生成带随机名的缓存副本；HarmonyOS 端采用 fd 级复制，兼容 `file://docs/...` 跨沙箱 URI
- **防误删**：`removeTempFile` 只删插件命名空间内的临时文件，越界路径返回 `removed: false` 而非报错

## 平台要求

| 平台 | 最低版本 | 实现方式 | 存储权限 |
|---|---|---|---|
| Android | 7.0（minSdk 24） | SAF（`ACTION_OPEN_DOCUMENT`） | 无需 |
| iOS | 16 | `UIDocumentPickerViewController` + `UTType` | 无需 |
| HarmonyOS Next | API 12+ | `@ohos.file.picker` | 无需 |

开发环境：HBuilderX >= 4.71，uni-app >= 4.71.0 / uni-app x >= 4.71.0。

## 快速开始

把本目录整个拷贝到项目的 `uni_modules/` 下即可，无需原生配置。

**完整流程：选文件 → 复制到本地 → 环信发送 → 清理**

```ts
import { chooseFile, prepareFile, removeTempFile } from '@/uni_modules/easemob-file-picker'

chooseFile({
  mimeTypes: ['image/*', 'application/pdf'],  // 不传或空数组 = 不限类型
  multiple: false,
  success: (result) => {
    // 注意：用户取消是正常结果，走 success 且 cancelled = true
    if (result.cancelled || result.files.length == 0) return

    prepareFile({
      uri: result.files[0].uri,
      success: (file) => {
        // file.path 为本地缓存路径，可直接传给环信 SDK 发送文件消息
        // 例：WebIM.message.create({ type: 'file', url: file.path, filename: file.name, ... })

        // 发送完成后清理临时文件
        removeTempFile({
          path: file.path!,
          success: (res) => console.log('临时文件清理:', res.removed)
        })
      },
      fail: (err) => console.error('文件复制失败', err.errCode, err.errMsg)
    })
  },
  fail: (err) => console.error('选择文件失败', err.errCode, err.errMsg)
})
```

**只查文件信息（不复制）**

```ts
import { getFileInfo } from '@/uni_modules/easemob-file-picker'

getFileInfo({
  uri: 'content://media/external/file/12345',
  success: (info) => console.log(info.name, info.size, info.mimeType)
})
```

## API 参考

| 方法 | 说明 |
|---|---|
| `chooseFile(options)` | 打开系统文件选择器，返回选中文件的元数据 |
| `getFileInfo(options)` | 根据 URI 查询文件元数据（不复制文件） |
| `prepareFile(options)` | 把 provider URI 复制到应用缓存目录，返回真实本地路径 |
| `removeTempFile(options)` | 删除 `prepareFile` 创建的临时文件 |

### chooseFile(options)

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| mimeTypes | string[] | 否 | 限制可选 MIME 类型，如 `['application/pdf']`、`['image/*']`；不传或空数组表示不限 |
| multiple | boolean | 否 | 是否多选，默认 false（多选上限 50） |
| success | (res: ChooseFileResult) => void | 否 | 成功回调（**用户取消也走 success**） |
| fail | (err: FilePickerError) => void | 否 | 失败回调 |
| complete | (res: any) => void | 否 | 完成回调（成功失败都触发） |

`ChooseFileResult`：`{ files: FileInfo[], cancelled: boolean }`，取消时 `files` 为空数组。

### getFileInfo(options) / prepareFile(options)

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| uri | string | 是 | `chooseFile` 返回的 `uri`，或 content:// / file:// 路径 |
| success | (res: FileInfo) => void | 否 | `prepareFile` 的返回中 `path` 有值且 `isTemporary: true` |
| fail / complete | - | 否 | 同上 |

### removeTempFile(options)

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| path | string | 是 | `prepareFile` 返回的 `path` |
| success | (res: { removed: boolean }) => void | 否 | 路径不在插件命名空间内时 `removed: false`，不报错 |

## FileInfo 结构

| 字段 | 类型 | 说明 |
|---|---|---|
| uri | string | 原始 URI |
| name | string | 文件名 |
| size | number | 文件大小（字节） |
| mimeType | string | MIME 类型，查询失败时为空字符串 |
| extension | string | 小写扩展名（不含点），无扩展名时为空字符串 |
| path | string \| null | 本地路径，仅 `prepareFile` 返回时有值 |
| isTemporary | boolean | 是否为插件创建的临时副本 |

## 错误码

`fail` 回调返回 `FilePickerError`（实现 `IUniError`），`errSubject` 固定为 `easemob-file-picker`：

| errCode | 说明 | 典型场景 |
|---|---|---|
| 9010001 | 文件选择失败 | 系统选择器返回异常 |
| 9010002 | 无法打开文件选择器 | Activity 不可用、已有进行中的选择操作 |
| 9010003 | 文件访问权限被拒 | 预留（系统 picker 模式下不会触发） |
| 9010004 | URI 无效或不可用 | `getFileInfo` / `prepareFile` 的 URI 已失效 |
| 9010005 | 文件元数据不可用 | 选中文件全部无法读取元数据 |
| 9010006 | 无法读取或复制文件 | `prepareFile` 复制失败、IO 异常 |
| 9010007 | 存储空间不足 | 预留 |
| 9010008 | 当前平台不支持该操作 | Web / 小程序等平台调用 |

## 平台行为说明

**Android**
- 使用 SAF，targetSdk 30+ 无需 `READ_EXTERNAL_STORAGE` 等权限
- 单 MIME 类型走 `Intent.setType`，多类型走 `EXTRA_MIME_TYPES`；`image/*` 等通配符由系统选择器解释
- 自动对可持久化 URI 调用 `takePersistableUriPermission`，避免重启后 URI 失效

**iOS**
- MIME 通过 `UTType(mimeType:)` 映射；`image/*` 等通配映射到预置 UTType；全部未识别时回退为不限类型
- 通过安全作用域资源（Security-Scoped Resource）访问选中文件，读完即释放
- 同时只能进行一次选择操作，重复调用返回 9010002

**HarmonyOS Next**
- 使用 `@ohos.file.picker` 的 DocumentViewPicker
- picker 授予的临时读权限挂在 fd 上，`prepareFile` 内部采用 fd 级 `copyFileSync`，兼容 `file://docs/storage/...` 等跨应用公共目录 URI
- 临时文件以 `easemob-file-` 前缀存放于应用 cacheDir

## 注意事项

1. **用户取消不是错误**：取消走 `success` 且 `cancelled: true`，不要用 `fail` 判断取消
2. **临时文件生命周期**：`prepareFile` 的副本由调用方通过 `removeTempFile` 清理；系统可能随时清缓存，不要长期持有 `path`
3. **大文件复制**：`prepareFile` 为同步复制，大文件耗时较长，建议 UI 加 loading
4. **Web / 小程序不支持**：仅支持 App 三端，其他平台调用返回 9010008

## 目录结构

```
easemob-file-picker/
├── package.json            # 插件清单（平台支持声明）
├── changelog.md
├── readme.md
└── utssdk/
    ├── interface.uts       # 公共类型定义（FileInfo / Options / Result / 错误码）
    ├── unierror.uts        # FilePickerError 实现
    ├── app-android/        # Android 实现（Kotlin）
    ├── app-ios/            # iOS 实现（Swift，deploymentTarget 16）
    └── app-harmony/        # HarmonyOS 实现（ArkTS）
```

## 相关链接

- [UTS 语法](https://uniapp.dcloud.net.cn/tutorial/syntax-uts.html)
- [UTS API 插件开发](https://uniapp.dcloud.net.cn/plugin/uts-plugin.html)
- [环信 IM 文档](https://doc.easemob.com)
