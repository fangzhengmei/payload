# PayloadCMS 上传图像处理管线分析报告

## 概述

本文档深入分析 PayloadCMS 管理后台上传图片后的完整处理管线，包括多尺寸变体生成、远端存储写入、存储适配器选择机制以及访问链接生成机制。

---

## 一、整体架构概览

### 1.1 核心处理流程图

```
用户上传图片
    ↓
[1] 多部分表单解析 (fetchAPI-multipart)
    ↓
[2] 文件数据生成 (generateFileData.ts)
    ├── 图片尺寸检测
    ├── 格式转换 (Sharp)
    ├── 裁剪处理 (cropImage.ts)
    └── 多尺寸变体生成 (createImageSizes.ts)
    ↓
[3] 本地文件保存 (可选，取决于 disableLocalStorage)
    ├── 如果 disableLocalStorage=false: 保存到本地磁盘
    └── 如果 disableLocalStorage=true: 仅保留 Buffer 在内存中
    ↓
[4] 云端存储上传 (plugin-cloud-storage afterChange hook)
    ├── 直接从 req.file.data 和 req.payloadUploadSizes 读取 Buffer
    ├── 主文件上传
    └── 各尺寸变体上传
    ↓
[5] URL 生成与数据持久化
    ├── beforeChange: 首写时生成 URL
    ├── afterRead: 读取时动态补全 URL
    └── staticHandler: 代理访问时处理
```

### 1.2 关键模块位置

| 模块 | 文件路径 | 职责 |
|------|----------|------|
| 核心上传处理 | `packages/payload/src/uploads/` | 图片处理、尺寸生成、本地存储 |
| 云端存储插件 | `packages/plugin-cloud-storage/src/` | 存储适配器集成、Hook 管理 |
| S3 适配器 | `packages/storage-s3/src/` | AWS S3 / 兼容 S3 存储 |
| R2 适配器 | `packages/storage-r2/src/` | Cloudflare R2 存储 |
| GCS 适配器 | `packages/storage-gcs/src/` | Google Cloud Storage |
| Azure 适配器 | `packages/storage-azure/src/` | Azure Blob Storage |
| Vercel Blob | `packages/storage-vercel-blob/src/` | Vercel Blob Storage |
| Uploadthing | `packages/storage-uploadthing/src/` | Uploadthing 服务 |

---

## 二、多尺寸图像变体生成机制

### 2.1 配置入口

图像尺寸配置通过 Collection 的 `upload.imageSizes` 选项定义：

```typescript
// types.ts:72-121
export type ImageSize = {
  name: string                    // 尺寸名称，如 'thumbnail', 'medium'
  width?: number                   // 目标宽度
  height?: number                  // 目标高度
  fit?: 'cover' | 'contain' | 'fill' | 'inside' | 'outside'
  position?: string                // 裁剪位置
  withoutEnlargement?: boolean     // 禁止放大小图
  formatOptions?: ImageUploadFormatOptions  // 输出格式选项
  trimOptions?: ImageUploadTrimOptions      // 裁切选项
  generateImageName?: GenerateImageName     // 自定义文件名生成
  crop?: string                    // 已废弃，使用 position
  admin?: {
    disableGroupBy?: boolean
    disableListColumn?: boolean
    disableListFilter?: boolean
  }
}
```

### 2.2 核心处理流程

**入口文件**: `packages/payload/src/uploads/image-resizing/createImageSizes.ts`

#### 2.2.1 处理步骤

```typescript
// createImageSizes.ts:55-298
export async function createImageSizes({
  config, dimensions, file, focalPoint, mimeType,
  req, savedFilename, sharp, staticPath, withMetadata
}): Promise<ImageSizesResult> {
  
  // 1. 检查是否有 imageSizes 配置和 sharp 依赖
  if (!imageSizes || !sharp) {
    return { sizeData: {}, sizesToSave: [] }
  }

  // 2. 创建 Sharp 基础实例（支持动画图片）
  const sharpBase: Sharp = sharp(
    file.tempFilePath || file.data,
    { animated: fileIsAnimatedType }  // 动画图片: avif, gif, webp
  ).rotate()  // 自动根据 EXIF 旋转

  // 3. 获取原图元数据，处理 EXIF 方向
  const originalImageMeta = await sharpBase.metadata()
  if ([5, 6, 7, 8].includes(originalImageMeta.orientation!)) {
    adjustedDimensions = {
      height: dimensions.width,
      width: dimensions.height,  // 交换宽高
    }
  }

  // 4. 并行处理所有尺寸配置
  await Promise.all(
    imageSizes.map(async (imageResizeConfig) => {
      // 4.1 标准化配置
      imageResizeConfig = sanitizeResizeConfig(imageResizeConfig)
      
      // 4.2 决定是否需要调整尺寸
      const resizeAction = getImageResizeAction({
        dimensions,
        hasFocalPoint: Boolean(focalPoint),
        imageResizeConfig,
      })

      // 三种可能的 action:
      // - 'omit': 跳过此尺寸（原图小于目标尺寸且 withoutEnlargement）
      // - 'resizeWithFocalPoint': 使用焦点裁剪
      // - 'resize': 普通 resize
      
      if (resizeAction === 'omit') {
        sizes[imageResizeConfig.name] = createImageSize({})
        return
      }

      // 4.3 克隆 Sharp 实例进行处理
      const imageToResize = sharpBase.clone()
      
      // 4.4 应用焦点裁剪逻辑（如有）
      if (resizeAction === 'resizeWithFocalPoint') {
        // 计算焦点区域
        const xFocalCenter = resizeImageMeta.width * (focalPoint.x / 100)
        const yFocalCenter = resizeImageMeta.height * (focalPoint.y / 100)
        
        // 先缩放，再提取焦点区域
        resized = imageToResize.resize({
          fastShrinkOnLoad: false,
          height: prioritizeHeight ? resizeHeight : undefined,
          width: prioritizeHeight ? undefined : resizeWidth,
        }).extract({
          height: resizeHeight,
          left: Math.floor(leftBound),
          top: Math.floor(topBound),
          width: resizeWidth,
        })
      } else {
        // 普通 resize
        resized = imageToResize.resize(imageResizeConfig)
      }

      // 4.5 应用格式转换和裁切
      if (imageResizeConfig.formatOptions) {
        resized = resized.toFormat(
          imageResizeConfig.formatOptions.format,
          imageResizeConfig.formatOptions.options,
        )
      }
      if (imageResizeConfig.trimOptions) {
        resized = resized.trim(imageResizeConfig.trimOptions)
      }

      // 4.6 可选保留元数据
      const metadataAppendedFile = await optionallyAppendMetadata({
        req, sharpFile: resized, withMetadata
      })

      // 4.7 生成输出 Buffer
      const { data: bufferData, info: bufferInfo } = 
        await metadataAppendedFile.toBuffer({ resolveWithObject: true })

      // 4.8 生成文件名
      const imageNameWithDimensions = imageResizeConfig.generateImageName
        ? imageResizeConfig.generateImageName({...})
        : generateImageSizeFilename({
            extension: mimeInfo?.ext || ext,
            height: bufferInfo.height,
            outputImageName: name,
            width: bufferInfo.width,
          })
      // 默认命名格式: {name}-{width}x{height}.{ext}

      // 4.9 收集结果
      sizes[imageResizeConfig.name] = createImageSize({
        filename: imageNameWithDimensions,
        filesize: size,
        height: animated ? height / pages : height,
        mimeType: mimeInfo?.mime || mimeType,
        width,
      })

      imageSizeFiles.push({
        buffer: bufferData,
        path: `${staticPath}/${imageNameWithDimensions}`,
      })
    })
  )

  return { sizeData: sizes, sizesToSave: imageSizeFiles }
}
```

