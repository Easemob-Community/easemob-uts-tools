# easemob-file-picker

跨平台文件选择器 UTS 插件，为环信（Easemob）IM 场景设计：调用系统文档选择器选文件、查询元数据、把 provider URI 拷贝为本地缓存副本（供 IM SDK 上传使用）、清理临时文件。

- 插件类型：UTS 插件（uni_modules）
- 支持平台：Android / iOS / HarmonyOS Next（uni-app 与 uni-app x）
- 公共接口定义：`utssdk/interface.uts`，错误码定义：`utssdk/unierror.uts`

## 平台要求

| 平台 | 最低版本 | 实现方式 | 存储权限 |
|---|---|---|---|
| Android | Android 7.0（minSdk 24） | SAF（`ACTION_OPEN_DOCUMENT`） | 无需任何存储权限 |
| iOS | iOS 16 | `UIDocumentPickerViewController` + `UTType` | 无需权限 |
| HarmonyOS Next | API 12+ | `@ohos.file.picker` | 无需权限 |

要求 HBuilderX >= 5.26。

三个平台均使用系统文档选择器，不申请广泛存储权限，符合各平台隐私合规要求。

## API 一览

| 方法 | 说明 |
|---|---|
| `chooseFile(options)` | 打开系统文件选择器，返回选中文件的元数据 |
| `getFileInfo(options)` | 根据 URI 查询文件元数据（不复制文件） |
| `prepareFile(options)` | 把 provider URI 复制到应用缓存目录，返回真实本地路径 |
| `removeTempFile(options)` | 删除 `prepareFile` 创建的临时文件 |

### chooseFile(options)

打开系统文件选择器。

**ChooseFileOptions**

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| mimeTypes | string[] | 否 | 限制可选的 MIME 类型，如 `['application/pdf']`、`['image/*']`。不传或空数组表示不限类型 |
| multiple | boolean | 否 | 是否多选，默认 false |
| success | (res: ChooseFileResult) => void | 否 | 成功回调（用户取消也会走 success） |
| fail | (err: FilePickerError) => void | 否 | 失败回调 |
| complete | (res: any) => void | 否 | 完成回调（成功失败都会触发） |

**ChooseFileResult**

| 字段 | 类型 | 说明 |
|---|---|---|
| files | FileInfo[] | 选中文件的元数据列表 |
| cancelled | boolean | 用户是否取消选择。取消时 `files` 为空数组，属于正常结果而非错误 |

### getFileInfo(options)

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| uri | string | 是 | 文件 URI（`chooseFile` 返回的 `uri`，或 content:// / file:// 路径） |
| success / fail / complete | - | 否 | 同 chooseFile，success 回调参数为 `FileInfo` |

### prepareFile(options)

将 URI 指向的文件复制到应用缓存目录，返回带真实 `path` 的 `FileInfo`（`isTemporary: true`）。环信 IM SDK 发送文件/图片消息时需要本地路径，应先调用此方法。

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| uri | string | 是 | 文件 URI |
| success / fail / complete | - | 否 | success 回调参数为 `FileInfo`，其中 `path` 为本地缓存路径 |

### removeTempFile(options)

删除 `prepareFile` 创建的临时文件。出于安全考虑，**仅允许删除插件命名空间内的文件**（Android 校验 cacheDir canonicalPath 前缀，iOS 校验临时目录前缀，HarmonyOS 校验 `easemob-file-` 前缀），路径不在范围内时返回 `removed: false` 而不是报错。

| 参数 | 类型 | 必填 | 说明 |
|---|---|---|---|
| path | string | 是 | `prepareFile` 返回的 `path` |
| success / fail / complete | - | 否 | success 回调参数为 `RemoveTempFileResult`：`{ removed: boolean }` |

## FileInfo 结构

| 字段 | 类型 | 说明 |
|---|---|---|
| uri | string | 原始 URI |
| name | string | 文件名 |
| size | number | 文件大小（字节） |
| mimeType | string | MIME 类型，查询失败时为空字符串 |
| extension | string | 小写扩展名（不含点），无扩展名时为空字符串 |
| path | string \| null | 本地路径。仅 `prepareFile` 返回时有值 |
| isTemporary | boolean | 是否为插件创建的临时副本 |

## 错误码

`fail` 回调返回 `FilePickerError`（实现 `IUniError`），`errSubject` 为 `easemob-file-picker`：

