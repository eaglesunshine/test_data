# 审核标准API文档

版本记录

| 日期       | 版本号 | 变更记录                       | 作者         | 审核人 |
|------------|--------|--------------------------------|--------------|--------|
| 2020.05.19 | v1.0.4 | 新增特征提取、布控、搜索plugin | 陈悠悠       | 焦义奎 |
| 2020.05.13 | v1.0.3 | 结果中增加处理时间             | 朱敏         | 焦义奎 |
| 2020.05.12 | v1.0.2 | 结果中增加任务状态和image_id   | 朱敏         | 焦义奎 |
| 2020.05.11 | v1.0.1 | 输出结果格式调整               | 焦义奎       | 杨欣   |
| 2020.04.30 | v1.0.0 | 基本功能                       | 杨欣/焦义奎  | 焦义奎 |

## 1. 概述

本文档给出了审核服务的基本接口。

## 2. API接口

### 2.1 API版本

版本号的结构：v{major}.{minor}.{patch}, 具体含义如下：

-   major表示主版本号，主版本号之间的接口无兼容性保证

-   minor表示次版本号，次版本号之间接口兼容但参数和结果都可能出现新字段

-   patch修订版本号，修订版本号直接接口完全兼容且参数和结果不变

### 2.2 视频格式

支持文件格式为mp4、ts，支持视频编码为H264。

### 2.3 创建任务

- 请求方式

  HTTP(S) POST

- 请求URI

  /api/v1/tasks

- 请求Header

| 字段名称     | 字段类型 | 可选? | 字段描述           | 版本   |
|--------------|----------|-------|--------------------|--------|
| Content-Type | string   | 必选  | application/json   | v1.0.0 |
| X-REQUEST-ID | string   | 可选  | UUID，用于请求追踪 | v1.0.0 |

- 请求参数

| 字段名称     | 类型   | 可选？ | 字段描述                                                                   | 版本   |
|--------------|--------|--------|----------------------------------------------------------------------------|--------|
| type         | string | 必选   | 值为video或者image                                                         | v1.0.0 |
| url          | string | 必选   | 文件地址，支持HTTP URL 协议前缀：http(s):// 如果type为image，文件格式为tar | v1.0.0 |
| callback_url | string | 可选   | 结果回调URL，结果解析出来后，API通知此URL                                  | v1.0.0 |

- 应答参数

| 字段名称         | 字段类型 | 字段描述     | 版本   |
|------------------|----------|--------------|--------|
| code             | int      | 错误码       | v1.0.0 |
| msg              | string   | 错误信息描述 | v1.0.0 |
| data             | object   |              | v1.0.0 |
| &ensp;\+ task_id | string   | 任务id       | v1.0.0 |

- 结果回调URL参数，视频解析完成后，商汤调用回调接口传回获取解析结果的URL。

| 字段名称                        | 字段类型  | 字段描述                                                | 版本   |
|---------------------------------|----------|----------------------------------------------------------|--------|
| code                            | int      | 错误码                                                   | v1.0.0 |
| msg                             | string   | 错误信息描述                                             | v1.0.0 |
| data                            | object   |                                                          | v1.0.0 |
| &ensp;\+ time_cost              | int      | 处理花费时间，单位为s                                    | v1.0.3 |
| &ensp;\+ status                 | string   | 任务状态： waiting, running, finished、canceled、failed  | v1.0.2 |
| &ensp;\+ task_id                | string   | 任务UUID                                                 | v1.0.0 |
| &ensp;\+ meta                   | object   | 视频的 meta信息                                          | v1.0.1 |
| &ensp;&ensp;\++ first_pts       | int64    | 视频第一帧的pts                                          | v1.0.1 |
| &ensp;&ensp;\++ time_base       | object   | 视频的time_base, 对应ffmpeg的AVRational结构的time_base   | v1.0.1 |
| &ensp;&ensp;&ensp;\+++num       | int      | time_base的分子                                          | v1.0.1 |
| &ensp;&ensp;&ensp;\+++den       | int      | time_base的分母                                          | v1.0.1 |
| &ensp;\+ results_url            | string   | 获取解析结果的URL，失败时为空                            | v1.0.0 |

结果文件格式：

<img src="images/result_format.jpg" style="zoom:80%;" />

每行是一个json对象，json对象中包含若干帧的结果。

- 各行的对象格式为：

| 字段名称 | 字段类型 | 字段描述          | 版本   |
|----------|----------|-------------------|--------|
| results  | object[] | 若干帧的解析结果  | V1.0.0 |

- results数组中对象的定义为：