#### 2.2.2 焦点裁剪算法详解

**位置**: `createImageSizes.ts:128-213`

焦点裁剪的核心思想是：**保持用户指定的焦点区域在裁剪后的图片中心**。

```
原始图片 (1920x1080)
┌─────────────────────────────────┐
│                                 │
│         焦点 (x=75%, y=30%)     │
│              ★                   │
│                                 │
└─────────────────────────────────┘

目标尺寸: 400x400 (1:1 比例)

处理步骤:
1. 按比例缩放至较小边为 400px
2. 以焦点为中心，提取 400x400 区域
3. 如果边界超出，调整到边缘
```

```typescript
// 计算步骤:
// 1. 计算宽高比，决定缩放策略
const originalAspectRatio = adjustedDimensions.width / adjustedDimensions.height
const resizeAspectRatio = resizeWidth / resizeHeight
const prioritizeHeight = resizeAspectRatio < originalAspectRatio

// 2. 先按一边缩放
resized = imageToResize.resize({
  fastShrinkOnLoad: false,
  height: prioritizeHeight ? resizeHeight : undefined,
  width: prioritizeHeight ? undefined : resizeWidth,
})

// 3. 计算焦点在缩放后图片中的像素位置
const xFocalCenter = resizeImageMeta.width * (focalPoint.x / 100)
const yFocalCenter = resizeImageMeta.height * (focalPoint.y / 100)

// 4. 计算裁剪边界
const halfResizeX = resizeWidth / 2
let leftBound = xFocalCenter - halfResizeX

// 5. 边界检查：确保不超出图片范围
if (xFocalCenter + halfResizeX > resizeImageMeta.width) {
  leftBound = resizeImageMeta.width - resizeWidth  // 靠右对齐
}
if (leftBound < 0) {
  leftBound = 0  // 靠左对齐
}

// 6. 执行提取
resized = resized.extract({
  height: resizeHeight,
  left: Math.floor(leftBound),
  top: Math.floor(topBound),
  width: resizeWidth,
})
```

### 2.3 尺寸生成决策逻辑

**文件**: `packages/payload/src/uploads/image-resizing/getImageResizeAction.ts`

```typescript
export function getImageResizeAction({
  dimensions, hasFocalPoint, imageResizeConfig,
}): 'omit' | 'resize' | 'resizeWithFocalPoint' {
  
  const { width: targetWidth, height: targetHeight, withoutEnlargement } = imageResizeConfig

  // 1. 检查 withoutEnlargement 选项
  if (withoutEnlargement === undefined) {
    // 默认行为: 如果原图宽高都小于目标尺寸，跳过
    if (dimensions.width < targetWidth && dimensions.height < targetHeight) {
      return 'omit'
    }
  } else if (withoutEnlargement === true) {
    // 小图保持原样，不放大
    if (dimensions.width < targetWidth && dimensions.height < targetHeight) {
      return 'omit'  // 返回 null 的 FileSize
    }
  }
  // withoutEnlargement === false: 始终放大到目标尺寸

  // 2. 决定是否使用焦点裁剪
  return hasFocalPoint ? 'resizeWithFocalPoint' : 'resize'
}
```

### 2.4 文件名生成

**默认实现**: `packages/payload/src/uploads/image-resizing/generateImageSizeFilename.ts`

```typescript
export function generateImageSizeFilename({
  extension, height, outputImageName, width,
}: GenerateImageSizeFilenameArgs): string {
  return `${outputImageName}-${width}x${height}.${extension}`
}
// 示例: "photo-1920x1080.jpg" → thumbnail 尺寸 → "photo-400x300.jpg"
```

**自定义实现** 可通过 `ImageSize.generateImageName` 配置：

```typescript
generateImageName: ({ extension, height, originalName, sizeName, width }) => {
  return `${originalName}_${sizeName}_${width}x${height}.${extension}`
  // 输出: "photo_thumbnail_400x300.jpg"
}
```

---

## 三、存储适配器选择机制

### 3.1 适配器架构

PayloadCMS 使用基于 **Plugin + Adapter** 的双层架构：

```
┌─────────────────────────────────────────────────────────────┐
│                      Payload Core                             │
│  (generateFileData, createImageSizes, saveBufferToFile)     │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│              plugin-cloud-storage (核心集成层)                │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  Hooks: afterChange, afterDelete, beforeChange      │   │
│  │  Fields: prefix, url (各 image sizes)                │   │
│  │  Handlers: staticHandler (文件访问)                   │   │
│  └─────────────────────────────────────────────────────┘   │
└───────────────────────────┬─────────────────────────────────┘
                            │
            ┌───────────────┼───────────────┐
            ▼               ▼               ▼
    ┌───────────┐   ┌───────────┐   ┌───────────┐
    │  S3/R2    │   │   GCS     │   │   Azure   │  ... 更多适配器
    │  Adapter  │   │  Adapter  │   │  Adapter  │
    └───────────┘   └───────────┘   └───────────┘
```

### 3.2 适配器接口定义

**文件**: `packages/plugin-cloud-storage/src/types.ts`

```typescript
// 适配器工厂函数（供用户配置使用）
export type Adapter = (args: {
  collection: CollectionConfig
  prefix?: string              // 集合级前缀
}) => GeneratedAdapter

// 运行时适配器实例
export interface GeneratedAdapter {
  name: string                  // 适配器标识: 's3', 'r2', 'gcs', 'azure' 等
  
  // 核心方法
  handleUpload: HandleUpload    // 上传文件
  handleDelete: HandleDelete    // 删除文件
  staticHandler: StaticHandler  // 文件访问处理（代理/重定向）
  
  // 可选方法
  generateURL?: GenerateURL     // 生成公开访问 URL
  fields?: Field[]              // 额外注入的字段
  clientUploads?: ClientUploadsConfig  // 客户端直传配置
  onInit?: () => void           // 初始化回调
}
```

