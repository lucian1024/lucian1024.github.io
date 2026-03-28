---
title: 鸿蒙 JSON 序列化方案探索：从 class-transformer 到 TurboTransJSON
date: 2026-03-28
categories: [harmony]
tags: [HarmonyOS, ArkTS, JSON, 序列化]
---

## 背景

在鸿蒙应用开发过程中，网络请求和响应数据均以 JSON 格式传输，需要在 JSON 字符串与 ArkTS 数据结构之间进行相互转换。

ArkTS 原生提供了 `JSON.parse()` / `JSON.stringify()` 方法，但其能力相当有限：

- **不支持自定义 JSON Key**：属性名必须与 JSON 的 Key 完全一致，导致命名无法符合 ArkTS 的驼峰命名规范（后端往往使用下划线分隔的 snake_case）
- **不支持多别名**：实际开发中，同一份数据可能来自不同接口，后端对同一字段使用了不同的 Key 名（此处不点名批评后端 🙂）。原生 JSON 解析无法将多个 Key 映射到同一个属性，只能为每个接口单独定义一个数据类

我们需要的，是类似 Android `@SerializedName` 注解的能力，能够：
1. 自定义属性与 JSON Key 的映射关系
2. 支持多别名，将不同接口定义的不同 Key 统一映射到同一个数据类属性

---

## class-transformer：第一方案

### 引入