| 字段名称 | 字段类型 | 字段描述                                                                                             | 版本   |
|----------|----------|------------------------------------------------------------------------------------------------------|--------|
| image_id | string   | 此字段只在type为image时有效，image_id为tar中的文件名                                                 | v1.0.1 |
| pts      | int64    | 此字段只在type为video时有效，为帧在视频中的pts，可以换算成播放时间                                   | v1.0.0 |
| results  | object[] | 帧AI分析结果，由多个AI能力模块聚合输出，object的定义详细参考[2.7 AI结构化Schema](#解析结构化schema)  | v1.0.0 |

### 2.4 查询任务

- 请求方式

  HTTP(S) GET

- 请求URI

  /api/v1/tasks/{task_id}

- 请求参数

| 字段名称 | 字段类型 | 是否可选 | 字段描述 | 版本 |
|----------|----------|----------|----------|------|
| task_id  | string   | 必选     | 任务id   |v1.0.0|

- 应答参数

| 字段名称         | 字段类型 | 字段描述                                                | 版本   |
|------------------|----------|---------------------------------------------------------|--------|
| code             | int      | 错误码                                                  | v1.0.0 |
| msg              | string   | 结果信息                                                | v1.0.0 |
| data             | object   |                                                         | v1.0.0 |
| &ensp;\+ task_id | string   | 任务id                                                  | v1.0.0 |
| &ensp;\+ status  | string   | 任务状态： waiting, running, finished、canceled、failed | v1.0.0 |

### 2.5 取消任务

- 请求方式

  HTTP(S) POST

- 请求URI

  /api/v1/tasks/{task_id}/cancel

- 请求参数

| 字段名称 | 字段类型 | 是否可选 | 字段描述 |
|----------|----------|----------|----------|
| task_id  | string   | 必选     | 任务id   |

- 应答参数

| 字段名称            | 字段类型 | 字段描述       | 版本   |
|---------------------|----------|----------------|--------|
| code                | int      | 错误码         | v1.0.0 |
| msg                 | string   | 错误信息描述   | v1.0.0 |
| data                | object   |                | v1.0.0 |
| &ensp;\+ old_status | string   | 原来的任务状态 | v1.0.0 |

### 2.6 任务结果标签

- 请求方式

  HTTP(S) GET

- 请求URI

  /api/v1/tasks/{task_id}/results

- 请求参数

| 字段名称  | 字段类型  | 是否可选  | 字段描述 |
|-----------|-----------|-----------|----------|
| task_id   | string    | 必选      | 任务id   |

- 应答参数

| 字段名称                    | 字段类型 | 字段描述                                                 | 版本   |
|-----------------------------|----------|----------------------------------------------------------|--------|
| code                        | int      | 错误码                                                   | v1.0.0 |
| msg                         | string   | 错误信息描述                                             | v1.0.0 |
| data                        | object   |                                                          | v1.0.0 |
| &ensp;\+ time_cost          | int      | 处理花费时间，单位为s                                    | v1.0.3 |
| &ensp;\+ status             | string   | 任务状态： waiting, running, finished、canceled、failed  | v1.0.2 |
| &ensp;\+ meta               | object   | 视频的 meta信息                                          | v1.0.1 |
| &ensp;&ensp;\++ first_pts   | int64    | 视频第一帧的pts                                          | v1.0.1 |
| &ensp;&ensp;\++ time_base   | object   | 视频的time_base, 对应ffmpeg的AVRational结构的time_base   | v1.0.1 |
| &ensp;&ensp;&ensp;\+++num   | int      | time_base的分子                                          | v1.0.1 |
| &ensp;&ensp;&ensp;\+++den   | int      | time_base的分母                                          | v1.0.1 |
| &ensp;\+ results_url        | string   | 获取解析结果的URL，失败时为空                            | v1.0.0 |

### 2.7 解析结构化Schema

- 系统支持的module列表如下：

| 模块名称(module)  | 审核类别          | 是否涉及OCR |
|--------------------|------------------|-------------|
| porn               | 鉴黄             | 否          |
| terror             | 暴恐             | 否          |
| child-abuse        | 虐童             | 否          |
| ad                 | 广告             | 是          |
| politics           | 政治敏感         | 是          |
| religion           | 宗教             | 是          |
| contraband         | 违禁品           | 是          |
| ocr                | 文字识别         | 是          |
| face               | 人脸布控         | 否          |
| object             | 刚性物体布控     | 否          |
| image              | 图像布控         | 否          |
| object-feature     | 刚性物体特征提取 | 否          |
| face-feature       | 人脸特征提取     | 否          |
| image-feature      | 图像特征提取     | 否          |
| face-search        | 以脸搜图         | 否          |
| image-search       | 以图搜图         | 否          |

- 解析结构化Schema遵循统一的规范，不同module的统一json schema参考如下：

```json
{ 
  "module": "module name", 
  "tags": [ 
    { 
      "catetgory": [ 
        { 
          "level": 1, 
           "tag":"parent tag name", 
            "type": 0, "probability": 0.0 
        } 
      ], 
      "tag": "leaf tag name", 
      "type": 0, 
      "probability": 0.0,
       "attributes": {} 
    } 
  ] 
} 
```

- Shema具体字段说明如下：

| 字段名称                       | 字段类型 | 字段描述                                                                                                                                           |
|--------------------------------|----------|----------------------------------------------------------------------------------------------------------------------------------------------------|
| module                         | string   | 不同类别的解析任务名称                                                                                                                             |
| tags                           | object[] | 一个解析任务中所有的结构化结果                                                                                                                     |
| &ensp;\+ category              | object[] | 父级标签列表，每一个object表示一层父级标签信息                                                                                                     |
| &ensp;&ensp;\++ level          | int      | 表示父级标签层级，递增排序，表示从顶层至底层的标签树高度                                                                                           |
| &ensp;&ensp;\++ tag            | string   | 父级标签名称                                                                                                                                       |
| &ensp;&ensp;\++ type           | int      | 父级标签类别                                                                                                                                       |
| &ensp;&ensp;\++ probability    | float    | 父级标签置信度                                                                                                                                     |
| &ensp;&ensp;\+ tag             | string   | leaf层标签名称                                                                                                                                     |
| &ensp;\+ type                  | object[] | leaf层标签类别                                                                                                                                     |
| &ensp;\+ probability           | string   | leaf层标签置信度                                                                                                                                   |
| &ensp;\+ attributes            | object   | leaf层标签属性，具体字段参考不同module的结构化示例                                                                                                 |
| &ensp;&ensp;\++ content        | string   | 表示文字内容                                                                                                                                       |
| &ensp;&ensp;\++ vertices       | int[]    | 检测框坐标数组，排列方式为【左上顶点x，左上顶点y，右上顶点x，右上顶点y，右下顶点x，右下顶点y，左下顶点x，左下顶点y】，即为检测框的顺时针顶点坐标值 |

- 关于tags.category.type和tags.tag之间的关系参考下图：

<img src="images/label_relationship.jpg" style="zoom:80%;" />

## 不同module的结构化示例：

### 鉴黄(porn)

```json
{ 
  "module": "porn", 
  "tags": [ 
    { 
      "category": [{"level": 1,"type ": 1}], 
      "type ": 0, 
      "probability": 0.987656721, 
      "attributes": {} 
    } 
  ] 
} 
```

- tags.category.type与tags.type的枚举结果如下：

<table>
	<tr>
		<td align="center">tags.category.type</td>
		<td align="center">tags.type</td>
		<td align="center">标签类别</td>
	</tr>
	<tr>
		<td rowspan="4" align="center">1(性感)</td>
		<td align="center">0</td>
		<td align="center">正常</td>
	</tr>
	<tr>
		<td align="center">1</td>
		<td align="center">性感</td>
	</tr>
	<tr>
		<td align="center">2</td>
		<td align="center">非常性感</td>
	</tr>
	<tr>
		<td align="center">3</td>
		<td align="center">色情</td>
	</tr>
	<tr>
		<td rowspan="3" align="center">2(疑似色情)</td>
		<td align="center">0</td>
		<td align="center">正常</td>
	</tr>
	<tr>
		<td align="center">1</td>
		<td align="center">疑似色情</td>
	</tr>
	<tr>
		<td align="center">2</td>
		<td align="center">色情</td>
	</tr>
	<tr>
		<td rowspan="22" align="center">3(色情)</td>
		<td align="center">0</td>
		<td align="center">一般正常</td>
	</tr>
	<tr>
		<td align="center">1</td>
		<td align="center">卡通正常</td>
	</tr>
	<tr>
		<td align="center">2</td>
		<td align="center">简笔画正常</td>
	</tr>
	<tr>
		<td align="center">3</td>
		<td align="center">性感1</td>
	</tr>
	<tr>
		<td align="center">4</td>
		<td align="center">性感2</td>
	</tr>
	<tr>
		<td align="center">5</td>
		<td align="center">非常性感1</td>
	</tr>
	<tr>
		<td align="center">6</td>
		<td align="center">非常性感2</td>
	</tr>
	<tr>
		<td align="center">7</td>
		<td align="center">非常性感3</td>
	</tr>
	<tr>
		<td align="center">8</td>
		<td align="center">非常性感4</td>
	</tr>
	<tr>
		<td align="center">9</td>
		<td align="center">非常性感5</td>
	</tr>
	<tr>
		<td align="center">10</td>
		<td align="center">非常性感6</td>
	</tr>
	<tr>
		<td align="center">11</td>
		<td align="center">一般色情(成人色情)</td>
	</tr>
	<tr>
		<td align="center">12</td>
		<td align="center">抚摸色情</td>
	</tr>
	<tr>
		<td align="center">13</td>
		<td align="center">SM</td>
	</tr>
	<tr>
		<td align="center">14</td>
		<td align="center">性道具</td>
	</tr>
	<tr>
		<td align="center">15</td>
		<td align="center">卡通色情</td>
	</tr>
	<tr>
		<td align="center">16</td>
		<td align="center">简笔画色情</td>
	</tr>
	<tr>
		<td align="center">17</td>
		<td align="center">雕塑色情</td>
	</tr>
	<tr>
		<td align="center">18</td>
		<td align="center">儿童裸露</td>
	</tr>
	<tr>
		<td align="center">19</td>
		<td align="center">油画色情</td>
	</tr>
	<tr>
		<td align="center">20</td>
		<td align="center">人兽色情</td>
	</tr>
	<tr>
		<td align="center">21</td>
		<td align="center">儿童性行为</td>
	</tr>
</table>


### 暴恐(terror)

```json
{ 
  "module": "terror", 
  "tags": [ 
    { 
      "category": [{"level": 1,"type": 0}], 
      "type": 0, 
      "probability": 0.987656721, 
      "attributes": {} 
    } 
  ] 
} 
```

- tags.category.type与tags.type的枚举结果如下：

<table>
	<tr>
		<td align="center">tags.category.type</td>
		<td align="center">tags.type</td>
		<td align="center">标签类别</td>
	</tr>
	<tr>
		<td rowspan="1" align="center">0(警察部队)</td>
		<td align="center">0</td>
		<td align="center">警察部队</td>
	</tr>
	<tr>
		<td rowspan="7" align="center">1(武器)</td>
		<td align="center">0</td>
		<td align="center">枪支</td>
	</tr>
	<tr>
		<td align="center">1</td>
		<td align="center">坦克/装甲车</td>
	</tr>
	<tr>
		<td align="center">2</td>
		<td align="center">子弹</td>
	</tr>
	<tr>
		<td align="center">3</td>
		<td align="center">导弹</td>
	</tr>
	<tr>
		<td align="center">4</td>
		<td align="center">战斗机</td>
	</tr>
	<tr>
		<td align="center">5</td>
		<td align="center">战舰</td>
	</tr>
	<tr>
		<td align="center">6</td>
		<td align="center">刀具</td>
	</tr>
	<tr>
		<td rowspan="22" align="center">3(台标)</td>
		<td align="center">0</td>
		<td align="center">亚洲台</td>
	</tr>
	<tr>
		<td align="center">1</td>
		<td align="center">翡翠台</td>
	</tr>
	<tr>
		<td align="center">2</td>
		<td align="center">三立电视</td>
	</tr>
	<tr>
		<td align="center">3</td>
		<td align="center">新唐人</td>
	</tr>
	<tr>
		<td align="center">4</td>
		<td align="center">苹果日报</td>
	</tr>
	<tr>
		<td align="center">5</td>
		<td align="center">中视电视</td>
	</tr>
	<tr>
		<td align="center">6</td>
		<td align="center">凤凰卫视</td>
	</tr>
	<tr>
		<td align="center">7</td>
		<td align="center">博讯</td>
	</tr>
	<tr>
		<td align="center">8</td>
		<td align="center">壹电视1</td>
	</tr>
	<tr>
		<td align="center">9</td>
		<td align="center">壹电视2</td>
	</tr>
	<tr>
		<td align="center">10</td>
		<td align="center">壹电视3</td>
	</tr>
	<tr>
		<td align="center">11</td>
		<td align="center">宏视</td>
	</tr>
	<tr>
		<td align="center">12</td>
		<td align="center">明镜火拍</td>
	</tr>
	<tr>
		<td align="center">13</td>
		<td align="center">明视1</td>
	</tr>
	<tr>
		<td align="center">14</td>
		<td align="center">明视2</td>
	</tr>
	<tr>
		<td align="center">15</td>
		<td align="center">美丽日报</td>
	</tr>
	<tr>
		<td align="center">16</td>
		<td align="center">美国之音</td>
	</tr>
	<tr>
		<td align="center">17</td>
		<td align="center">自由亚洲</td>
	</tr>
	<tr>
		<td align="center">18</td>
		<td align="center">阳光卫视</td>
	</tr>
	<tr>
		<td align="center">19</td>
		<td align="center">香港电台</td>
	</tr>
	<tr>
		<td align="center">20</td>
		<td align="center">伊斯兰之声</td>
	</tr>
	<tr>
		<td align="center">21</td>
		<td align="center">半岛电视台</td>
	</tr>
	<tr>
		<td rowspan="18" align="center">4(旗帜)</td>
		<td align="center">0</td>
		<td align="center">isis</td>
	</tr>
	<tr>
		<td align="center">1</td>
		<td align="center">雪山狮子旗</td>
	</tr>
	<tr>
		<td align="center">2</td>
		<td align="center">蓝色星月旗</td>
	</tr>
	<tr>
		<td align="center">3</td>
		<td align="center">青天白日旗</td>
	</tr>
	<tr>
		<td align="center">4</td>
		<td align="center">港英旗</td>
	</tr>
	<tr>
		<td align="center">5</td>
		<td align="center">蒙古青旗</td>
	</tr>
	<tr>
		<td align="center">6</td>
		<td align="center">沪独旗</td>
	</tr>
	<tr>
		<td align="center">7</td>
		<td align="center">神州电影厂</td>
	</tr>
	<tr>
		<td align="center">8</td>
		<td align="center">支联会</td>
	</tr>
	<tr>
		<td align="center">9</td>
		<td align="center">全能神</td>
	</tr>
	<tr>
		<td align="center">10</td>
		<td align="center">法轮功</td>
	</tr>
	<tr>
		<td align="center">11</td>
		<td align="center">纳粹</td>
	</tr>
	<tr>
		<td align="center">12</td>
		<td align="center">绿岛</td>
	</tr>
	<tr>
		<td align="center">13</td>
		<td align="center">神韵</td>
	</tr>
	<tr>
		<td align="center">14</td>
		<td align="center">基地组织</td>
	</tr>
	<tr>
		<td align="center">15</td>
		<td align="center">泰米尔</td>
	</tr>
	<tr>
		<td align="center">16</td>
		<td align="center">哈马斯</td>
	</tr>
	<tr>
		<td align="center">17</td>
		<td align="center">旭日旗</td>
	</tr>
	<tr>
		<td rowspan="1" align="center">6(血腥)</td>
		<td align="center">0</td>
		<td align="center">血腥</td>
	</tr>
	<tr>
		<td rowspan="1" align="center">7(蒙面)</td>
		<td align="center">0</td>
		<td align="center">蒙面</td>
	</tr>
	<tr>
		<td rowspan="2" align="center">8(横幅)</td>
		<td align="center">0</td>
		<td align="center">横幅</td>
	</tr>
	<tr>
		<td align="center">1</td>
		<td align="center">举牌</td>
	</tr>
	<tr>
		<td rowspan="5" align="center">10(爆炸与火灾)</td>
		<td align="center">0</td>
		<td align="center">明火</td>
	</tr>
	<tr>
		<td align="center">1</td>
		<td align="center">烟雾</td>
	</tr>
	<tr>
		<td align="center">2</td>
		<td align="center">灰烬</td>
	</tr>
	<tr>
		<td align="center">3</td>
		<td align="center">爆炸</td>
	</tr>
	<tr>
		<td align="center">4</td>
		<td align="center">火灾场景</td>
	</tr>
	<tr>
		<td rowspan="3" align="center">11(暴乱与游行)</td>
		<td align="center">0</td>
		<td align="center">暴乱</td>
	</tr>
	<tr>
		<td align="center">1</td>
		<td align="center">游行</td>
	</tr>
	<tr>
		<td align="center">2</td>
		<td align="center">人群聚集</td>
	</tr>
	<tr>
		<td rowspan="1" align="center">12(尸体)</td>
		<td align="center">0</td>
		<td align="center">尸体</td>
	</tr>
</table>


### 虐童(child-abuse)

```json
{ 
  "module": "child-abuse", 
  "tags": [
    { 
      "category": [{"level": 1,"type": 1}],
      "type": 0, 
      "probability": 0.987656721, 
      "attributes": {} 
    }
  ] 
} 
```

- tags.category.type与tags.type的枚举结果如下：

<table>
    <tr>
		<td rowspan="2" align="center">1(虐童)</td>
		<td align="center">0</td>
		<td align="center">虐童结果</td>
	</tr>
	<tr>
		<td align="center">1</td>
		<td align="center">虐童行为</td>
	</tr>
</table>

### 广告(ad)

```json
{ 
  "module": "ad", 
  "tags": [ 
    { 
      "category": [{"level": 1,"type": 3}], 
      "type": 4, 
      "probability": 0.987656721, 
      "attributes": { 
        "content": "文字内容", 
        "hit_keywords": ["微信号"] 
      } 
    }, 
    { 
      "category": [{"level": 1,"type": 1}], 
      "type": 0, 
      "probability": 0.987656721, 
      "attributes": {} 
    } 
  ] 
} 
```

- tags.category.type与tags.type的枚举结果如下：

<table>
    <tr>
		<td rowspan="3" align="center">1(扫码)</td>
		<td align="center">0</td>
		<td align="center">二维码</td>
	</tr>
	<tr>
		<td align="center">1</td>
		<td align="center">小程序码</td>
	</tr>
	<tr>
		<td align="center">2</td>
		<td align="center">条形码</td>
	</tr>
	<tr>
		<td rowspan="3" align="center">2(水印)</td>
		<td align="center">3</td>
		<td align="center">水印</td>
	</tr>
	<tr>
		<td rowspan="3" align="center">3(文字广告)</td>
		<td align="center">4</td>
		<td align="center">文字广告 (依赖ocr)</td>
	</tr>
</table>


- 其中当标签类别为文字广告时，tags.attributes的结构说明如下：

| 字段名称     | 字段类型 | 是否可选 | 字段描述      |
|--------------|----------|----------|---------------|
| content      | string   |          | 表示文字内容  |
| hit_keywords | string[] |          | ocr命中关键词 |

### 政治敏感(politics)

```json
{ 
  "module": "politics", 
  "tags": [ 
    { 
      "category": [{ "level": 1,"type": 1}], 
      "type": 7, 
      "probability": 0.98732341, 
      "attributes": { 
        "content": "文字内容", 
        "hit_keywords": ["政治事件"] 
      } 
    }
  ] 
} 
```

- tags.category.type与tags.type的枚举结果如下：

<table>
    <tr>
		<td rowspan="13" align="center">1(政治敏感)</td>
		<td align="center">0</td>
		<td align="center">台独(依赖ocr)</td>
	</tr>
	<tr>
		<td align="center">1</td>
		<td align="center">疆独(依赖ocr)</td>
	</tr>
	<tr>
		<td align="center">2</td>
		<td align="center">藏独(依赖ocr)</td>
	</tr>
	<tr>
		<td align="center">3</td>
		<td align="center">蒙独(依赖ocr)</td>
	</tr>
	<tr>
		<td align="center">4</td>
		<td align="center">港独(依赖ocr)</td>
	</tr>
	<tr>
		<td align="center">5</td>
		<td align="center">文革(依赖ocr)</td>
	</tr>
	<tr>
		<td align="center">6</td>
		<td align="center">大跃进(依赖ocr)</td>
	</tr>
	<tr>
		<td align="center">7</td>
		<td align="center">知青下乡(依赖ocr)</td>
	</tr>
	<tr>
		<td align="center">8</td>
		<td align="center">乱港(依赖ocr)</td>
	</tr>
	<tr>
		<td align="center">9</td>
		<td align="center">913事件(依赖ocr)</td>
	</tr>
	<tr>
		<td align="center">10</td>
		<td align="center">六四学潮(依赖ocr)</td>
	</tr>
	<tr>
		<td align="center">11</td>
		<td align="center">法轮功(依赖ocr)</td>
	</tr>
	<tr>
		<td align="center">12</td>
		<td align="center">丑化领导人</td>
	</tr>
</table>

- 政治敏感中的标签类别都依赖于ocr的结果，tags.attributes的结构说明如下：

| 字段名称     | 字段类型 | 字段描述      |
|--------------|----------|---------------|
| content      | string   | 表示文字内容  |
| hit_keywords | string[] | ocr命中关键词 |

### 宗教(religion)

```json
{ 
  "module": "religion", 
  "tags": [ 
    { 
      "category": [{ "level": 1,"type": 3}], 
      "type": 0, 
      "probability": 0.98732341, 
      "attributes": { 
        "content": "文字内容", 
        "hit_keywords": ["古兰经"] 
      }
    }, 
    { 
      "category": [{"level": 1,"type": 4}], 
      "type": 0, 
      "probability": 0.98732341, 
      "attributes": {} 
    }
  ] 
} 
```

- tags.category.type与tags.type的枚举结果如下：

<table>
    <tr>
		<td rowspan="2" align="center">1(宗教服饰)</td>
		<td align="center">0</td>
		<td align="center">佛教服饰</td>
	</tr>
	<tr>
		<td align="center">1</td>
		<td align="center">吉里巴甫</td>
	</tr>
	<tr>
		<td rowspan="2" align="center">2(宗教建筑)</td>
		<td align="center">0</td>
		<td align="center">基督教建筑</td>
	</tr>
	<tr>
		<td align="center">1</td>
		<td align="center">伊斯兰教建筑</td>
	</tr>
	<tr>
		<td rowspan="3" align="center">3(宗教典籍)</td>
		<td align="center">0</td>
		<td align="center">典籍-圣经(基督教) (依赖ocr)</td>
	</tr>
	<tr>
		<td align="center">1</td>
		<td align="center">文字广告 (依赖ocr)</td>
	</tr>
	<tr>
		<td align="center">2</td>
		<td align="center">典籍-大藏经(佛教) (依赖ocr)</td>
	</tr>
	<tr>
		<td rowspan="6" align="center">4(宗教标志)</td>
		<td align="center">0</td>
		<td align="center">标志/旗帜-基督教</td>
	</tr>
	<tr>
		<td align="center">1</td>
		<td align="center">标志/旗帜-伊斯兰教</td>
	</tr>
	<tr>
		<td align="center">2</td>
		<td align="center">标志/旗帜-印度教</td>
	</tr>
	<tr>
		<td align="center">3</td>
		<td align="center">标志/旗帜-犹太教</td>
	</tr>
	<tr>
		<td align="center">4</td>
		<td align="center">标志/旗帜-佛教</td>
	</tr>
	<tr>
		<td align="center">5</td>
		<td align="center">标志/旗帜-道教</td>
	</tr>
</table>


- 当标签类别依赖ocr时，tags.attributes的结构说明如下：

| 字段名称     | 字段类型 | 是否可选 | 字段描述      |
|--------------|----------|----------|---------------|
| content      | string   |          | 表示文字内容  |
| hit_keywords | string[] |          | ocr命中关键词 |

### 违禁品(contraband)

```json
{ 
  "module": "contraband", 
  "tags": [ 
    { 
      "category": [{ "level": 1,"type": 4}], 
      "type": 0, 
      "probability": 0.98732341,
      "attributes": { 
        "content": "文字内容", 
        "hit_keywords": ["微信号"] 
      }
    },
    { 
      "category": [{"level": 1,"type": 1}],
      "type": 2, 
      "probability": 0.98732341, 
      "attributes": {} 
    } 
  ] 
} 
```

- tags.category.type与tags.type的枚举结果如下：

<table>
    <tr>
		<td rowspan="3" align="center">1(武器)</td>
		<td align="center">0</td>
		<td align="center">武器-枪支</td>
	</tr>
	<tr>
		<td align="center">1</td>
		<td align="center">武器-刀具</td>
	</tr>
	<tr>
		<td align="center">2</td>
		<td align="center">武器-弹药 </td>
	</tr>
	<tr>
		<td rowspan="3" align="center">2(毒害品)</td>
		<td align="center">0</td>
		<td align="center">毒害品-放射性(标志)</td>
	</tr>
	<tr>
		<td align="center">1</td>
		<td align="center">毒害品-剧毒品(标志)</td>
	</tr>
	<tr>
		<td align="center">2</td>
		<td align="center">毒害品-易燃易爆品(标志)</td>
	</tr>
	<tr>
		<td rowspan="3" align="center">3(违禁物品)</td>
		<td align="center">0</td>
		<td align="center">违禁物品-汽油桶</td>
	</tr>
	<tr>
		<td align="center">1</td>
		<td align="center">违禁物品-象牙</td>
	</tr>
	<tr>
		<td align="center">2</td>
		<td align="center">违禁物品-烟花爆竹</td>
	</tr>
	<tr>
		<td rowspan="3" align="center">3(违禁物品)</td>
		<td align="center">0</td>
		<td align="center">违禁物品-六合彩 (依赖ocr)</td>
	</tr>
	<tr>
		<td align="center">1</td>
		<td align="center">违禁物品-毒品 (依赖ocr)</td>
	</tr>
	<tr>
		<td align="center">2</td>
		<td align="center">违禁物品-汽油 (依赖ocr)</td>
	</tr>
	<tr>
		<td rowspan="5" align="center">5(禁品logo)</td>
		<td align="center">0</td>
		<td align="center">违禁logo-小米</td>
	</tr>
	<tr>
		<td align="center">1</td>
		<td align="center">违禁logo-vivo</td>
	</tr>
	<tr>
		<td align="center">2</td>
		<td align="center">违禁logo-三星</td>
	</tr>
	<tr>
		<td align="center">3</td>
		<td align="center">违禁logo-oppo</td>
	</tr>
	<tr>
		<td align="center">4</td>
		<td align="center">违禁logo-苹果</td>
	</tr>
	<tr>
		<td rowspan="2" align="center">6(不文明行为)</td>
		<td align="center">0</td>
		<td align="center">不文明行为-吸烟</td>
	</tr>
	<tr>
		<td align="center">1</td>
		<td align="center">不文明行为-竖中指</td>
	</tr>
</table>

- 当标签类别依赖ocr时，tags.attributes的结构说明如下：

| 字段名称     | 字段类型 | 字段描述      |
|--------------|----------|---------------|
| content      | string   | 表示文字内容  |
| hit_keywords | string[] | ocr命中关键词 |

### 文字识别(ocr)

```json
{ 
  "module": "ocr", 
  "tags": [ 
    { 
      "category": [], 
      "type": 0,
      "probability": 0.98732341, 
      "attributes": { 
        "content": "文字内容", 
        "vertices": [0,0,100,0,100,100,0,100] 
      }
    } 
  ] 
} 
```

- tags中的字段说明如下：

| 字段名称 | 字段类型 | 是否可选 | 字段描述                                                                                                                                            |
|----------|----------|----------|-----------------------------------------------------------------------------------------------------------------------------------------------------|
| type     | int      |          | 表示不同语种：0-中文，1-英文，2-中英文                                                                                                              |
| content  | string   |          | 表示文字内容                                                                                                                                        |
| vertices | int[]    |          | 检测框坐标数组，排列方式为【左上顶点x，左上顶点y，右上顶点x，右上顶点y，右下顶点x，右下顶点y，左下顶点x，左下顶点y】，即为检测框的顺时针顶点坐标值  |

### 人脸布控(face)

```json
{ 
  "module": "face", 
   "tags": [ 
    { 
      "category": [], 
      "tag": "人名", 
      "probability": 0.98732341, 
      "attributes": { 
        "object_id": "1231", 
        "label": "暴恐_塔利班", 
        "search_db": "1,2,3,4", 
        "vertices": [0,0,100,0,100,100,0,100], 
        "extend_info": { 
          "china_race_type": "汉族", 
          "world_race_type": "东亚", 
          "face_type": "漫画脸", 
          "age": 28 
        } 
      } 
    } 
  ] 
} 
```


- attributes具体字段说明：

| tag                         | 人物名称                                                                              |
|-----------------------------|---------------------------------------------------------------------------------------|
| label                       | 人物的标签，默认支持三种label：暴恐人物、涉政人物、宗教人物 <br>配合mdb可自定义label  |
| object_id                   | 库管理中的人物ID                                                                      |
| search_db                   | 过滤采用的人脸搜索库                                                                  |
| vertices                    | 人脸框坐标                                                                            |
| extend_info.china_race_type | 国内人种信息，默认支持三种：汉族、维族、藏族                                          |
| extend_info.world_race_type | 世界人种信息，默认支持5种：东亚、非洲、欧美、南亚、西亚                               |
| extend_info.face_type       | 人脸信息，默认支持三种：背景、真人、漫画                                              |
| extend_info.age             | 人脸年龄信息                                                                          |


### 图像布控(image)

```json
{ 
  "module": "image", 
  "tags": [ 
    { 
      "category": [], 
      "tag": "风景", 
      "probability": 0.98732341, 
      "attributes": { 
        "object_id": "1231", 
        "label": "自定义标签", 
        "search_db": "1,2,3,4" 
      } 
    } 
  ] 
} 
```


- attributes具体字段说明：

| tag       | 图像名称                      |
|-----------|-------------------------------|
| label     | 图像的标签，支持自定义label   |
| object_id | 库管理中的元素ID              |
| search_db | 过滤采用的搜索库              |

### 刚性物体布控(object)

```json
{ 
  "module": "object", 
  "tags": [ 
    { 
      "category": [], 
      "tag": "旗帜", 
      "probability": 0.98732341, 
      "attributes": { 
        "vertices": [0,0,100,0,100,100,0,100], 
        "object_id": "1231", 
        "label": "自定义标签", 
        "search_db": "1,2,3,4" 
      } 
    } 
  ] 
} 
```


- attributes具体字段说明：

| tag       | 刚性物体名称                 |
|-----------|------------------------------|
| label     | 图像的标签，支持自定义label  |
| object_id | 物体的标签                   |
| search_db | 过滤采用的物体搜索库         |
| vertices  | 刚性物体坐标                 |

### 人脸特征提取(face-feature)

```json
{ 
  "module": "face-feature", 
  "tags": [ 
    { 
      "category": [], 
      "feature": "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx", 
      "quality": 0.98732341, 
      "attributes": { 
        "remap_info": [1, 0.90442], 
        "extend_info": { 
          "china_race_type": "汉族", 
          "world_race_type": "东亚", 
          "face_type": "漫画脸", 
          "age": 28 
        } 
      } 
    } 
  ] 
} 
```


- attributes具体字段说明：

| feature                | 人脸特征                   |
|------------------------|----------------------------|
| quality                | 人脸特征质量分数           |
| attributes.remap_info  | 人脸特征携带的remap 信息   |
| attributes.extent_info | 见人脸布控模块             |

### 图像特征提取(image-feature)

```json
{ 
  "module": "image-feature", 
  "tags": [ 
    { 
      "category": [], 
      "feature": "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx", 
      "quality": 0.98732341, 
      "attributes": {} 
    } 
  ] 
} 
```


- attributes具体字段说明：

| feature | 图像特征          |
|---------|------------------|
| quality | 图像特征质量分数   |

### 以脸搜图(face-search)

```json
{ 
  "module": "face-search", 
  "tags": [ 
    { 
      "category": [], 
      "task_id": "10276-276382-26327-26378", 
      "probability": 0.98732341, 
      "attributes": { 
        "vertices": [0,0,100,0,100,100,0,100] 
      } 
    } 
  ] 
} 
```


- attributes具体字段说明：

| task_id             | 命中图片的task id            |
|---------------------|------------------------------|
| probability         | 输入图片与命中图片的相似度   |
| attributes.vertices | 命中图片中的人脸坐标         |

### 以图搜图(image-search)

```json
{ 
  "module": "image-search", 
  "tags": [ 
    { 
      "category": [], 
      "task_id": "10276-276382-26327-26378", 
      "probability": 0.98732341, 
      "attributes": {} 
    } 
  ] 
} 
```


- attributes具体字段说明：

| task_id     | 命中图片的task id           |
|-------------|-----------------------------|
| probability | 输入图片与命中图片的相似度  |

## 3 错误码

- API接口返回的错误码：

| 错误码 | http状态码 | 错误信息              | 备注          |
|--------|------------|-----------------------|---------------|
| 0      | 200        | success               | 成功          |
| 1      | 400        | Bad request           | 参数错误      |
| 2      | 500        | Internal server error | 系统内部错误  |
| 3      | 429        | Too many requests     | 请求过于频繁  |
| 4      | 403        | Permission denied     | 没有权限      |

- 返回给HTTP callback接口的错误码：

| 错误码 | http状态码 | 错误信息                    | 备注           |
|--------|------------|-----------------------------|----------------|
| 5      | NA         | File type not supported     | 文件类型不支持 |
| 6      | NA         | Failed to download the file | 下载文件失败   |
| 7      | NA         | Failed open video           | 视频无法打开   |
