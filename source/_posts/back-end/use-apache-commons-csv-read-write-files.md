---
title: 使用apache commons-csv读写csv文件
date: 2025-04-07 20:55:16
tags: [java, commons-csv]
---

# 使用apache commons-csv读写csv文件

## 前言

本文的目的是使用[apache commons-csv](https://commons.apache.org/proper/commons-csv/)读写csv文件。

## 定义实体

Worker是通过ai随机定义的一个java类型:

```java
public class Worker {

    private String id;

    private String employeeNumber;

    private int version;

    private String lastName;

    private String firstName;

    private String gender;

    private String department;

    private String position;

    private LocalDate hireDate;

    private String email;

    private String phoneNumber;

    private String status;

    ...
}
```

可以通过[faker](https://github.com/DiUS/java-faker)随机生成Worker类型的对象。具体实现可以查看下文。

## Apache Commons CSV

Commons CSV 能够读写多种变体的逗号分隔值（CSV）格式文件，由Apache负责开发。

使用maven引入：

```xml
<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-csv</artifactId>
    <version>1.14.0</version>
</dependency>
```

使用groovy gradle引入：

```groovy
implementation 'org.apache.commons:commons-csv:1.14.0'
```

使用kotlin：

```kotlin
implementation("org.apache.commons:commons-csv:1.14.0")
```

### 写csv文件

先创建好文件：

```java
var path = nfsRootPath + File.separator + "workers_" + DateUtils.getYearMonthDay() + ".csv";
var file = new File(path);
if (!file.exists()) {
    file.createNewFile();
}
```

nfsRootPath为`${user.dir}/nfs-root`， DateUtils.getYearMonthDay()表示获取当前年月日（yyyy-MM-dd）。

CSVFormat是`apache commons csv`中用于指定CSV文件解析及写入的格式规范。

`commons csv`内置了12种格式的支持：

1. DEFAULT: 标准逗号分隔值（CSV）格式，遵循[RFC4180规范](https://commons.apache.org/proper/commons-csv/apidocs/org/apache/commons/csv/CSVFormat.html#RFC4180)，但允许空行存在
2. EXCEL: [微软Excel文件格式](https://support.microsoft.com/en-us/office/import-or-export-text-txt-or-csv-files-5250ac4c-663c-47ce-937b-339e391393ba)（使用逗号作为值分隔符）
3. INFORMIX_UNLOAD: [Informix 默认 CSV 导出格式](https://www.ibm.com/docs/en/informix-servers/14.10.0?topic=statements-unload-statement)（通过 UNLOAD TO file_name 操作生成）
4. INFORMIX_UNLOAD_CSV: [Informix 默认 CSV 导出格式](https://www.ibm.com/docs/en/informix-servers/14.10?topic=statements-unload-statement)（通过 UNLOAD TO file_name 操作生成，禁用转义功能）
5. MONGODB_CSV: MongoDB 默认 CSV 格式（通过 mongoexport 命令导出使用）
6. MONGODB_TSV: MongoDB 默认 TSV 格式（通过 mongoexport 命令导出使用）
7. MYSQL:[MySQL](https://dev.mysql.com/doc/refman/8.0/en/mysqldump-delimited-text.html) 默认数据格式（通过 SELECT INTO OUTFILE 和 LOAD DATA INFILE 命令使用）
8. ORACLE: [Oracle](https://docs.oracle.com/en/database/oracle/oracle-database/23/sutil/oracle-sql-loader-control-file-contents.html#GUID-D1762699-8154-40F6-90DE-EFB8EB6A9AB0) 默认数据格式（通过 SQL*Loader 工具使用）
9. POSTGRESQL_CSV: [PostgreSQL默认 CSV 格式](https://www.postgresql.org/docs/current/static/sql-copy.html)（通过 COPY 命令操作使用）
10. POSTGRESQL_TEXT: [PostgreSQL默认文本格式](https://www.postgresql.org/docs/current/static/sql-copy.html)（通过 COPY 命令使用）
11. RFC4180: [RFC 4180](https://tools.ietf.org/html/rfc4180) 标准定义的逗号分隔值（CSV）格式
12. TDF: [制表符分隔格式](https://en.wikipedia.org/wiki/Tab-separated_values)（TDF）

这里使用DEFAULT的csv format：

```java
var headers = new String[] {
    "employee_number",
    "version",
    "last_name",
    "first_name",
    "gender",
    "department",
    "position",
    "hire_date",
    "email",
    "phone_number",
    "status",
};
var csvFormat = CSVFormat.DEFAULT.builder().setHeader(headers).get();
```

利用BufferedWriter和CSVPrinter将Worker对象写入csv文件中：

```java
try (var fw = new FileWriter(path); var bw = new BufferedWriter(fw)) {
    var csvPrinter = new CSVPrinter(bw, csvFormat);
    var format = DateTimeFormatter.ofPattern("yyyy-MM-dd");
    var workers = WorkerMockFactory.createWorkers(100);
    workers.forEach(worker -> {
        try {
            csvPrinter.printRecord(
                worker.getEmployeeNumber(),
                worker.getVersion(),
                worker.getLastName(),
                worker.getFirstName(),
                worker.getGender(),
                worker.getDepartment(),
                worker.getPosition(),
                worker.getHireDate().format(format),
                worker.getEmail(),
                worker.getPhoneNumber(),
                worker.getStatus()
            );
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
    });
} catch (Exception e) {
    e.printStackTrace();
    throw e;
}
```


## Faker

faker移植自Ruby的faker gem（以及Perl的Data::Faker模块），能够生成仿真数据。

针对中国格式的手机号码，可以使用如下代码随机生成：

```java
public class ChinaPhoneNumberGenerator {

    // 中国运营商号段（2023年最新）
    private static final List<String> PREFIXES = Arrays.asList(
            "130", "131", "132", "133", "134", "135", "136", "137", "138", "139",
            "145", "146", "147", "148", "149",
            "150", "151", "152", "153", "155", "156", "157", "158", "159",
            "165", "166", "167", "168", "169",
            "170", "171", "172", "173", "174", "175", "176", "177", "178", "179",
            "180", "181", "182", "183", "184", "185", "186", "187", "188", "189",
            "190", "191", "192", "193", "194", "195", "196", "197", "198", "199"
    );

    // 常见显示格式（随机选择）
    private static final List<String> FORMATS = Arrays.asList(
            "%s%s%s"          // 13800138000
            "%s-%s-%s",        // 138-0013-8000
            "%s %s %s",        // 138 0013 8000
            "+86 %s%s%s",      // +86 13800138000
            "+86-%s-%s-%s",    // +86-138-0013-8000
            "0086 %s%s%s"      // 0086 13800138000
    );

    public static String generate() {
        Faker faker = new Faker(Locale.CHINA);

        // 生成基础号码部分
        String prefix = PREFIXES.get(faker.random().nextInt(PREFIXES.size()));
        String middle = String.format("%04d", faker.number().numberBetween(0, 9999));
        String tail = String.format("%04d", faker.number().numberBetween(0, 9999));

        // 随机选择显示格式
        String format = FORMATS.get(faker.random().nextInt(FORMATS.size()));

        // 生成最终号码
        String phone = String.format(format, prefix, middle, tail);

        // 二次验证确保有效性
        return validate(phone) ? phone : generate();
    }

    // 严格验证手机号逻辑
    private static boolean validate(String phone) {
        // 统一去除格式符号
        String cleanNumber = phone.replaceAll("[+\\- ()]", "");

        // 验证长度和开头
        if (cleanNumber.startsWith("86")) {
            return cleanNumber.length() == 13 && cleanNumber.matches("^861[3-9]\\d{9}$");
        } else if (cleanNumber.startsWith("0086")) {
            return cleanNumber.length() == 14 && cleanNumber.matches("^00861[3-9]\\d{9}$");
        } else {
            return cleanNumber.length() == 11 && cleanNumber.matches("^1[3-9]\\d{9}$");
        }
    }
}
```

随机生成Worker的代码如下：

```java
public class WorkerMockFactory {
    private static final Faker faker = new Faker(Locale.ENGLISH);
    private static final FakeValuesService fakeValues = new FakeValuesService(
            Locale.of("en-US"), new RandomService());

    // 预定义枚举选项
    private static final List<String> GENDERS = Arrays.asList(
            "male", "female", "other", "not_specified");
    private static final List<String> STATUSES = Arrays.asList(
            "active", "resigned", "on_leave", "probation");

    // 部门与职位映射
    private static final List<String> TECH_DEPTS = Arrays.asList(
            "Engineering", "R&D", "DevOps", "QA");
    private static final List<String> TECH_POSITIONS = Arrays.asList(
            "Junior Developer", "Senior Developer", "Tech Lead", "Architect");


    public static Worker createWorker() {
        Worker worker = new Worker();

        // 生成符合业务规则的工号 (示例: WK2024-00123)
        worker.setEmployeeNumber(fakeValues.bothify("WK####-#####").replaceAll("#", "0"));

        // 使用构建者模式设置字段
        worker.setVersion(faker.random().nextInt(1, 5)); // 模拟历史版本
        worker.setLastName(faker.name().lastName());
        worker.setFirstName(faker.name().firstName());
        worker.setGender(GENDERS.get(faker.random().nextInt(GENDERS.size())));
        worker.setDepartment(faker.options().nextElement(TECH_DEPTS));
        worker.setPosition(faker.options().nextElement(TECH_POSITIONS));
        worker.setHireDate(generateHireDate());
        worker.setEmail(generateCorporateEmail(worker));
        worker.setPhoneNumber(ChinaPhoneNumberGenerator.generate()); // 统一号码格式
        worker.setStatus(faker.options().nextElement(STATUSES));

        return worker;
    }

    private static LocalDate generateHireDate() {
        return faker.date().past(365 * 5, TimeUnit.DAYS)
                .toInstant().atZone(ZoneId.systemDefault())
                .toLocalDate();
    }

    private static String generateCorporateEmail(Worker worker) {
        return String.format("%s.%s@%s",
                worker.getFirstName().toLowerCase().replaceAll("'", ""),
                worker.getLastName().toLowerCase().replaceAll("'", ""),
                faker.internet().domainName());
    }

    public static boolean validate(Worker worker) {
        return worker.getEmail().matches("^[a-z.]+@[a-z]+\\.[a-z]{2,}$")
                && worker.getEmployeeNumber().startsWith("WK")
                && worker.getHireDate().isAfter(LocalDate.of(2010, 1, 1));
    }

    // 批量生成方法
    public static List<Worker> createWorkers(int count) {
        return IntStream.range(0, count)
                .mapToObj(i -> createWorker())
                .collect(Collectors.toList());
    }

    // 大量数据（超过10万）批量生成方法
    public static List<Worker> createWorkersParallel(int count) {
        return IntStream.range(0, count)
                .parallel()
                .mapToObj(i -> createWorker())
                .collect(Collectors.toList());
    }
}
```

## 引用

1. [faker repo](https://github.com/DiUS/java-faker)
2. [apache commons csv](https://commons.apache.org/proper/commons-csv/)
3. [commons-csv mvaen repository.](https://mvnrepository.com/artifact/org.apache.commons/commons-csv)