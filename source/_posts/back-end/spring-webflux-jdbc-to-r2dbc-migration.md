---
title: 在 Spring WebFlux 项目中将 JDBC 迁移到 R2DBC
date: 2026-03-31 22:23:56
tags: [gradle, spring boot, spring webflux, r2dbc]
categories: [后端]
---

# 前言

在最近的项目中，我尝试将原本使用 JDBC 的数据访问迁移到 R2DBC，以充分发挥 Spring WebFlux 的响应式优势。在这个过程中，我记录了迁移的理由、步骤、遇到的坑以及一些遗憾，希望对同样做迁移的同学有所帮助。

# 迁移理由

## 不熟悉 R2DBC

在项目最初阶段，我完全没有接触过 R2DBC，对响应式数据库访问不了解，所以直接使用了 JDBC

## 既然使用 WebFlux，就应该使用非堵塞式语法

Spring WebFlux 的核心优势在于非阻塞、异步。JDBC 是阻塞式的，如果在 WebFlux 中使用，会阻塞 Netty 的 EventLoop，导致性能无法提升。迁移到 R2DBC 可以充分利用响应式编程和事件循环模型

# 迁移步骤示例

## 以 WorkerDao 为例，原先使用 JDBC：

```java
public List<Worker> list() {
    var sql = "SELECT * FROM workers WHERE id = ? deleted_at IS NULL";
    return jdbcTemplate.query(sql, (rs, i) -> {
        var worker = new Worker();
        worker.setId(rs.getString("id"));
        worker.setEmployeeNumber(rs.getString("employee_number"));
        // ... 其他字段
        return worker;
    });
}
```

迁移到 R2DBC 后：

```java
public Flux<Worker> list() {
    var sql = "SELECT * FROM workers WHERE id = :id deleted_at IS NULL";
    return databaseClient.sql(sql)
        .map((row, metadata) -> {
            var worker = new Worker();
            worker.setId(row.get("id", String.class));
            worker.setEmployeeNumber(row.get("employee_number", String.class));
            // ... 其他字段
            return worker;
        })
        .all(); // 这里返回 Flux<Worker>
}
```

## 注意点：

1. Flux 替代了 List，Mono 替代了单个返回值。
2. 不要在 subscribe() 中直接做阻塞操作，否则会堵塞 Netty 的 EventLoop。

## 处理分页

原来的 JDBC：

```java
public List<Worker> list(Pageable page) {
    var sql = "SELECT * FROM workers ORDER BY updated_at DESC LIMIT ? OFFSET ?";
    return jdbcTemplate.query(sql, (rs, i) -> mapRow(rs), page.limit(), page.offset());
}
```

R2DBC 写法：

```java
public Flux<Worker> list(Pageable page) {
    var sql = "SELECT * FROM workers ORDER BY updated_at DESC LIMIT :limit OFFSET :offset";
    return databaseClient.sql(sql)
        .bind("limit", page.limit())
        .bind("offset", page.offset())
        .map((row, meta) -> mapRow(row))
        .all();
}
```

## 计数示例

```java
 public Mono<Long> count() {
    var sql = "select count(*) as cnt from workers where deleted_at is null";
    return databaseClient.sql(sql)
            .map((row, meta) -> {
                Number n = row.get("cnt", Number.class);
                return n != null ? n.longValue() : 0L;
            })
            .one();
}
```

# 遇到的问题和解决方案

## 1. Quartz 仍然需要 JDBC

错误示例：

```bash
org.quartz.SchedulerConfigException: No local DataSource found for configuration - 'dataSource' property must be set on SchedulerFactoryBean
```

解决方案：手动配置一个 JDBC DataSource Bean 给 Quartz：

```java
@Bean
public DataSource quartzDataSource(
            @Value("${spring.datasource.url}") String url,
            @Value("${spring.datasource.username}") String username,
            @Value("${spring.datasource.password}") String password
    ) {
        var ds = new HikariDataSource();
        ds.setJdbcUrl(url);
        ds.setUsername(username);
        ds.setPassword(password);
        ds.setDriverClassName("org.postgresql.Driver");
        ds.setMaximumPoolSize(10);
        return ds;
    }


@Bean
public SchedulerFactoryBean schedulerFactoryBean(DataSource quartzDataSource) {
    SchedulerFactoryBean factory = new SchedulerFactoryBean();
    factory.setDataSource(quartzDataSource);
    factory.setOverwriteExistingJobs(true);
    factory.setStartupDelay(5);
    return factory;
}
```

## 2. WebFlux 中阻塞操作

在迁移过程中，如果使用了 jdbcTemplate.query() 或 subscribe() 内部有阻塞方法，可能会导致 Netty EventLoop 堵塞。

解决方案：尽量把 JDBC 调用封装成`Mono.fromCallable(...).subscribeOn(Schedulers.boundedElastic())`，或者直接迁移到 R2DBC。

## 3. 自动配置冲突

当同时引入 spring-boot-starter-jdbc 和 spring-boot-starter-data-r2dbc 时，Spring Boot 的自动配置可能不会自动创建 DataSource，导致 Quartz 启动报错。

解决方案：手动创建 DataSource Bean（如上），明确指定 JDBC 的配置。

# 遗憾

- 虽然业务代码可以完全迁移到 R2DBC，但 Quartz 仍然依赖 JDBC，导致项目中不能完全删除 JDBC 依赖。
- 如果未来需要完全响应式的调度方案，可能要考虑其他 Quartz 替代方案或者等待 Quartz 支持 R2DBC。

# 总结

通过这次迁移，我收获了以下几点：

1. 了解了 R2DBC 的使用方式和响应式数据库访问思路。
2. 熟悉了 Flux/Mono 在 DAO 层的应用，以及与 Pageable、计数等配合使用的方法。
3. 意识到在 WebFlux 项目中，阻塞操作必须谨慎处理。
4. 学会了在 Spring Boot 中 手动创建 DataSource Bean 来解决 Quartz 的兼容性问题。

虽然 Quartz 还依赖 JDBC，但业务层已经实现了完全非阻塞的访问，这是向 WebFlux 响应式世界迈出的第一步。