### 3.3 各方法详细说明

#### 3.3.1 handleUpload

```typescript
export type HandleUpload = (args: {
  clientUploadContext: unknown   // 客户端直传上下文
  collection: CollectionConfig    // 集合配置
  data: any                       // 文档数据（含 prefix 等）
  file: File                      // 要上传的文件
  req: PayloadRequest
}) => Partial<FileData & TypeWithID> | Promise<void> | void
```

**返回值说明**:
- 返回对象: 包含需要更新到数据库的元数据（如 CDN URL）
- 返回 void/undefined: 使用默认行为

#### 3.3.2 handleDelete

```typescript
export type HandleDelete = (args: {
  collection: CollectionConfig
  doc: FileData & TypeWithID & TypeWithPrefix  // 包含 prefix 字段
  filename: string
  req: PayloadRequest
}) => Promise<void> | void
```

#### 3.3.3 generateURL

```typescript
export type GenerateURL = (args: {
  collection: CollectionConfig
  data: any                       // 文档数据
  filename: string
  prefix?: string
}) => Promise<string> | string
```

#### 3.3.4 staticHandler

用于文件访问路由 `/api/{collection}/file/{filename}` 的处理：

```typescript
export type StaticHandler = (
  req: PayloadRequest,
  args: {
    doc?: TypeWithID
    headers?: Headers
    params: {
      clientUploadContext?: unknown
      collection: string
      filename: string
      prefix?: string              // 查询参数传递的 prefix
    }
  },
) => Promise<Response> | Response
```

**常见实现模式**:
1. **代理模式**: 从云存储读取文件，通过 Payload 服务器返回（可应用访问控制）
2. **重定向模式**: 返回 302 重定向到云存储的公开 URL 或预签名 URL
3. **预签名模式**: 生成带过期时间的访问 URL

### 3.4 配置方式示例

#### 3.4.1 S3 适配器配置

```typescript
// packages/storage-s3/src/index.ts
import { s3Storage } from '@payloadcms/storage-s3'

export const config = buildConfig({
  collections: [
    {
      slug: 'media',
      upload: true,  // 启用上传
      fields: [...],
    },
  ],
  plugins: [
    s3Storage({
      bucket: process.env.S3_BUCKET,
      config: {
        credentials: {
          accessKeyId: process.env.S3_ACCESS_KEY_ID,
          secretAccessKey: process.env.S3_SECRET_ACCESS_KEY,
        },
        region: process.env.S3_REGION,
        // endpoint: 'https://<account-id>.r2.cloudflarestorage.com',  // R2 兼容
      },
      collections: {
        media: {
          prefix: 'uploads/media',    // 集合级前缀
          // generateFileURL: customURLGenerator,  // 自定义 URL
          // disablePayloadAccessControl: true,      // 禁用 Payload 访问控制
        },
      },
      acl: 'public-read',             // 访问控制列表
      // disableLocalStorage: true,    // 默认: true
      useCompositePrefixes: true,     // 组合前缀模式
    }),
  ],
})
```

#### 3.4.2 适配器注册机制

**S3 适配器内部实现**: `packages/storage-s3/src/index.ts:122-252`

```typescript
export const s3Storage = (s3StorageOptions) => (incomingConfig) => {
  // 1. 创建/复用 S3 客户端（带缓存）
  const getStorageClient: () => AWS.S3 = () => {
    const cacheKey = s3StorageOptions.clientCacheKey || `s3:${s3StorageOptions.bucket}`
    if (s3Clients.has(cacheKey)) {
      return s3Clients.get(cacheKey)!
    }
    s3Clients.set(cacheKey, new AWS.S3({...}))
    return s3Clients.get(cacheKey)!
  }

  // 2. 为每个集合配置适配器
  const collectionsWithAdapter = Object.entries(
    s3StorageOptions.collections,
  ).reduce((acc, [slug, collOptions]) => ({
    ...acc,
    [slug]: {
      ...(collOptions === true ? {} : collOptions),
      adapter: createS3Adapter({
        acl: s3StorageOptions.acl,
        bucket: s3StorageOptions.bucket,
        config: s3StorageOptions.config,
        getStorageClient,
        useCompositePrefixes: s3StorageOptions.useCompositePrefixes,
        // ...
      }),
    },
  }), {})

  // 3. 修改集合配置，设置 disableLocalStorage
  const config = {
    ...incomingConfig,
    collections: (incomingConfig.collections || []).map((collection) => {
      if (!collectionsWithAdapter[collection.slug]) {
        return collection
      }
      return {
        ...collection,
        upload: {
          ...collection.upload,
          disableLocalStorage: true,  // 关键：禁用本地存储
        },
      }
    }),
  }

  // 4. 调用底层 cloudStoragePlugin
  return cloudStoragePlugin({
    collections: collectionsWithAdapter,
    useCompositePrefixes: s3StorageOptions.useCompositePrefixes,
  })(config)
}
```

### 3.5 可用适配器一览

| 适配器包 | 存储服务 | 关键特性 |
|---------|---------|---------|
| `@payloadcms/storage-s3` | AWS S3 / 兼容 S3 | ACL 控制、分段上传、预签名 URL |
| `@payloadcms/storage-r2` | Cloudflare R2 | Workers 环境原生支持、无出口费 |
| `@payloadcms/storage-gcs` | Google Cloud Storage | 签名 URL、IAM 集成 |
| `@payloadcms/storage-azure` | Azure Blob Storage | SAS 令牌、Blob 层级 |
| `@payloadcms/storage-vercel-blob` | Vercel Blob | Edge 友好、简单 API |
| `@payloadcms/storage-uploadthing` | Uploadthing | 类型安全、内置图像处理 |

---

## 四、远端存储写入流程

### 4.1 完整写入时序（基于 create.ts 源码）

根据 `packages/payload/src/collections/operations/create.ts` 的源码，正确的执行顺序如下：