在调研过程中，我们在[鸿蒙官方 FAQ 文档](https://developer.huawei.com/consumer/cn/doc/harmonyos-faqs/faqs-arkts-75)中找到了 [class-transformer](https://ohpm.openharmony.cn/#/cn/detail/class-transformer) 这个三方组件，并将其引入项目。

### 原理

class-transformer 是一个基于装饰器（Decorator）的对象序列化/反序列化库，最初来自 TypeScript 生态。其核心原理如下：

**1. 装饰器元数据收集**

通过 `@Expose`、`@Transform`、`@Type` 等装饰器，在类定义阶段将映射关系以元数据（metadata）的形式存储起来。例如：

```typescript
import { Expose, Type, plainToClass } from 'class-transformer';

class Album {
  @Expose({ name: 'album_name' })
  albumName: string = '';
}

class Photo {
  @Expose({ name: 'photo_url' })
  url: string = '';

  @Expose({ name: 'album' })
  @Type(() => Album)
  album: Album = new Album();
}

const photo = plainToClass(Photo, JSON.parse(jsonString));
```

**2. 运行时反射**

在 `plainToClass()` 调用时，class-transformer 根据之前收集的元数据，通过运行时反射将 plain object（普通 JS 对象）的字段映射到目标类的属性上，并完成类型转换和嵌套对象的递归处理。

**3. 序列化**

`classToPlain()` / `instanceToPlain()` 则反向操作，将类实例转成符合指定 Key 名称的 plain object，再交给 `JSON.stringify()` 输出。

### 不支持多别名的问题与修改

class-transformer 官方的 `@Expose({ name: 'key' })` 只接受单个字符串，无法同时匹配多个 Key。为支持多别名，我们对源码进行了修改并单独发布。

**第一步：在 `ExposeOptions` 接口中新增 `alias` 字段**

```typescript
// src/interfaces/decorator-options/expose-options.interface.ts
export interface ExposeOptions {
  /**
   * Name of the property to expose.
   */
  name?: string;

  /**
   * Alias names of property on the target object to expose the value of this property,
   * which is only used when transforming to class object (PLAIN_TO_CLASS).
   */
  alias?: string[];

  // ...
}
```

**第二步：修改 `findExposeMetadataByCustomName`，同时匹配 `name` 和 `alias`**

```typescript
// src/MetadataStorage.ts
findExposeMetadataByCustomName(target: Function, name: string): ExposeMetadata {
  return this.getExposedMetadatas(target).find(metadata => {
    return (
      metadata.options &&
      ((metadata.options.name && metadata.options.name === name) ||
        (metadata.options.alias && metadata.options.alias.indexOf(name) >= 0))
    );
  });
}
```

**第三步：修改 `TransformOperationExecutor`，在 `PLAIN_TO_CLASS` 时将 `alias` 也加入 key 列表**

```typescript
// src/TransformOperationExecutor.ts
if (this.transformationType === TransformationType.PLAIN_TO_CLASS) {
  const tmpExposedProperties: string[] = [];
  exposedProperties.forEach(key => {
    const exposeMetadata = defaultMetadataStorage.findExposeMetadata(target, key);
    if (exposeMetadata && exposeMetadata.options) {
      if (exposeMetadata.options.name) {
        tmpExposedProperties.push(exposeMetadata.options.name);
      }
      if (exposeMetadata.options.alias && exposeMetadata.options.alias.length > 0) {
        tmpExposedProperties.push(...exposeMetadata.options.alias);
      }
    }
    if (tmpExposedProperties.length === 0) {
      tmpExposedProperties.push(key);
    }
  });
  exposedProperties = tmpExposedProperties;
}
```

修改后的使用方式：

```typescript
class SongInfo {
  @Expose({ name: 'song_id', alias: ['songId', 'id'] })
  songId: number = 0;
}

// 无论后端返回 song_id、songId 还是 id，都能正确反序列化到 songId 属性
const song = plainToClass(SongInfo, JSON.parse(jsonString), { excludeExtraneousValues: true });
```

---

## 性能问题：class-transformer 在手表端的瓶颈

接入 class-transformer 后，开发阶段暴露了一个严重的性能问题：**对于回包较大的数据，在手表端反序列化耗时极长**。

原因分析：

1. **运行时反射开销**：class-transformer 在运行时通过反射机制处理元数据，每次反序列化都需要遍历元数据、做类型判断和属性映射，计算量与数据规模线性相关
2. **递归处理嵌套对象**：对于复杂的嵌套数据结构，递归调用层级深，调用栈开销大
3. **手表端性能受限**：相比手机，手表的 CPU 性能和内存资源十分有限，上述开销在手表端被放大，导致耗时明显
4. **JavaScript 引擎限制**：ArkTS 运行在方舟引擎（ArkCompiler）上，运行时反射相比静态代码执行有额外开销

---

## TurboTransJSON：高性能方案

### 发现

在寻找替代方案时，我们在鸿蒙官方文档中找到了 [TurboTransJSON](https://developer.huawei.com/consumer/cn/doc/best-practices/bpta-high-performance-json-parsing)。

> 吐槽一下：最初调研 JSON 序列化方案时，鸿蒙官方文档还没有提供 TurboTransJSON 解决方案，只有 class-transformer 的推荐文档。如果当时就有 TurboTransJSON，也许就不会走那段弯路了。

### 原理

TurboTransJSON 的核心思路与 class-transformer 截然不同：它采用**编译期代码生成**，而不是运行时反射。

具体来说：

1. **ArkTS Transformer 插件**：TurboTransJSON 通过 ArkTS 编译插件（基于 ArkCompiler 的 Transformer 机制），在编译阶段扫描被 `@Serializable` 等装饰器标注的类
2. **生成静态序列化代码**：编译器为每个数据类自动生成对应的序列化/反序列化代码，这些方法是纯粹的字段赋值代码，没有任何运行时反射
3. **零运行时开销**：运行时执行的是编译好的静态代码，直接按字段映射，性能接近手写代码

### 相比 class-transformer 的性能优势

官方提供了详细的性能对比数据（序列化/反序列化吞吐量，单位 MB/s，数值越高越好）：

| 数据集 | class-transformer | TurboTransJSON | 提升幅度 |
|---|---|---|---|
| small (4KB) 序列化 | 6.12 | **48.98** | +700% |
| huge (2765KB) 序列化 | 18.90 | **116.10** | +514% |
| small (4KB) 反序列化 | 20.78 | **62.34** | +200% |
| huge (2765KB) 反序列化 | 140.40 | **207.90** | +48% |

在小数据包场景下，性能提升尤为明显（序列化提升 7 倍），这与编译期生成的静态代码消除了运行时反射开销直接相关。

### TurboTransJSON 的现有缺陷与修复

TurboTransJSON 虽然性能出众，但开箱即用时存在几个缺陷，需要修改源码以满足实际需求。

---

#### 缺陷一：不支持多别名

**问题**：TurboTransJSON 原版不支持为一个属性配置多个 JSON Key 别名，无法处理不同接口返回同一字段但 Key 不同的情况。

**修复**：已向官方仓库提交 PR：[openharmony-sig/turbo_trans#130](https://gitcode.com/openharmony-sig/turbo_trans/pull/130)

**修改思路**：

在 `@SerialName` 装饰器的 `SerialNameOptions` 接口中，新增 `alternate` 字段，用于指定反序列化时的备用 Key 列表：

```typescript
// TurboTransCore/src/main/ets/annotation/SerialName.ets
export interface SerialNameOptions {
  /**
   * The name option.
   */
  name: string;

  /**
   * The alternative names of the field when it is deserialized.
   * 1. This configuration only takes effect when @SerialName is applied to properties
   * 2. Duplicate values are allowed in the alias list of the same property
   * 3. Within the same class, aliases are allowed to duplicate with SerialNameOptions.name of itself or other properties.
   *    When duplicates occur, the key in serialized data only matches the property with the same SerialNameOptions.name.
   * 4. Within the same class, different properties are not allowed to use the same alias,
   *    otherwise ANNOTATION_CONFLICT exception should be thrown
   */
  alternate?: string[];
}
```

使用示例：

```typescript
@Serializable()
class SongInfo {
  // 不同接口可能用 song_id、songId 或 id 表示同一字段
  @SerialName({ name: 'song_id', alternate: ['songId', 'id'] })
  songId: number = 0;
}
```

**运行时解析逻辑**：

在反序列化解码阶段（`PlainJsonDecoding.ets`），先按主 Key（`elementDecodingNames`）匹配，若未找到则遍历别名列表（`elementDecodingAlternate`）：

```typescript
// TurboTransJSON/src/main/ets/implementation/decoding/PlainJsonDecoding.ets
decodeElementIndex(descriptor: SerialDescriptor, key: string, value: ESObject): number {
  let index = descriptor.elementDecodingNames.indexOf(key);
  if (index === -1) {
    // 主 Key 未匹配，尝试从别名列表查找
    index = descriptor.elementDecodingAlternate.findIndex((alternate) => alternate.includes(key));
  }
  if (index === -1) {
    if (this.json.options.ignoreUnknownKeys) {
      return DecodedElementIndex.IGNORED_NAME;
    }
    // ...
  }
  // ...
}
```

**`ObjectUtils` 中新增按别名取值的工具方法**，先按主 Key 取值，取不到再逐个尝试别名：

```typescript
// TurboTransCore/src/main/ets/util/ObjectUtils.ets
export function getPropObject(plain: ESObject, name: string, alternate: string[]): ESObject {
  if (plain === undefined || plain === null) {
    return undefined;
  }

  let value: ESObject = plain[name];
  if (value !== undefined && value !== null) {
    return value;
  }

  for (const key of alternate) {
    value = plain[key];
    if (value !== undefined && value !== null) {
      break;
    }
  }

  return value;
}
```

**编译期插件处理**：在 `SerialDecoratorsAnalyzer.ts` 中，新增对 `alternate` 字段的解析，将别名列表从 AST 中提取出来，写入 `PropertyDecorators`，再由代码生成阶段将其注入生成的序列化代码：

```typescript
// TurboTransJSONPlugin/src/core/Types.ts
export interface PropertyDecorators {
  serialName?: string;       // @SerialName 自定义序列化名称
  alternate?: string[];      // @SerialName 自定义反序列化别名（新增）
  isRequired: boolean;       // @Required 是否必需字段
  isTransient: boolean;      // @Transient 是否跳过序列化
  isPlainValue: boolean;     // @PlainValue 是否使用原始值（跳过类型转换）
  // ...
}
```

---

#### 缺陷二：不支持可选参数

**问题**：TurboTransJSON 原版在反序列化时，若 JSON 中缺少某个字段会报错或赋值异常。而实际业务中，后端接口返回的字段往往是不完整的，某些字段是可选的。

**修复**：已向官方仓库提交 PR：[openharmony-sig/turbo_trans#136](https://gitcode.com/openharmony-sig/turbo_trans/pull/136)

**修改思路**：

通过在 `TypeStructure` 中新增 `isOptional` 标识，编译期插件在解析属性类型时检测是否为可选类型（`T | undefined` 或 `T?`），若为可选则生成 `OptionalSerializer` 处理逻辑，反序列化时对缺失字段跳过赋值而不是报错。

**第一步：`SerializerGenerator` 中为可选类型分配 `OptionalSerializer`**

```typescript
// TurboTransJSONPlugin/src/core/services/CodeGenerationService/generators/SerializerGenerator.ts
if (structure.kind === PropertyKind.GENERIC) {
  return []; // 跳过类和泛型
}
// 新增：可选类型使用 OptionalSerializer
if (structure.isOptional) {
  return ['OptionalSerializer'];
}
const result = new Set<string>();
// ... 原有逻辑
```

**第二步：`HandlebarsTemplateEngine` 中修正 `eqPrimitiveType` 和 `useInstanceSerializer` 判断，排除可选类型**

```typescript
// TurboTransJSONPlugin/src/core/template/HandlebarsTemplateEngine.ts

// eqPrimitiveType：原来只判断 kind，现在同时排除可选类型
this.registerHelper('eqPrimitiveType', (type: TypeStructure) => {
  return this.isPrimitiveType(type.kind) && !type.isOptional;
});

// useInstanceSerializer：原来不判断可选，现在排除可选类型（可选类型走 OptionalSerializer）
this.registerHelper('useInstanceSerializer', (prop: PropertyAnalysis) => {
  return this.isBuildInType(prop.type.kind) && !prop.type.isOptional && !prop.decorators?.with;
});

// getSerializer：同样排除可选类型
if (this.isBuildInType(analysis.type.kind) && !analysis.type.isOptional && !analysis.decorators?.with) {
  return `${analysis.name}Serializer`;
} else {
  return `get${this.toUpperCase(analysis.name)}Serializer()`;
}
```

**第三步：更新 Handlebars 模板，传入完整 `type` 对象（含 `isOptional`）而非仅 `kind` 字符串**

{% raw %}
```handlebars
{{! SerializerTemplate.hbs / SerializerStrictTemplate.hbs }}
{{! 修改前：eqPrimitiveType 只传 analysis.type.kind }}
{{#if (and (eqPrimitiveType analysis.type.kind) (not (hasDecoratorWith analysis.decorators)))}}

{{! 修改后：传入完整 analysis.type 对象，内含 isOptional 标识 }}
{{#if (and (eqPrimitiveType analysis.type) (not (hasDecoratorWith analysis.decorators)))}}
```
{% endraw %}

使用方式不变，ArkTS 原生的 `?` 可选语法即可触发：

```typescript
@Serializable()
class SongInfo {
  songId: number = 0;
  songName: string = '';
  // 可选字段：JSON 中缺失时不报错，保持默认值
  album?: string;
  lyric?: string;
}
```

---

#### 缺陷三：编译速度变慢

**问题**：接入 TurboTransJSON 后，项目编译速度明显下降。

**原因分析**：

1. **编译期代码生成的代价**：TurboTransJSON 通过 ArkTS Transformer 插件在编译阶段扫描并为每个数据类生成序列化代码。项目中数据类数量越多，插件处理的 AST 节点越多，编译耗时越长
2. **增量编译失效**：Transformer 插件可能导致更多文件被标记为"需重新编译"，破坏增量编译的缓存效果，即便只修改了少量文件也可能触发较多文件的重新生成
3. **代码量增加**：编译器为每个数据类生成的序列化方法本身也是代码，增加了后续编译阶段（类型检查、代码优化等）需要处理的代码量

这是使用编译期代码生成方案不可避免的代价，需要在**运行时性能**与**编译速度**之间做取舍。对于手表等性能受限设备，运行时性能更为关键，编译速度的损耗是可以接受的。

---

## 总结

| | 原生 JSON | class-transformer | TurboTransJSON（修改后）|
|---|---|---|---|
| 自定义 Key | ❌ | ✅ | ✅ |
| 多别名 | ❌ | ✅（需改源码）| ✅（已提 PR）|
| 可选参数 | ✅ | ✅ | ✅（已提 PR）|
| 运行时性能 | 高 | 低（运行时反射） | 高（编译期生成）|
| 手表端大包耗时 | — | 明显 | 大幅优化 |
| 编译速度 | 不影响 | 不影响 | 略慢 |

TurboTransJSON 是鸿蒙平台目前最适合生产环境的 JSON 序列化方案，特别是对于性能敏感的穿戴设备场景。原版的多别名和可选参数缺陷已通过修改源码解决，PR 已提交官方仓库等待合并。
