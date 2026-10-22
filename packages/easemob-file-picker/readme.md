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
- **防误删**：`removeTempFile` 只删应用缓存 / 临时目录内的文件，且拒绝包含 `..` 的路径，越界路径返回 `removed: false` 而非报错

## 平台要求

| 平台 | 最低版本 | 实现方式 | 存储权限 |
|---|---|---|---|
| Android | 7.0（minSdk 24） | SAF（`ACTION_OPEN_DOCUMENT`） | 无需 |
| iOS | 16 | `UIDocumentPickerViewController` + `UTType` | 无需 |
| HarmonyOS Next | API 14+ | `@ohos.file.picker` | 无需 |

开发环境：HBuilderX >= 5.07（uni-app / uni-app x 同版本线）。已在 HBuilderX 5.07 与 5.26 完成 Android / iOS / HarmonyOS Next 三端真机验证。

## 快速开始

把本目录整个拷贝到项目的 `uni_modules/` 下即可，无需原生配置。

**完整流程：选文件 → 复制到本地 → 环信发送 → 清理**

```ts
import { chooseFile, prepareFile, removeTempFile } from '@/uni_modules/easemob-file-picker'

chooseFile({
  mimeTypes: ['image/*', 'application/pdf'],  // 不传或空数组 = 不限类型（鸿蒙端限制见「平台行为说明」）
  multiple: false,
  success: (result) => {
    // 注意：用户取消是正常结果，走 success 且 cancelled = true
    if (result.cancelled || result.files.length == 0) return

    prepareFile({
      uri: result.files[0].uri,
      success: (file) => {
        // file.path 为本地缓存路径，可直接传给环信 SDK 发送文件消息
        // （以下为示意代码，字段以环信 SDK 文档为准）
        // WebIM.message.create({ type: 'file', url: file.path, filename: file.name, ... })

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
| mimeTypes | string[] | 否 | 限制可选 MIME 类型，如 `['application/pdf']`、`['image/*']`；不传或空数组表示不限。无法识别的具体 MIME 会被忽略（不报错），全部无法识别时同样退化为不限；鸿蒙端支持范围见「平台行为说明」 |
| multiple | boolean | 否 | 是否多选，默认 false。仅 HarmonyOS 端限制最多 50 个；Android / iOS 不设插件层上限，由系统选择器决定 |
| success | (res: ChooseFileResult) => void | 否 | 成功回调（**用户取消也走 success**） |
| fail | (err: FilePickerError) => void | 否 | 失败回调 |
| complete | (res: any) => void | 否 | 完成回调（成功失败都触发，以下各 API 同） |

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
| success | (res: { removed: boolean }) => void | 否 | 路径不在应用缓存 / 临时目录内（或含 `..`）时 `removed: false`，不报错 |
| fail / complete | - | 否 | 同上 |

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
| 9010001 | 文件选择失败 | 保留码，当前版本不会主动返回 |
| 9010002 | 无法打开文件选择器 | Activity / 页面不可用、已有进行中的选择操作 |
| 9010003 | 文件访问权限被拒 | 保留码，当前版本不会主动返回 |
| 9010004 | URI 无效或不可用 | `getFileInfo` 的 URI 已失效；iOS 端 `prepareFile` 的 URI 已失效 |
| 9010005 | 文件元数据不可用 | 选中文件全部无法读取元数据 |
| 9010006 | 无法读取或复制文件 | `prepareFile` 复制失败、IO 异常；Android / 鸿蒙端 `prepareFile` 的 URI 失效也返回此码 |
| 9010007 | 存储空间不足 | 保留码，当前版本不会主动返回 |
| 9010008 | 当前平台不支持该操作 | 保留码（预留给后续 Web 等平台实现），当前三端不会主动返回 |

> 9010001 / 9010003 / 9010007 / 9010008 为保留码：业务侧无需为它们编写分支，判断失败原因只需关注 9010002 / 9010004 / 9010005 / 9010006。


## 平台行为说明

**Android**
- 使用 SAF，targetSdk 30+ 无需 `READ_EXTERNAL_STORAGE` 等权限
- 单 MIME 类型走 `Intent.setType`，多类型走 `EXTRA_MIME_TYPES`；`image/*` 等通配符由系统选择器解释
- 自动对可持久化 URI 调用 `takePersistableUriPermission`，避免重启后 URI 失效
- 选择时的文件元数据会缓存（最多 20 条），URI 读权限丢失后 `getFileInfo` 仍能返回元数据；`prepareFile` 需要真实读取，URI 失权后会返回 9010006，此时需重新选择

**iOS**
- MIME 通过 `UTType(mimeType:)` 映射；`image/*`、`video/*`、`audio/*`、`text/*` 通配映射到预置 UTType，无法识别的具体 MIME 忽略并输出告警日志（不会阻断选择）
- 通过安全作用域资源（Security-Scoped Resource）访问选中文件
- **选择回调内即完成沙箱复制**：picker 返回的是系统 File Provider Storage 暂存文件，选择结束且访问权释放后系统会回收，延迟重放 URI 不可靠。因此 iOS 端在选中时立即复制到沙箱 tmp 并内部注册，`prepareFile` 直接返回该副本（对外契约不变：`chooseFile` 仍只返回元数据）。该复制走 APFS 写时复制（clonefile），同卷内耗时与文件大小无关（700MB 实测瞬时完成），也不额外占用磁盘空间
- **`success` 回调推迟到下一 main tick 触发**：可在 `success` 中直接同步调用 `prepareFile` 等其他插件 API，无需自行 setTimeout
- 副本注册表上限 100 条（按 uri 去重），超限自动删除最旧副本，避免只选不发场景下 tmp 文件堆积；单次选择超过 100 个文件时，最早的文件需重新选择
- 同时只能进行一次选择操作，重复调用返回 9010002

**HarmonyOS Next**
- 使用 `@ohos.file.picker` 的 DocumentViewPicker
- picker 授予的临时读权限挂在 fd 上，`prepareFile` 内部采用 fd 级 `copyFileSync`，兼容 `file://docs/storage/...` 等跨应用公共目录 URI
- 临时文件以 `easemob-file-` 前缀存放于应用 cacheDir
- 同时只能进行一次选择操作，重复调用返回 9010002；用户取消按 `BusinessError.code` 判定（不依赖 message 文案，中文环境同样正确）
- **mimeTypes 通过内置映射表（pdf / txt / zip / json / office 文档 / 常见图片音视频等 19 种）转换为后缀过滤**：映射表外的具体 MIME 会被忽略并输出告警日志，不会报错
- mimeTypes 中任一 `*/*` 或 `xxx/*` 通配符都会使整组过滤失效，展示全部文件（通配符无法映射为后缀过滤）


## 注意事项

1. **用户取消不是错误**：取消走 `success` 且 `cancelled: true`，不要用 `fail` 判断取消
2. **临时文件生命周期**：`prepareFile` 的副本由调用方通过 `removeTempFile` 清理（存放位置：Android 应用 cacheDir、iOS NSTemporaryDirectory、鸿蒙应用 cacheDir）；系统可能随时清缓存，不要长期持有 `path`
3. **大文件复制**：Android / 鸿蒙为真实字节复制，耗时与体积线性相关，建议 UI 加 loading；iOS 端副本走 APFS 写时复制，同卷内耗时与文件大小无关（700MB 实测瞬时完成），无需额外 loading。若所选文件来自需要联网下载的云盘提供方，耗时发生在系统选择器阶段，不在插件内
4. **失败分支建议自测一次**：`prepareFile({ uri: 'file:///not-exist/a.pdf', fail: ... })` 应回调 9010004（iOS）/ 9010006（Android、鸿蒙），不应无响应或崩溃
5. **Web / 小程序不支持**：仅支持 App 三端；这些平台未实现该插件，无法调用
6. **鸿蒙 MIME 限制**：`mimeTypes` 尽量使用内置映射表中的类型或通配符，映射表外的类型会被忽略（详见「平台行为说明」）

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