```
┌────────────────────────────────────────────────────────────────────────────┐
│  阶段 1: 图片处理 & Hook 执行 (首次写库之前)                                  │
└────────────────────────────────────────────────────────────────────────────┘

1. beforeOperation - Collection 级别 Hook
         ↓
2. Access 权限检查
         ↓
3. generateFileData (关键：图片处理)
   ├── 调用 createImageSizes 生成各尺寸变体
   ├── Buffer 存储到:
   │     ├── req.file.data = 主文件 Buffer
   │     └── req.payloadUploadSizes = { sizeName: Buffer, ... }
   └── 返回: { data: newFileData, files: filesToUpload }
         ↓
4. beforeValidate - Fields → Collection
         ↓
5. beforeChange - Collection 级别 Hook
         ↓
6. beforeChange - Fields 级别 Hook (关键：URL 首写)
   ├── url 字段的 beforeChange hook 执行
   ├── 根据配置生成 URL
   └── 结果存储到 resultWithLocales（即将写入数据库的数据）
         ↓
7. uploadFiles (条件执行)
   └── 仅当 !disableLocalStorage 时，保存到本地磁盘
         ↓
8. db.create (首次写库)
   └── 将 resultWithLocales 写入数据库

┌────────────────────────────────────────────────────────────────────────────┐
│  阶段 2: 读取补全 & 云端上传 (首次写库之后)                                   │
└────────────────────────────────────────────────────────────────────────────┘

9. afterRead - Fields → Collection (关键：URL 读取补全)
   ├── url 字段的 afterRead hook 执行
   ├── 动态补全 URL（如追加 ?prefix=... 参数）
   └── 结果不存储到数据库，只在返回时生效
         ↓
10. afterChange - Fields 级别 Hook
          ↓
11. afterChange - Collection 级别 Hook (关键：云端上传)
    ├── cloud-storage 的 hook 执行
    ├── 从 req.file.data 和 req.payloadUploadSizes 读取 Buffer
    ├── 调用 adapter.handleUpload 上传到云端
    └── 如果 adapter 返回元数据，可能执行二次写库
          ↓
12. afterOperation - Collection 级别 Hook
```

**关键修正点**：

| 之前的错误描述 | 正确描述 |
|---------------|---------|
| `beforeChange` 在首次写库**之后**执行 | `beforeChange` 在首次写库**之前**执行，用于生成 URL 并写入数据库 |
| `afterRead` 在 `afterChange` **之后**执行 | `afterRead` 在 `afterChange` **之前**执行，用于动态补全 URL |
| Buffer 在 `afterChange` 之后才可用 | Buffer 在 `generateFileData` 阶段就已生成，存储在 `req.file.data` 和 `req.payloadUploadSizes` |

### 4.2 核心 Hook 解析

#### 4.2.1 afterChange Hook（关键上传入口）

**文件**: `packages/plugin-cloud-storage/src/hooks/afterChange.ts`

```typescript
export const getAfterChangeHook = ({ adapter, collection }) => 
  async ({ doc, operation, previousDoc, req }) => {
    
    // 防止无限循环：内部更新时跳过
    if (req.context?.skipCloudStorage) {
      return doc
    }

    try {
      // 1. 获取所有待上传文件（主文件 + 各尺寸变体）
      const files = getIncomingFiles({ data: doc, req })

      if (files.length > 0) {
        // 2. 更新操作：先删除旧文件
        if (previousDoc && operation === 'update') {
          let filesToDelete: string[] = []
          
          // 收集旧文件名
          if (typeof previousDoc?.filename === 'string') {
            filesToDelete.push(previousDoc.filename)
          }
          if (typeof previousDoc.sizes === 'object') {
            filesToDelete = filesToDelete.concat(
              Object.values(previousDoc.sizes)
                .map(size => size?.filename as string)
            )
          }

          // 并行删除
          await Promise.all(
            filesToDelete.map(async (filename) => {
              if (filename) {
                await adapter.handleDelete({
                  collection, doc: previousDoc, filename, req
                })
              }
            })
          )
        }

        // 3. 并行上传新文件（排除客户端直传的文件）
        const uploadResults = await Promise.all(
          files
            .filter(file => !file.clientUploadContext)
            .map(file => adapter.handleUpload({
              clientUploadContext: file.clientUploadContext,
              collection,
              data: doc,
              file,
              req,
            }))
        )

        // 4. 合并上传返回的元数据
        const uploadMetadata = uploadResults
          .filter((result): result is Partial<FileData & TypeWithID> =>
            result != null && typeof result === 'object'
          )
          .reduce((acc, metadata) => ({ ...acc, ...metadata }), {})

        // 5. 如果有元数据更新，二次写入数据库
        if (Object.keys(uploadMetadata).length > 0) {
          try {
            // 设置跳过标记，防止循环
            req.context = req.context || {}
            req.context.skipCloudStorage = true
            
            // 清理 request 中的文件数据
            req.file = undefined
            req.payloadUploadSizes = undefined

            // 执行更新
            await req.payload.update({
              id: doc.id,
              collection: collection.slug,
              data: uploadMetadata,
              depth: 0,
              req,
            })
            
            delete req.context.skipCloudStorage
            return { ...doc, ...uploadMetadata }
          } catch (updateError) {
            // 记录警告但不抛出
            req.payload.logger.warn(`Failed to persist upload data...`)
          }
        }
      }
    } catch (err) {
      req.payload.logger.error(`Error uploading files...`)
      throw err
    }
    return doc
  }
```

#### 4.2.2 getIncomingFiles - 收集待上传文件

**文件**: `packages/plugin-cloud-storage/src/utilities/getIncomingFiles.ts`

```typescript
export function getIncomingFiles({ data, req }): File[] {
  // 支持从 context 恢复（防止 req.file 被清空）
  const ctx = req.context?._payloadCloudStorage
  const file = req.file ?? ctx?.file
  const payloadUploadSizes = req.payloadUploadSizes ?? ctx?.uploadSizes

  let files: File[] = []

  if (file && data.filename && data.mimeType) {
    // 主文件
    const mainFile: File = {
      buffer: file.data,
      clientUploadContext: file.clientUploadContext,
      filename: data.filename,
      filesize: file.size,
      mimeType: data.mimeType,
      tempFilePath: file.tempFilePath,
    }
    files = [mainFile]

    // 各尺寸变体
    if (data?.sizes) {
      Object.entries(data.sizes).forEach(([key, resizedFileData]) => {
        if (payloadUploadSizes?.[key] && resizedFileData.mimeType) {
          files = files.concat([{
            buffer: payloadUploadSizes[key],
            filename: `${resizedFileData.filename}`,
            filesize: payloadUploadSizes[key].length,
            mimeType: resizedFileData.mimeType,
          }])
        }
      })
    }
  }

  return files
}
```

### 4.3 S3 上传实现详解

**文件**: `packages/storage-s3/src/uploadFile.ts`