| errCode | 说明 | 典型场景 |
|---|---|---|
| 9010001 | 文件选择失败 | 系统选择器返回异常 |
| 9010002 | 无法打开文件选择器 | Activity 不可用、已有进行中的选择操作 |
| 9010003 | 文件访问权限被拒 | 预留（当前 SAF/系统 picker 模式下不会触发） |
| 9010004 | URI 无效或不可用 | `getFileInfo` 的 URI 已失效 |
| 9010005 | 文件元数据不可用 | 选中文件全部无法读取元数据 |
| 9010006 | 无法读取或复制文件 | `prepareFile` 复制失败、IO 异常 |
| 9010007 | 存储空间不足 | 预留 |
| 9010008 | 当前平台不支持该操作 | 在不支持的平台上调用 |

## 使用示例

### 完整流程：选文件 → 复制到本地 → 环信发送 → 清理

```ts
import { chooseFile, prepareFile, removeTempFile } from '@/uni_modules/easemob-file-picker'

chooseFile({
  mimeTypes: ['image/*', 'application/pdf'],
  multiple: false,
  success: (result) => {
    // 用户取消是正常结果，不是错误
    if (result.cancelled || result.files.length == 0) return

    prepareFile({
      uri: result.files[0].uri,
      success: (file) => {
        // file.path 为本地缓存路径，可直接传给环信 SDK 发送文件消息
        // WebIM.message.create({ type: 'file', file: ..., url: file.path ... })

        // 发送完成后清理临时文件
        removeTempFile({
          path: file.path!,
          success: (res) => { console.log('临时文件清理:', res.removed) }
        })
      },
      fail: (err) => { console.error('文件复制失败', err.errCode, err.errMsg) }
    })
  },
  fail: (err) => { console.error('选择文件失败', err.errCode, err.errMsg) }
})
```

### 只查询文件信息（不复制）

```ts
import { getFileInfo } from '@/uni_modules/easemob-file-picker'

getFileInfo({
  uri: 'content://media/external/file/12345',
  success: (info) => {
    console.log(info.name, info.size, info.mimeType)
  }
})
```

## 平台行为说明

**Android**
- 使用 SAF（Storage Access Framework），targetSdk 30+ 下无需 `READ_EXTERNAL_STORAGE` 等权限。
- 单 MIME 类型走 `Intent.setType`，多类型走 `EXTRA_MIME_TYPES`；`image/*` 等通配符由系统选择器解释。
- 回调中对可持久化的 URI 自动调用 `takePersistableUriPermission`，避免重启后 URI 失效。

**iOS**
- 最低 iOS 16，MIME 类型通过 `UTType(mimeType:)` 映射为系统内容类型；`image/*`、`video/*`、`audio/*`、`text/*` 通配映射到对应 `UTType` 预置类型；未识别的 MIME 自动跳过，全部未识别时回退为不限类型。
- 选中文件通过安全作用域资源（Security-Scoped Resource）访问，读取后即释放。
- 同时只能进行一次选择操作，重复调用返回 9010002。

**HarmonyOS Next**
- 使用 `@ohos.file.picker` 的 DocumentViewPicker。
- 临时文件命名为 `easemob-file-` 前缀，存放于应用 cacheDir。

## 注意事项

1. **用户取消不是错误**：取消走 `success` 回调且 `cancelled: true`，不要用 `fail` 判断取消。
2. **临时文件生命周期**：`prepareFile` 的副本由调用方负责通过 `removeTempFile` 清理；应用缓存目录可能被系统清理，不要长期依赖 `path`。
3. **大文件**：`prepareFile` 为同步复制，大文件可能耗时较长，建议在 UI 上加 loading 提示。
4. **Web / 小程序不支持**：本插件仅支持 App 三端，其他平台调用会返回 9010008。

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

### 开发文档
[UTS 语法](https://uniapp.dcloud.net.cn/tutorial/syntax-uts.html)
[UTS API插件](https://uniapp.dcloud.net.cn/plugin/uts-plugin.html)
[UTS uni-app兼容模式组件](https://uniapp.dcloud.net.cn/plugin/uts-component.html)
[UTS 标准模式组件](https://doc.dcloud.net.cn/uni-app-x/plugin/uts-vue-component.html)
[Hello UTS](https://gitcode.net/dcloud/hello-uts)
