---
title: markdown合并单元格
permalink: markdown-he-bing-dan-yuan-ge
date: 2019-12-18 11:38:06
tags: markdown
categories: util
---

> markdown本身不支持合并单元格, 但是兼容html, html支持该功能

## html实现表格
### 基础用法

```
<table>
    <tr>
        <td>姓名</td>
        <td>年龄</td>
        <td>性别</td>
    </tr>
    <tr>
        <td>杨过</td>
        <td>18</td>
        <td>南</td>
    </tr>
    <tr>
        <td>小龙女</td>
        <td>18</td>
        <td>女</td>
    </tr>
</table>

```

<table>
    <tr>
        <td>姓名</td>
        <td>年龄</td>
        <td>性别</td>
    </tr>
    <tr>
        <td>杨过</td>
        <td>18</td>
        <td>男</td>
    </tr>
    <tr>
        <td>小龙女</td>
        <td>18</td>
        <td>女</td>
    </tr>
</table>

<!--more-->

### 合并行
```
<table>
    <tr>
        <td>姓名</td>
        <td>年龄</td>
        <td>性别</td>
    </tr>
    <tr>
        <td rowspan="2">杨过</td>
        <td>18</td>
        <td>男</td>
    </tr>
    <tr>
        <td>18</td>
        <td>女</td>
    </tr>
</table>
```

<table>
    <tr>
        <td>姓名</td>
        <td>年龄</td>
        <td>性别</td>
    </tr>
    <tr>
        <td rowspan="2">杨过</td>
        <td>18</td>
        <td>男</td>
    </tr>
    <tr>
        <td>18</td>
        <td>女</td>
    </tr>
</table>


### 合并列
```
<table>
    <tr>
        <td>姓名</td>
        <td>年龄</td>
        <td>性别</td>
    </tr>
    <tr>
        <td colspan="2">杨过</td>
        <td>男</td>
    </tr>
    <tr>
        <td>小龙女</td>
        <td>18</td>
        <td>女</td>
    </tr>
</table>
```
<table>
    <tr>
        <td>姓名</td>
        <td>年龄</td>
        <td>性别</td>
    </tr>
    <tr>
        <td colspan="2">杨过</td>
        <td>男</td>
    </tr>
    <tr>
        <td>小龙女</td>
        <td>18</td>
        <td>女</td>
    </tr>
</table>

### 混合使用
```
<table>
    <tr>
        <td>姓名</td>
        <td>年龄</td>
        <td>性别</td>
    </tr>
    <tr>
        <td colspan="2" rowspan="2">杨过</td>
        <td>男</td>
    </tr>
    <tr>
        <td>女</td>
    </tr>
</table>
```
<table>
    <tr>
        <td>姓名</td>
        <td>年龄</td>
        <td>性别</td>
    </tr>
    <tr>
        <td colspan="2" rowspan="2">杨过</td>
        <td>男</td>
    </tr>
    <tr>
        <td>女</td>
    </tr>
</table>

## java读取excel自动生成