```typescript
const multipartThreshold = 1024 * 1024 * 50  // 50MB 阈值

export async function uploadFile({
  acl, bucket, buffer, client, collectionPrefix,
  docPrefix, filename, mimeType, tempFilePath,
  useCompositePrefixes = false,
}: UploadArgs): Promise<void> {
  
  // 1. 计算文件 Key（路径）
  const { fileKey } = getFileKey({
    collectionPrefix,
    docPrefix,
    filename,
    useCompositePrefixes,
  })
  // 示例: 
  // - useCompositePrefixes=true:  {collectionPrefix}/{docPrefix}/{filename}
  // - useCompositePrefixes=false: {docPrefix || collectionPrefix}/{filename}

  // 2. 选择数据源（Buffer 或文件流）
  const fileBufferOrStream = tempFilePath 
    ? fs.createReadStream(tempFilePath)  // 大文件使用流
    : buffer

  // 3. 小文件：简单 putObject
  if (buffer.length > 0 && buffer.length < multipartThreshold) {
    await client.putObject({
      ACL: acl,                    // 'private' | 'public-read'
      Body: fileBufferOrStream,
      Bucket: bucket,
      ContentType: mimeType,
      Key: fileKey,
    })
    return
  }

  // 4. 大文件：分段并行上传
  const parallelUploadS3 = new Upload({
    client,
    params: {
      ACL: acl,
      Body: fileBufferOrStream,
      Bucket: bucket,
      ContentType: mimeType,
      Key: fileKey,
    },
    partSize: multipartThreshold,   // 每段 50MB
    queueSize: 4,                    // 并行 4 段
  })

  await parallelUploadS3.done()
}
```

### 4.4 文件 Key（路径）计算

**文件**: `packages/plugin-cloud-storage/src/utilities/getFileKey.ts`

```typescript
export function getFileKey({
  collectionPrefix, docPrefix, filename, useCompositePrefixes = false,
}): GetFileKeyResult {
  
  const safeCollectionPrefix = sanitizePrefix(collectionPrefix || '')
  const safeDocPrefix = sanitizePrefix(docPrefix || '')
  const safeFilename = sanitizeFilename(filename)

  // 两种模式:
  const fileKey = useCompositePrefixes
    // 组合模式: 集合前缀 + 文档前缀 + 文件名
    ? path.posix.join(safeCollectionPrefix, safeDocPrefix, safeFilename)
    // 覆盖模式: 文档前缀存在则用文档前缀，否则用集合前缀
    : path.posix.join(safeDocPrefix || safeCollectionPrefix, safeFilename)

  return { fileKey, ... }
}
```

**示例**:

| 配置 | collectionPrefix | docPrefix | filename | useCompositePrefixes | 结果 fileKey |
|------|-----------------|-----------|----------|----------------------|--------------|
| 组合模式 | `uploads/` | `2024/05/` | `photo.jpg` | `true` | `uploads/2024/05/photo.jpg` |
| 覆盖模式(有 docPrefix) | `uploads/` | `avatars/` | `user1.jpg` | `false` | `avatars/user1.jpg` |
| 覆盖模式(无 docPrefix) | `uploads/` | `''` | `file.pdf` | `false` | `uploads/file.pdf` |

---

## 五、访问链接生成机制（三个时机）

PayloadCMS 的 URL 生成分为 **三个不同的时机**，每个时机有不同的用途和行为：

```
┌─────────────────────────────────────────────────────────────────────────┐
│  时机 1: beforeChange Hook (首写/写入数据库前)                            │
│  ───────────────────────────────────────────────────────────────────────  │
│  触发: 数据库写入之前                                                       │
│  位置: url 字段的 hooks.beforeChange                                       │
│  用途: 生成 URL 并存储到数据库                                              │
│  特点: 只在创建/更新时执行一次，结果持久化                                   │
└─────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────┐
│  时机 2: afterRead Hook (读取补全/运行时)                                  │
│  ───────────────────────────────────────────────────────────────────────  │
│  触发: 从数据库读取数据时                                                   │
│  位置: url 字段的 hooks.afterRead                                          │
│  用途: 动态生成/补全 URL，可能不存储到数据库                                 │
│  特点: 每次读取都执行，可动态计算（如签名 URL、追加 prefix 参数）            │
└─────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────┐
│  时机 3: staticHandler (代理访问/请求时)                                   │
│  ───────────────────────────────────────────────────────────────────────  │
│  触发: 用户访问 /api/{collection}/file/{filename} 时                      │
│  位置: upload.handlers 数组                                                │
│  用途: 实际文件访问处理                                                     │
│  特点: 可能代理返回文件，或重定向到云端 URL，或返回签名 URL                 │
└─────────────────────────────────────────────────────────────────────────┘
```

### 5.1 时机 1: beforeChange Hook（首写/写入数据库前）

**文件**: `packages/plugin-cloud-storage/src/hooks/beforeChange.ts`

**触发时机**: 
- 创建文档时 (`operation: 'create'`)
- 更新文档时 (`operation: 'update'`)
- 在数据库写入**之前**执行

**行为**:
```typescript
export const getBeforeChangeHook = ({
  adapter, collection, disablePayloadAccessControl, generateFileURL, size,
}): FieldHook => async ({ data, originalDoc, value }) => {
  
  const filename = size 
    ? data?.sizes?.[size.name]?.filename 
    : data?.filename
  const prefix = data?.prefix
  let url = value

  // 优先级 1: 用户自定义 generateFileURL
  if (generateFileURL && filename) {
    url = await generateFileURL({
      collection, filename, prefix, size,
    })
  } 
  // 优先级 2: 禁用访问控制时使用适配器生成的 URL
  else if (disablePayloadAccessControl && filename && adapter.generateURL) {
    url = await adapter.generateURL({
      collection,
      data: data || originalDoc,
      filename,
      prefix,
    })
  }
  // 优先级 3: 保持原值（可能是 Payload 代理 URL 格式）
  // 不做任何修改，value 保持不变

  return url
}
```

**URL 策略优先级（首写时）**:

| 优先级 | 策略 | 触发条件 | 结果存储 |
|--------|------|---------|---------|
| 1 | `generateFileURL` | 用户配置了此函数 | ✅ 存储到数据库 |
| 2 | `adapter.generateURL` | `disablePayloadAccessControl === true` | ✅ 存储到数据库 |
| 3 | Payload 默认 URL | 其他情况 | ⚠️ 可能是代理格式，后续由 afterRead 补全 |

### 5.2 时机 2: afterRead Hook（读取补全/运行时）

**文件**: `packages/plugin-cloud-storage/src/hooks/afterRead.ts`

**触发时机**: 
- 从数据库读取数据时（每次查询、每次列表获取）
- 在数据返回给用户**之前**执行

**行为**:
```typescript
export const getAfterReadHook = ({
  adapter, collection, disablePayloadAccessControl, generateFileURL, size,
}): FieldHook => async ({ data, value }) => {
  
  const filename = size ? data?.sizes?.[size.name]?.filename : data?.filename
  const prefix = data?.prefix
  let url = value

  if (filename) {
    // 优先级 1: 用户自定义 generateFileURL
    if (generateFileURL) {
      url = await generateFileURL({ collection, filename, prefix, size })
    } 
    // 优先级 2: 禁用访问控制时使用适配器生成的 URL
    else if (disablePayloadAccessControl && adapter.generateURL) {
      url = await adapter.generateURL({ collection, data, filename, prefix })
    } 
    // 优先级 3: 代理 URL 模式，追加 prefix 查询参数
    else if (url && prefix) {
      // 关键点: 数据库中存储的 URL 可能没有 prefix
      // afterRead 时动态追加 ?prefix=... 参数
      const separator = url.includes('?') ? '&' : '?'
      url = `${url}${separator}prefix=${encodeURIComponent(prefix)}`
    }
  }

  return url
}
```

