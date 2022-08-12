---
layout:     post
title:      OpenCSV - java utils
subtitle:   utils
date:       2020-11-18
author:     Claire
header-img: img/post-bg-github-cup.jpg
catalog: true
tags:
    - OpenCSV
    - JAVA
    - Utils
---

## OpenCSV 

- 快速实现CSV的导入导出
- 可以通过position方式、header名称注解形式、列名指定形式导入导出CSV文件，自定义类型转换

## 如何读取？

- 以下都通过实体注解形式实现，依托java+springboot

```java
//=======================================Position==================================//
//通过position形式
//实体内的注解，基于position
   @CsvBindByPosition(position = 0)
    @Excel(isWrap = false,name = "样例",orderNum = "1",width = 30)
    private String demo;

 public static <T> List<T> readDataFromLocalPathWithPosition(Class<T> clazz, String filePath) {
        List<T> parseResult = new ArrayList<>();
        if (FileUtils.checkFileExist(filePath)) {
            ColumnPositionMappingStrategy<T> mapper = new ColumnPositionMappingStrategy<>();
            mapper.setType(clazz);
            try {
                CsvToBean<T> csvToBean = new CsvToBeanBuilder<T>(new FileReader(filePath))
                        .withMappingStrategy(mapper)
                        .withSeparator(DEFAULT_BIZ_SEPERATOR)
                        .build();
                parseResult = csvToBean.parse();
            } catch (FileNotFoundException e) {
                log.error("[CsvUtils][readDataFromLocalPathWithPosition] meet error when convert csv to java bean ,file path :", filePath);
                log.error("[CsvUtils][readDataFromLocalPathWithPosition]",e);
            }
        }
        if(CollectionUtils.isNotEmpty(parseResult)){
            return parseResult;
        }
        return Collections.emptyList();
    }

//=======================================Header==================================//
    @CsvCustomBindByName(column = "num",converter = ConvertCsvStringToLong.class)
    @Excel(isWrap = false,name = "人数",orderNum = "4",width = 20)
    private Long num;
    //ConvertCsvStringToLong是自定义的类型转换，仅需要继承AbstractBeanField即可，自行实现类型的转换
    public class ConvertCsvStringToLong<T, I>   extends AbstractBeanField<T, I> {

    @Override
    protected Object convert(String s) throws CsvDataTypeMismatchException, CsvConstraintViolationException {
        return StringUtils.isNotBlank(s)?Long.valueOf(s):0L;
    }
}
//以下读取时指定编码UTF-8，建议读取和写入的时候统一指定编码，避免出现中文乱码
    public static <T> List<T> readDataFromLocalPathWithHeader(Class<T> clazz, String filePath) {
        List<T> parseResult = new ArrayList<>();
        if (FileUtils.checkFileExist(filePath)) {
            HeaderColumnNameMappingStrategy<T> headerColumnNameMappingStrategy = new HeaderColumnNameMappingStrategy<>();
            headerColumnNameMappingStrategy.setType(clazz);
            FileInputStream fin = null;
            InputStreamReader reader = null;
            try {
                fin= new FileInputStream(filePath);
                reader =  new InputStreamReader(fin, StandardCharsets.UTF_8);
                CsvToBean<T> csvToBean = new CsvToBeanBuilder<T>(reader)
                        .withMappingStrategy(headerColumnNameMappingStrategy)
                        .withSeparator(DEFAULT_BIZ_SEPERATOR)
                        .build();
                parseResult = csvToBean.parse();
            } catch (FileNotFoundException e) {
                log.error("[CsvUtils][readDataFromLocalPathWithHeader] meet error when convert csv to java bean ,file path :", filePath);
                log.error("[CsvUtils][readDataFromLocalPathWithHeader]",e);
            }finally {
                try {
                    if (reader != null) {
                        reader.close();
                    }
                    if (fin != null) {
                        fin.close();
                    }
                }catch (Exception e){
                    log.error("[CsvUtils][readDataFromLocalPathWithHeader]fail to close stream for csv",e);
                }
            }
        }
        if(CollectionUtils.isNotEmpty(parseResult)){
            return parseResult;
        }
        return Collections.emptyList();
    }

```

## 如何写入？

- 以下都通过实体注解形式实现，依托java+springboot

```java
 public static<T>void writeDataToCsvFile(List<T> dataList,Class<T> clazz, String finalPath) {
        try {
            Writer writer = new FileWriter(finalPath);
            // 手动加上BOM标识
            writer.write(new String(new byte[] { (byte) 0xEF, (byte) 0xBB, (byte) 0xBF }));

            // 设置显示的顺序
            ColumnPositionMappingStrategy<T> mapper = new ColumnPositionMappingStrategy<T>();
            mapper.setType(clazz);

            // 写表头
            CSVWriter csvWriter = new CSVWriter(writer);

            StatefulBeanToCsv<T> beanToCsv = new StatefulBeanToCsvBuilder<T>(writer)
                    .withMappingStrategy(mapper)
                    .withQuotechar(CSVWriter.NO_QUOTE_CHARACTER)
                    .withSeparator(DEFAULT_BIZ_SEPERATOR)
                    .withEscapechar('\\').build();
            beanToCsv.write(dataList);
            csvWriter.close();
            writer.close();
        } catch (IOException | CsvDataTypeMismatchException | CsvRequiredFieldEmptyException e) {
           log.error("[CsvUtils][readModelDataFromLocalPathWithHeader] fail to write data into csv file ,",e);
        }
       log.info("[CsvUtils][readModelDataFromLocalPathWithHeader]write to csv file successfully ,local path is "+finalPath);
    }
```

## 更多可参考

[csv文件处理——Opencsv](https://www.jianshu.com/p/6414185b2f01)