### 自己写的小工具
```
package com.learning.demo.util.html;

import org.apache.commons.lang3.StringUtils;
import org.apache.poi.ss.usermodel.*;
import org.apache.poi.ss.util.CellRangeAddress;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.io.File;
import java.io.FileInputStream;
import java.text.DecimalFormat;
import java.util.HashMap;
import java.util.Map;

/**
 * Excel解析处理
 */
public class HtmlExcelUtil {

    private Logger log = LoggerFactory.getLogger(this.getClass());


    private static final String UNION = "table_union_used";

    private Workbook workbook;


    public HtmlExcelUtil(File file) throws Exception {
        workbook = WorkbookFactory.create(new FileInputStream(file));
    }


    public void buildHtmlExcel() {
        TableColumnGroup tableColumnMap = getTableColumnMap();
        soutHtmlTable(tableColumnMap);
    }

    private void soutHtmlTable(TableColumnGroup tableColumnMap) {
        Map<String, TableColumn> map = tableColumnMap.getMap();
        Integer maxCell = tableColumnMap.getMaxCell();
        Integer maxRow = tableColumnMap.getMaxRow();
        StringBuilder sb = new StringBuilder();
        sb.append("<table>");
        for (int y = 0; y <= maxRow; y++) {
            sb.append("<tr>");
            for (int x = 0; x <= maxCell; x++) {
                TableColumn tableColumn = map.get(x + "-" + y);
                if (tableColumn == null) {
                    continue;
                }
                String value = tableColumn.getValue();
                if (!UNION.equals(value)) {
                    sb.append("<td ");
                    if (tableColumn.getHigh() > 1) {
                        sb.append("rowspan=\"");
                        sb.append(tableColumn.getHigh());
                        sb.append("\" ");
                    }
                    if (tableColumn.getWide() > 1) {
                        sb.append("colspan=\"");
                        sb.append(tableColumn.getWide());
                        sb.append("\" ");
                    }

                    sb.append(">");
                    sb.append(value);
                    sb.append("</td>");
                }
            }
            sb.append("</tr>");
        }
        sb.append("</table>");

        System.out.println(sb.toString());
    }

    /**
     * 读取处理excel数据
     *
     * @throws Exception
     */
    public TableColumnGroup getTableColumnMap() {
        try {
            Sheet sheet = workbook.getSheetAt(0);
            Map<String, TableColumn> map = new HashMap<>();
            // 处理合并单元格
            buildUnionTable(sheet, map);

            int lastRowNum = sheet.getLastRowNum();
            short lastCellNum = sheet.getRow(0).getLastCellNum();
            for (int rowIndex = 0; rowIndex <= lastRowNum; rowIndex++) {
                Row row = sheet.getRow(rowIndex);
                for (short cellIndex = 0; cellIndex < lastCellNum; cellIndex++) {

                    Cell cell = row.getCell(cellIndex);
                    String cellValue = (String) getCellValue(cell);
                    String key = cellIndex + "-" + rowIndex;
                    if (map.get(key) == null) {
                        map.put(key, new TableColumn(cellValue));
                    }
                }
            }
            return new TableColumnGroup(map, (int) lastCellNum, lastRowNum);
        } catch (Exception e) {
            log.error("解析Excel异常", e);
        } finally {
            try {
                workbook.close();
            } catch (Exception e) {
                log.error("workbook关闭异常", e);
            }
        }
        return null;
    }

    /**
     * 处理合并单元格
     *
     * @param sheet
     * @param map
     */
    private void buildUnionTable(Sheet sheet, Map<String, TableColumn> map) {
        // 遍历合并区域
        for (int i = 0; i < sheet.getNumMergedRegions(); i++) {
            CellRangeAddress region = sheet.getMergedRegion(i);//
            int firstColumn = region.getFirstColumn();             // 合并区域首列位置
            int lastColumn = region.getLastColumn();
            int firstRow = region.getFirstRow();                     // 合并区域首行位置
            int lastRow = region.getLastRow();
            Row firRow = sheet.getRow(firstRow);
            Cell firCell = firRow.getCell(firstColumn);
            String cellValue = (String) getCellValue(firCell);
            TableColumn tableColumn = new TableColumn(lastColumn - firstColumn + 1, lastRow - firstRow + 1, cellValue);
            map.put(firstColumn + "-" + firstRow, tableColumn);
            // 处理合并单元格
            boolean flag = true;
            for (int y = firstRow; y <= lastRow; y++) {
                for (int x = firstColumn; x <= lastColumn; x++) {
                    if (flag) {
                        flag = false;
                        continue;
                    }
                    TableColumn value = new TableColumn(UNION);
                    map.put(x + "-" + y, value);

                }
            }
        }
    }

    /**
     * 获取cell内容
     *
     * @param cell
     * @return
     */
    private String getCellValue(Cell cell) {
        String value = null;
        if (cell != null) {
            switch (cell.getCellTypeEnum()) {
                case STRING:
                    if (cell.getStringCellValue().trim().equals("'")) {
                        value = "";
                    } else if (cell.getStringCellValue().trim().equals("")) {
                        value = null;
                    } else {
                        value = cell.getStringCellValue();
                    }
                    break;

                case NUMERIC:
                    if (DateUtil.isCellDateFormatted(cell)) {
                        value = cell.getDateCellValue().toString();
                    } else {
                        value = new DecimalFormat("#.######").format(cell.getNumericCellValue());
                    }
                    break;

                case BLANK:
                    value = null;
                    break;

                default:
                    value = null;
            }
        }
        if (StringUtils.isNotEmpty(value)) {
            // 替换多余2个的空格为单个空格,替换空行为单个空格
            value = value.replaceAll("\\s{2,}", " ").replaceAll("(?m)^\\s*$" + System.lineSeparator(), " ");
        }
        return value;
    }


    class TableColumnGroup {
        private Map<String, TableColumn> map;
        private Integer maxCell;
        private Integer maxRow;

        TableColumnGroup(Map<String, TableColumn> map, Integer maxCell, Integer maxRow) {
            this.map = map;
            this.maxCell = maxCell;
            this.maxRow = maxRow;
        }

        Map<String, TableColumn> getMap() {
            return map;
        }

        Integer getMaxCell() {
            return maxCell;
        }

        Integer getMaxRow() {
            return maxRow;
        }
    }


    class TableColumn {
        private Integer wide;
        private Integer high;
        private String value;

        TableColumn(String value) {
            this.high = 1;
            this.wide = 1;
            this.value = value;
        }

        TableColumn(Integer wide, Integer high, String value) {
            this.wide = wide;
            this.high = high;
            this.value = value;
        }

        Integer getHigh() {
            return high;
        }


        Integer getWide() {
            return wide;
        }

        String getValue() {
            return value;
        }
    }


}

```
### 调用
```
package com.learning.demo.util.html;

import java.io.File;

/**
 * Classname：HtmlTableTest
 * Description：TODO
 * date：2019-12-17 16:58
 * author：yuyu
 */
public class HtmlTableTest {

    public static void main(String[] args) throws Exception {
        HtmlExcelUtil htmlExcelUtil = new HtmlExcelUtil(new File("/Users/yuyu/Documents/secooDoc/update-仅修改.xlsx"));

        htmlExcelUtil.buildHtmlExcel();
    }


}

```

# 其他方法
从excel复制到Typora即可, 哈哈哈哈