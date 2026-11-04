
Apache POI
可以使用它在Java程序里对Miscrosoft Office各种文件进行读写操作。
一般情况下，POI都是用于操作Excel文件。

应用场景：
银行网联系统导出交易明细
各种业务系统导出Excel报表
批量导入业务数据

---

## 结合 ReportServiceImpl.exportBussinessData 的 POI 知识点

（代码位置：sky-server `com.sky.service.impl.ReportServiceImpl` 的导出方法）

### 1. 核心类（XSSF 系列 = .xlsx 格式）

| 类 | 作用 | 对应 Excel 概念 |
|---|---|---|
| `XSSFWorkbook` | 工作簿，操作整个 .xlsx 文件的入口 | 整个 Excel 文件 |
| `XSSFSheet` | 工作表 | 文件里的某个 Sheet 标签页 |
| `XSSFRow` | 行 | 一行（行号从 0 开始） |
| `Cell` | 单元格 | 一格，设值用 `setCellValue(...)` |

> 老格式 .xls 用 HSSFWorkbook 系列；.xlsx 用 XSSF 系列。本模板是 .xlsx，所以全用 XSSF。

### 2. 完整导出流程（四步）

**① 从 classpath 读模板文件**

```java
InputStream in = this.getClass().getClassLoader()
        .getResourceAsStream("template/运营数据报表模板.xlsx");
```

- 模板必须放在 `src/main/resources/template/` 下，打包后在 jar 的 classpath 里。
- `getResourceAsStream` 按**相对路径**取文件；**文件不存在时返回 null，不报错**——后面 new 出来直接 NPE，这是最坑的点。

**② 基于模板创建 Excel 对象**

```java
XSSFWorkbook excel = new XSSFWorkbook(in);   // 基于现有文件（模板）创建，不是空簿
XSSFSheet sheet = excel.getSheet("Sheet1");  // 按标签页名字取（不是序号）
```

**③ 往指定单元格写值**

```java
sheet.getRow(1).getCell(1).setCellValue("时间：" + beginDay + "至" + endDay);
XSSFRow row = sheet.getRow(3);
row.getCell(2).setCellValue(businessDataVO.getTurnover());
```

坐标规律：**行号从 0 起**。`getRow(1)` = 第 2 行，`getCell(1)` = 第 2 列（B 列）。
`getRow(n)` 在模板里该行不存在时返回 **null**，调用前要确认模板里这一行是存在的（本模板预置了这些行）。

**④ 通过响应输出流下载**

```java
ServletOutputStream out = response.getOutputStream();
excel.write(out);   // 把整个 Excel 写进输出流 → 浏览器收到文件
out.close();
excel.close();
```

- 参数必须是 **`HttpServletResponse`**（响应对象才有 `getOutputStream()`；`HttpServletRequest` 只有 `getInputStream()`，写错类型直接编译不过——本次就踩过）。
- Controller 的方法参数接收 `HttpServletResponse`，Spring 自动注入当前响应对象。

### 3. 资源关闭

- `excel.close()` / `out.close()` 必须执行，否则文件句柄泄漏。
- 更规范写法是用 try-with-resources（POI 的 XSSFWorkbook 实现了 Closeable）。
- 本代码把 close 放在 try 块内，若 write 中途异常，close 不会执行（模板文件小，实际影响不大）。

### 4. 易错点清单

| 易错点 | 表现 |
|---|---|
| 模板没放进 resources | `in == null` → `new XSSFWorkbook(null)` NPE（500） |
| 参数类型写成 HttpServletRequest | 编译错误：请求对象没有 getOutputStream() |
| getRow 取不到行 | 返回 null → NPE（确认模板里有该行） |
| cell 不存在 | `getCell` 返回 null（模板预置过就不会有） |
| 忘记 close | 句柄泄漏 |