**afterRead 的关键作用**:

1. **动态 URL 生成**: 对于签名 URL、CDN URL 等，每次读取时重新生成
2. **prefix 参数补全**: 代理 URL 模式下，数据库存储的是 `/api/media/file/photo.jpg`，afterRead 时动态追加 `?prefix=uploads/`
3. **运行时计算**: URL 可能不存储在数据库，而是每次读取时计算

**示例场景**:

```
数据库中存储的文档:
{
  filename: 'photo.jpg',
  prefix: 'uploads/2024/05/',
  url: '/api/media/file/photo.jpg',  // 没有 prefix 参数
  sizes: {
    thumbnail: {
      filename: 'photo-400x300.jpg',
      url: '/api/media/file/photo-400x300.jpg'
    }
  }
}

afterRead 执行后返回给用户的数据:
{
  filename: 'photo.jpg',
  prefix: 'uploads/2024/05/',
  url: '/api/media/file/photo.jpg?prefix=uploads%2F2024%2F05%2F',  // 追加了 prefix
  sizes: {
    thumbnail: {
      filename: 'photo-400x300.jpg',
      url: '/api/media/file/photo-400x300.jpg?prefix=uploads%2F2024%2F05%2F'
    }
  }
}
```

### 5.3 时机 3: staticHandler（代理访问/请求时）

**文件**: 各适配器的 `getFile.ts` 或 `staticHandler` 实现

**触发时机**: 
- 用户实际访问文件 URL 时
- 如 `GET /api/media/file/photo.jpg?prefix=uploads/`

**行为模式**:

staticHandler 有三种常见实现模式：

#### 模式 A: 代理模式（默认，可应用访问控制）

```typescript
// S3 getFile.ts 简化版
export function getFile({
  bucket, client, filename, prefixQueryParam, useCompositePrefixes,
}: GetFileArgs): Response {
  
  // 1. 计算文件 Key（使用 URL 中的 prefix 查询参数）
  const { fileKey } = getFileKey({
    collectionPrefix,
    docPrefix: prefixQueryParam,  // 从查询参数获取 prefix
    filename,
    useCompositePrefixes,
  })

  // 2. 从云存储获取文件
  const getObjectRequest = client.getObject({
    Bucket: bucket,
    Key: fileKey,
  })

  // 3. 代理返回给客户端
  return new Response(
    getObjectRequest.Body?.transformToWebStream(),
    {
      status: 200,
      headers: {
        'Content-Type': getObjectRequest.ContentType || 'application/octet-stream',
        // ...
      },
    }
  )
}
```

**代理模式的优势**:
- 可应用 Payload 的访问控制（权限检查、API 密钥验证等）
- 可隐藏真实云存储 URL
- 可添加自定义响应头

#### 模式 B: 预签名 URL 重定向

```typescript
// 启用 signedDownloads 时
if (signedDownloads) {
  const signedUrl = await file.getSignedUrl({
    action: 'read',
    expires: expiresIn,  // 如 900 秒
  })
  
  // 返回 302 重定向
  return new Response(null, {
    status: 302,
    headers: {
      Location: signedUrl,
    },
  })
}
```

#### 模式 C: 直接云端 URL（disablePayloadAccessControl）

当配置了 `disablePayloadAccessControl: true` 时：
1. beforeChange 时直接调用 `adapter.generateURL` 生成云端 URL
2. URL 存储到数据库
3. afterRead 时可能再次生成（如果是动态签名）
4. 用户直接访问云端 URL，不经过 Payload 代理

### 5.4 三种时机的完整交互示例

让我们通过一个完整的上传和访问流程来理解三种时机的协同工作：

```
┌────────────────────────────────────────────────────────────────────────────┐
│  阶段 1: 上传图片                                                            │
└────────────────────────────────────────────────────────────────────────────┘

1. 用户上传图片
   ↓
2. generateFileData 处理图片，生成主文件和各尺寸变体
   - req.file.data = 主文件 Buffer
   - req.payloadUploadSizes = { thumbnail: Buffer, medium: Buffer, ... }
   ↓
3. beforeChange Hook（时机 1: 首写）执行
   - 为 url 字段生成初始值
   - 如果是代理模式: url = '/api/media/file/photo.jpg' (无 prefix)
   - 如果是 disablePayloadAccessControl: url = adapter.generateURL(...)
   ↓
4. 数据库首次写入
   - 存储 filename, prefix, sizes, 以及 beforeChange 生成的 url
   ↓
5. afterChange Hook 执行
   - 从 req.file.data 和 req.payloadUploadSizes 读取 Buffer
   - 调用 adapter.handleUpload 上传到云端
   - 如果 adapter 返回元数据，二次写入数据库

┌────────────────────────────────────────────────────────────────────────────┐
│  阶段 2: 读取文档（查询 API）                                                 │
└────────────────────────────────────────────────────────────────────────────┘

1. 用户调用 GET /api/media?limit=10
   ↓
2. 数据库查询，返回文档数据
   {
     filename: 'photo.jpg',
     prefix: 'uploads/2024/05/',
     url: '/api/media/file/photo.jpg',  // 数据库中存储的 URL 无 prefix
     sizes: {
       thumbnail: {
         filename: 'photo-400x300.jpg',
         url: '/api/media/file/photo-400x300.jpg'
       }
     }
   }
   ↓
3. afterRead Hook（时机 2: 读取补全）执行
   - 检测到 prefix 存在
   - 动态追加 ?prefix=... 参数
   ↓
4. 返回给用户的数据
   {
     filename: 'photo.jpg',
     prefix: 'uploads/2024/05/',
     url: '/api/media/file/photo.jpg?prefix=uploads%2F2024%2F05%2F',  // 补全了 prefix
     sizes: {
       thumbnail: {
         filename: 'photo-400x300.jpg',
         url: '/api/media/file/photo-400x300.jpg?prefix=uploads%2F2024%2F05%2F'
       }
     }
   }

┌────────────────────────────────────────────────────────────────────────────┐
│  阶段 3: 实际访问文件                                                        │
└────────────────────────────────────────────────────────────────────────────┘

1. 用户访问补全后的 URL:
   GET /api/media/file/photo.jpg?prefix=uploads%2F2024%2F05%2F
   ↓
2. Payload 文件路由匹配
   ↓
3. staticHandler（时机 3: 代理访问）执行
   - 从查询参数解析 prefix = 'uploads/2024/05/'
   - 计算完整 fileKey = 'uploads/2024/05/photo.jpg'
   - 从云存储获取文件
   - 代理返回给用户（或重定向到签名 URL）
```

