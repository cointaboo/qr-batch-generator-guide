# 批量生成二维码并打包 ZIP：容量限制、纠错级别与文件名坑

一次要生成几十上百个二维码时（商品码、活动码、设备码），手工一个一个导出很痛苦。批量做的时候又有几个具体问题：内容太长生成失败、日文中文乱码、ZIP 解压报错、文件名重复被覆盖。

这篇把这些讲清楚。

## QR 码的容量不是无限的

二维码有 40 个"版本"（version 1 到 40），版本越大格子越多：

```
版本  尺寸(模块)   数字容量   字母数字   字节(UTF-8)
1     21×21         41         25         17
10    57×57        652        395        271
20    97×97       2061       1249        858
30    137×137     3993       2420       1663
40    177×177     7089       4296       2953
```

**注意最后一列**：即使最大版本，UTF-8 也只有 **2953 字节**。一个中文字符占 3 字节，所以**最多约 980 个汉字**。

超过就生成失败——不是库的问题，是标准的上限。

解决办法：
- 短链接（用 bit.ly / 自建短链）代替长 URL
- 内容改成分片（多个码指向同一组数据的第 N 片）

## 纠错级别怎么选

四个级别，越高越抗损但容量越小：

| 级别 | 冗余 | 可恢复 | 容量损失 | 适用 |
|---|---|---|---|---|
| L | 7% | 轻微污损 | 0% | 屏幕显示、电子票 |
| M | 15% | 一般污损 | ~15% | **默认推荐** |
| Q | 25% | 明显遮挡 | ~25% | 印刷、可能被折 |
| H | 30% | 大块缺失 | ~30% | logo 遮挡、工业环境 |

**默认用 M**。屏幕展示的用 L 就够（码本身不会坏），要印在物料上、可能被手摸脏的用 Q。

如果要在码中间放 logo，用 H（30% 冗余），logo 面积不超过码面的 20%。

## 生成时的编码问题

**中文乱码**：QR 标准默认用 ISO-8859-1 编码文本。中文内容必须：

1. 生成时明确指定 UTF-8
2. 扫描端也要支持 UTF-8（现代手机都支持，但老扫码枪可能不支持）

```js
// 用 qrcode 库
import QRCode from 'qrcode';
await QRCode.toDataURL('中文内容', {
  errorCorrectionLevel: 'M',
  encoding: 'utf8',      // 关键
  margin: 2,
  width: 512,
});
```

**没有 `encoding: 'utf8'` 时，中文会变成问号或乱码。**

## 批量生成的四个坑

### 1. 文件名重复被覆盖

输入里有两个"商品A"，导出时后一个覆盖前一个。

处理方式：重名自动加序号。

```js
const used = new Set();
function safeName(base) {
  let name = base, i = 2;
  while (used.has(name)) { name = `${base}-${i++}`; }
  used.add(name);
  return name;
}
```

### 2. 文件系统非法字符

`/ \ : * ? " < > |` 在某些系统上是非法的，文件名带这些会导致下载失败或解压报错。

```js
function sanitize(name) {
  return name.replace(/[\/:*?"<>|]/g, '_').replace(/^\.+/, '').slice(0, 120);
}
```

### 3. 内容里的换行/空白

Excel 复制出来的内容常带尾随空格或 `\r`。**二维码会把空格也编码进去**，导致扫出来的内容和预期不一致。

```js
const clean = raw.replace(/[\r\n]+$/, '').trim();
```

### 4. ZIP 结构写错

如果不用库、自己拼 ZIP，最容易错在 local file header：

```
Local file header (30 字节 + 文件名)
  signature      0x04034b50
  version        20
  flags          0x0800（UTF-8 文件名）
  method         0（store，不压缩）
  mtime/mdate
  crc32          数据校验
  compressed size
  uncompressed size
  filename length
  extra length
  [filename]
  [data]

Central directory entry (46 字节 + 文件名)
  signature      0x02014b50
  ...（多一倍的字段）
```

**常见错误**：

- **CRC32 算错**（包括文件名的字节？不，只算数据）→ 解压报 `Bad CRC`
- **偏移量算错** → 报 `Bad magic number for file header`
- **UTF-8 标志位没设**（`0x0800`）→ 中文文件名乱码

手写 ZIP 时，每个 offset 都要精确。用 `zip.js` 或 `fflate` 更省事。

## 用命令行验证生成的 ZIP

```bash
# 完整性（会逐个校验 CRC）
unzip -t out.zip

# 文件列表
unzip -l out.zip

# 单个二维码内容是否正确（需要装 zbar）
zbarimg out/商品A.png
```

`zbarimg` 解码出来的内容必须和输入**逐字节一致**，包括空格。

## 尺寸和边距

```
width       ≥ 300px 保证手机能扫
margin      至少 2 模块（标准要求 4 模块 quiet zone）
```

margin 太小（贴边）会导致部分扫描器识别失败，尤其是印在深色背景上时。

## 免费在线工具

每行一条、批量出图、打包 ZIP，浏览器本地生成：

- https://yuantou.one/tools/qr-batch/

支持「名称|内容」格式、全角竖线、# 注释行，最多 500 条。

---

## 相关阅读

- [PDF 合并拆分：结构原理与浏览器端实现](https://github.com/cointaboo/pdf-merge-split-guide)
- [图片压缩到指定大小：三个旋钮与二分查找](https://github.com/cointaboo/image-compress-guide)
- [字幕格式互转：SRT / VTT / ASS / LRC](https://github.com/cointaboo/subtitle-format-converter)
- [Favicon 完全指南：ICO 与 PNG](https://github.com/cointaboo/favicon-generator)