### 5.5 本地存储 URL 生成

**文件**: `packages/payload/src/uploads/generateFilePathOrURL.ts`

本地存储模式下，URL 生成逻辑相对简单：

```typescript
export function generateFilePathOrURL({
  collectionSlug, config, filename, relative, serverURL, urlOrPath,
}): null | string {
  
  // 1. 如果已有外部 URL，直接返回
  if (urlOrPath) {
    if (!urlOrPath.startsWith('/') && !urlOrPath.startsWith(serverURL || '')) {
      return urlOrPath  // 外部 URL
    }
  }

  // 2. 构建本地文件 API URL
  if (filename) {
    return formatAdminURL({
      apiRoute: config.routes?.api || '',
      path: `/${collectionSlug}/file/${encodeURIComponent(filename)}`,
      relative,
      serverURL: config.serverURL,
    })
  }

  return null
}
```

**生成的 URL 格式**:
- `relative=false`: `{serverURL}/api/{collection}/file/{filename}`
- `relative=true`: `/api/{collection}/file/{filename}`

### 5.6 URL 策略选择总结

| 策略 | beforeChange (首写) | afterRead (读取) | staticHandler (访问) | 适用场景 |
|------|---------------------|------------------|---------------------|---------|
| **代理模式 (默认)** | 生成 `/api/...` URL | 追加 `?prefix=...` | 从云存储读取并代理返回 | 需要访问控制、隐藏真实 URL |
| **直接云端 URL** | 调用 `adapter.generateURL` 生成并存储 | 可能重新生成（如签名 URL） | 用户直接访问云端，不经过 Payload | 公开文件、CDN 加速 |
| **签名 URL** | 可能不存储（每次动态生成） | 每次读取时生成新的签名 URL | 返回 302 重定向到签名 URL | 私有文件、带过期时间的访问 |
| **自定义 URL** | 调用用户的 `generateFileURL` | 同样调用 `generateFileURL` | 取决于自定义实现 | 完全自定义 URL 逻辑 |

---

## 六、关键数据结构

### 6.1 FileData（文档中存储的文件信息）

```typescript
// packages/payload/src/uploads/types.ts:31-42
export type FileData = {
  filename: string           // 文件名
  filesize: number           // 文件大小（字节）
  height: number             // 图片高度
  width: number              // 图片宽度
  mimeType: string           // MIME 类型
  focalX?: number            // 焦点 X 坐标 (0-100)
  focalY?: number            // 焦点 Y 坐标 (0-100)
  url?: string               // 访问 URL
  tempFilePath?: string      // 临时文件路径
  sizes: FileSizes           // 各尺寸变体
}

export type FileSizes = {
  [size: string]: FileSize
}

export type FileSize = {
  filename: null | string
  filesize: null | number
  height: null | number
  mimeType: null | string
  url?: null | string        // TODO V4: make non-optional
  width: null | number
}
```

### 6.2 集合上传配置

```typescript
// packages/payload/src/uploads/types.ts:147-316
export type UploadConfig = {
  // 存储适配器
  adapter?: string           // 适配器名称（用于遥测）
  disableLocalStorage?: boolean  // 禁用本地存储（默认: false）
  staticDir?: string         // 静态文件目录（默认: 集合 slug）
  
  // 图片处理
  imageSizes?: ImageSize[]   // 尺寸配置数组
  focalPoint?: boolean       // 启用焦点功能（默认: true）
  crop?: boolean              // 启用裁剪（默认: true）
  resizeOptions?: ResizeOptions        // 原图 resize 选项
  formatOptions?: ImageUploadFormatOptions  // 原图格式转换
  trimOptions?: ImageUploadTrimOptions      // 原图裁切
  constructorOptions?: SharpOptions        // Sharp 构造选项
  withMetadata?: WithMetadata // 是否保留元数据
  
  // 文件限制
  mimeTypes?: string[]        // 允许的 MIME 类型
  allowRestrictedFileTypes?: boolean  // 允许危险文件类型
  filesRequiredOnCreate?: boolean      // 创建时必须有文件
  
  // URL 和访问
  adminThumbnail?: GetAdminThumbnail | string
  pasteURL?: { allowList: AllowList } | false  // URL 粘贴配置
  externalFileHeaderFilter?: (headers) => headers
  handlers?: ((req, args) => Response | void)[]  // 自定义文件处理
  modifyResponseHeaders?: ({ headers }) => Headers | void
}
```

---

## 七、完整执行流程总结

### 7.1 上传时序（带云端存储）

```
1. 用户在 Admin UI 选择图片上传
   ↓
2. 前端发送 multipart/form-data POST 请求到 /api/{collection}
   ↓
3. fetchAPI-multipart 解析请求，提取文件到 req.file
   ↓
4. collections 操作进入 beforeChange 钩子链
   ↓
5. generateFileData 执行（核心处理）:
   a. 检测文件类型和尺寸
   b. 应用裁剪、格式转换
   c. 调用 createImageSizes 生成所有尺寸变体
   d. 收集 filesToSave 数组（主文件 + 变体）
   e. 返回修改后的 data（包含 sizes 信息）
   ↓
6. cloudStorage beforeChange hook 执行:
   a. 为 url 字段（及各 sizes 的 url）生成初始值
   ↓
7. 数据库首次写入（无云端 URL）
   ↓
8. afterChange 钩子链执行
   ↓
9. cloudStorage afterChange hook 执行:
   a. 调用 getIncomingFiles 收集主文件和所有变体的 Buffer
   b. 如果是更新操作，先调用 adapter.handleDelete 删除旧文件
   c. 并行调用 adapter.handleUpload 上传所有新文件
   d. 合并上传返回的元数据
   e. 调用 payload.update 二次写入数据库（更新 url 等字段）
   ↓
10. 返回完整文档给前端
```

### 7.2 文件读取时序

```
1. 前端请求图片 URL（如 /api/media/file/photo.jpg?prefix=uploads/）
   ↓
2. Payload 文件路由匹配: GET /api/{collection}/file/{filename}
   ↓
3. 检查 upload.handlers 数组
   ↓
4. cloudStorage staticHandler 执行:
   a. 如果 signedDownloads 启用:
      i. 生成预签名 URL
      ii. 返回 302 重定向
   b. 否则（代理模式）:
      i. 从云存储获取文件流
      ii. 转换为 Web Response 返回
      iii. 可应用访问控制检查
```

---

## 八、关键代码位置索引

| 功能 | 文件路径 | 关键函数/类型 |
|------|----------|--------------|
| **核心上传处理** | | |
| 主文件数据生成 | `packages/payload/src/uploads/generateFileData.ts` | `generateFileData` |
| 多尺寸生成 | `packages/payload/src/uploads/image-resizing/createImageSizes.ts` | `createImageSizes` |
| 单尺寸生成 | `packages/payload/src/uploads/image-resizing/createImageSize.ts` | `createImageSize` |
| 尺寸决策 | `packages/payload/src/uploads/image-resizing/getImageResizeAction.ts` | `getImageResizeAction` |
| 文件名生成 | `packages/payload/src/uploads/image-resizing/generateImageSizeFilename.ts` | `generateImageSizeFilename` |
| 图片裁剪 | `packages/payload/src/uploads/cropImage.ts` | `cropImage` |
| 本地保存 | `packages/payload/src/uploads/saveBufferToFile.ts` | `saveBufferToFile` |
| 上传入口 | `packages/payload/src/uploads/uploadFiles.ts` | `uploadFiles` |
| | | |
| **云端存储插件** | | |
| 插件主逻辑 | `packages/plugin-cloud-storage/src/plugin.ts` | `cloudStoragePlugin` |
| 上传 Hook | `packages/plugin-cloud-storage/src/hooks/afterChange.ts` | `getAfterChangeHook` |
| URL Hook | `packages/plugin-cloud-storage/src/hooks/beforeChange.ts` | `getBeforeChangeHook` |
| 读取 Hook | `packages/plugin-cloud-storage/src/hooks/afterRead.ts` | `getAfterReadHook` |
| 删除 Hook | `packages/plugin-cloud-storage/src/hooks/afterDelete.ts` | `getAfterDeleteHook` |
| 文件收集 | `packages/plugin-cloud-storage/src/utilities/getIncomingFiles.ts` | `getIncomingFiles` |
| 路径计算 | `packages/plugin-cloud-storage/src/utilities/getFileKey.ts` | `getFileKey` |
| 类型定义 | `packages/plugin-cloud-storage/src/types.ts` | `GeneratedAdapter`, `Adapter` |
| | | |
| **S3 适配器** | | |
| 适配器工厂 | `packages/storage-s3/src/adapter.ts` | `createS3Adapter` |
| 插件入口 | `packages/storage-s3/src/index.ts` | `s3Storage` |
| 上传实现 | `packages/storage-s3/src/uploadFile.ts` | `uploadFile` |
| URL 生成 | `packages/storage-s3/src/generateURL.ts` | `generateURL` |
| 文件读取 | `packages/storage-s3/src/getFile.ts` | `getFile` |
| 文件删除 | `packages/storage-s3/src/deleteFile.ts` | `deleteFile` |
| | | |
| **URL 生成** | | |
| 本地 URL | `packages/payload/src/uploads/generateFilePathOrURL.ts` | `generateFilePathOrURL` |

---

## 九、配置示例

### 9.1 完整的 S3 存储配置

```typescript
import { buildConfig } from 'payload'
import { s3Storage } from '@payloadcms/storage-s3'
import sharp from 'sharp'

export default buildConfig({
  serverURL: process.env.PAYLOAD_SERVER_URL,
  collections: [
    {
      slug: 'media',
      upload: {
        // 图片尺寸配置
        imageSizes: [
          {
            name: 'thumbnail',
            width: 400,
            height: 300,
            position: 'centre',
            withoutEnlargement: true,
          },
          {
            name: 'medium',
            width: 1200,
            height: 900,
            fit: 'inside',
            withoutEnlargement: true,
          },
          {
            name: 'hero',
            width: 1920,
            height: 1080,
            fit: 'cover',
            formatOptions: {
              format: 'webp',
              options: { quality: 80 },
            },
          },
        ],
        // 焦点功能
        focalPoint: true,
        // 格式转换
        formatOptions: {
          format: 'webp',
          options: { quality: 85 },
        },
      },
      fields: [
        {
          name: 'alt',
          type: 'text',
          required: true,
        },
      ],
    },
  ],
  plugins: [
    s3Storage({
      bucket: process.env.S3_BUCKET,
      config: {
        credentials: {
          accessKeyId: process.env.S3_ACCESS_KEY_ID,
          secretAccessKey: process.env.S3_SECRET_ACCESS_KEY,
        },
        region: process.env.S3_REGION,
        // 对于 Cloudflare R2:
        // endpoint: `https://${process.env.R2_ACCOUNT_ID}.r2.cloudflarestorage.com`,
      },
      collections: {
        media: {
          prefix: 'media',
          // 自定义 CDN URL 生成
          generateFileURL: ({ filename, prefix }) => {
            return `https://cdn.example.com/${prefix}/${filename}`
          },
          // 或者使用预签名下载
          // signedDownloads: { expiresIn: 3600 },
        },
      },
      acl: 'public-read',
      useCompositePrefixes: true,
    }),
  ],
  sharp,
})
```

### 9.2 带动态前缀的配置

```typescript
// 在集合中添加 prefix 字段，允许用户为每个文件设置路径
{
  slug: 'media',
  upload: true,
  fields: [
    {
      name: 'prefix',
      type: 'text',
      defaultValue: 'uploads',
      admin: {
        description: '文件存储路径前缀',
      },
    },
  ],
}
```

---

## 十、注意事项与最佳实践

### 10.1 性能考虑

1. **大文件处理**: 使用 `useTempFiles` 模式避免内存溢出
2. **并行上传**: `afterChange` 中使用 `Promise.all` 并行上传所有尺寸
3. **分段上传**: S3 适配器对 50MB+ 文件自动使用分段上传
4. **Sharp 缓存**: 同一图片的多个尺寸使用 `sharpBase.clone()` 避免重复解码

### 10.2 安全考虑

1. **文件类型限制**: 使用 `upload.mimeTypes` 限制允许的文件类型
2. **访问控制**: 
   - 默认使用代理模式，可应用 Payload 的访问控制
   - `disablePayloadAccessControl: true` 会绕过访问控制
3. **签名 URL**: 对于私有文件，使用 `signedDownloads` 生成带过期时间的 URL

### 10.3 数据一致性

1. **二次写入**: 云端上传在 `afterChange` 中执行，可能导致：
   - 数据库先写入，云端上传失败 → 数据不一致
   - 建议: 实现自定义错误处理和补偿机制
2. **删除顺序**: 更新操作中先删除旧文件，再上传新文件

### 10.4 URL 策略选择

| 场景 | 推荐策略 | 配置 |
|------|---------|------|
| 需要访问控制 | 代理模式 | 默认行为 |
| 公开文件 + CDN | 直接云端 URL | `disablePayloadAccessControl: true` |
| 私有文件 | 签名 URL | `signedDownloads` |
| 完全自定义 | `generateFileURL` | 实现自定义函数 |

---

*文档生成日期: 2026-05-05*
*基于 PayloadCMS 代码分析*
