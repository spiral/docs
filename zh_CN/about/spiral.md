# Spiral - 高性能 PHP/Go 开发框架
[![Latest Stable Version](https://poser.pugx.org/spiral/framework/version)](https://packagist.org/packages/spiral/framework)
[![Build Status](https://travis-ci.org/spiral/framework.svg?branch=master)](https://travis-ci.org/spiral/framework)
[![Codecov](https://codecov.io/gh/spiral/framework/graph/badge.svg)](https://codecov.io/gh/spiral/framework)
[![Scrutinizer Code Quality](https://scrutinizer-ci.com/g/spiral/framework/badges/quality-score.png)](https://scrutinizer-ci.com/g/spiral/framework/?branch=master)

<img src="https://user-images.githubusercontent.com/796136/67560465-9d827780-f723-11e9-91ac-9b2fafb027f2.png" height="135px" alt="Spiral Framework" align="left"/>

Spiral Framework 让 PHP 开发重新令人振奋。它利用 PHP 的快速，易部署特性来实现快速开发业务逻辑，同时独特地借助 Golang 来优雅地构建起原生支持 HTTP/2, GRPC, Queue 等特性的基础架构层。Spiral Framework 非常灵活且完全遵守 PSR 规范。开发者在用它构建更快、更高效的应用时一定会感到非常愉悦。

[官网](https://spiral-framework.com) | [WEB 项目框架](https://github.com/spiral/app) ([命令行项目框架](https://github.com/spiral/app-cli), [ GRPC 项目框架](https://github.com/spiral/app-grpc)) | [**框架文档**](https://github.com/spiral/guide) | [Twitter](https://twitter.com/spiralphp) | [更新日志](/CHANGELOG.md) | [贡献指引](https://github.com/spiral/guide/blob/master/contributing.md)
<br/>

## 功能特性
- 从 2013 年以来经历了充分的实战检验
- [超高性能的 PHP 全栈开发框架](https://www.techempower.com/benchmarks/#section=test&runid=61b3287c-3afc-4c7a-85bb-4cd48df51e7a&hw=ph&test=fortune&l=zik073-v&c=6&d=4&o=e)
- 遵守 PSR-{2,3,4,6,7,11,15,16,17}
- 强大的 [应用服务器](https://roadrunner.dev/), 常驻内存式应用内核
- 原生支持队列（ AMQP, SQS, Beanstalk ）和 PHP 后台工作进程
- GRPC 服务和客户端
- Pub/Sub, WebSocket broadcasting
- HTTPS, HTTP/2+Push, 加密 cookies, sessions, CSRF 防护
- 支持 MySQL, MariaDB, SQLite, Postgres, SQLServer, 自动数据迁移（migration)
- 未来 25 年你都要用的 [ORM](https://github.com/cycle/orm)
- 直观的脚手架和原型（它确实可以帮您编写代码）
- 基于静态分析的有用的类型发现
- 身份验证, 基于角色的访问控制(RBAC), 验证, 以及加密
- 可创建自定义 HTML 标签的动态模板引擎（你也可以使用原生 PHP 模板或者 Twig）
- MVC, HMVC, CQRS, Queue-oriented, RPC-oriented, 命令行程序... 任何类型的应用

## 项目框架
| 应用类型 | 当前状态 | 项目地址       
| ---       | --- | ---
spiral/app | [![Latest Stable Version](https://poser.pugx.org/spiral/app/version)](https://packagist.org/packages/spiral/app) | https://github.com/spiral/app
spiral/app-cli | [![Latest Stable Version](https://poser.pugx.org/spiral/app-cli/version)](https://packagist.org/packages/spiral/app-cli) | https://github.com/spiral/app-cli
spiral/app-grpc | [![Latest Stable Version](https://poser.pugx.org/spiral/app-grpc/version)](https://packagist.org/packages/spiral/app-grpc) | https://github.com/spiral/app-grpc

## 核心组件
| 组件 | 当前状态        
| ---       | ---
spiral/core | [![Latest Stable Version](https://poser.pugx.org/spiral/core/version)](https://packagist.org/packages/spiral/core) [![Build Status](https://travis-ci.org/spiral/core.svg?branch=master)](https://travis-ci.org/spiral/core) [![Codecov](https://codecov.io/gh/spiral/core/branch/master/graph/badge.svg)](https://codecov.io/gh/spiral/core/)
spiral/boot | [![Latest Stable Version](https://poser.pugx.org/spiral/boot/version)](https://packagist.org/packages/spiral/boot) [![Build Status](https://travis-ci.org/spiral/boot.svg?branch=master)](https://travis-ci.org/spiral/boot) [![Codecov](https://codecov.io/gh/spiral/boot/branch/master/graph/badge.svg)](https://codecov.io/gh/spiral/boot/)
spiral/config | [![Latest Stable Version](https://poser.pugx.org/spiral/config/version)](https://packagist.org/packages/spiral/config) [![Build Status](https://travis-ci.org/spiral/config.svg?branch=master)](https://travis-ci.org/spiral/config) [![Codecov](https://codecov.io/gh/spiral/config/branch/master/graph/badge.svg)](https://codecov.io/gh/spiral/config/)

## 任务调度组件
| 组件 | 当前状态        
| ---       | ---
spiral/http | [![Latest Stable Version](https://poser.pugx.org/spiral/http/version)](https://packagist.org/packages/spiral/http) [![Build Status](https://travis-ci.org/spiral/http.svg?branch=master)](https://travis-ci.org/spiral/http) [![Codecov](https://codecov.io/gh/spiral/http/branch/master/graph/badge.svg)](https://codecov.io/gh/spiral/http/)
spiral/console | [![Latest Stable Version](https://poser.pugx.org/spiral/console/version)](https://packagist.org/packages/spiral/console) [![Build Status](https://travis-ci.org/spiral/console.svg?branch=master)](https://travis-ci.org/spiral/console) [![Codecov](https://codecov.io/gh/spiral/console/branch/master/graph/badge.svg)](https://codecov.io/gh/spiral/console/)
spiral/roadrunner | [![Latest Stable Version](https://poser.pugx.org/spiral/roadrunner/version)](https://packagist.org/packages/spiral/roadrunner) [![Build Status](https://travis-ci.org/spiral/roadrunner.svg?branch=master)](https://travis-ci.org/spiral/roadrunner) [![Codecov](https://codecov.io/gh/spiral/roadrunner/branch/master/graph/badge.svg)](https://codecov.io/gh/spiral/roadrunner/)
spiral/jobs | [![Latest Stable Version](https://poser.pugx.org/spiral/jobs/version)](https://packagist.org/packages/spiral/jobs) [![Build Status](https://travis-ci.org/spiral/jobs.svg?branch=master)](https://travis-ci.org/spiral/jobs) [![Codecov](https://codecov.io/gh/spiral/jobs/branch/master/graph/badge.svg)](https://codecov.io/gh/spiral/jobs/)
spiral/php-grpc | [![Latest Stable Version](https://poser.pugx.org/spiral/php-grpc/version)](https://packagist.org/packages/spiral/php-grpc) [![Build Status](https://travis-ci.org/spiral/php-grpc.svg?branch=master)](https://travis-ci.org/spiral/php-grpc) [![Codecov](https://codecov.io/gh/spiral/php-grpc/branch/master/graph/badge.svg)](https://codecov.io/gh/spiral/php-grpc/)

## HTTP 组件
| 组件 | 当前状态        
| ---       | ---
spiral/cookies | [![Latest Stable Version](https://poser.pugx.org/spiral/cookies/version)](https://packagist.org/packages/spiral/cookies) [![Build Status](https://travis-ci.org/spiral/cookies.svg?branch=master)](https://travis-ci.org/spiral/cookies) [![Codecov](https://codecov.io/gh/spiral/cookies/branch/master/graph/badge.svg)](https://codecov.io/gh/spiral/cookies/)
spiral/csrf | [![Latest Stable Version](https://poser.pugx.org/spiral/csrf/version)](https://packagist.org/packages/spiral/csrf) [![Build Status](https://travis-ci.org/spiral/csrf.svg?branch=master)](https://travis-ci.org/spiral/csrf) [![Codecov](https://codecov.io/gh/spiral/csrf/branch/master/graph/badge.svg)](https://codecov.io/gh/spiral/csrf/)
spiral/hmvc | [![Latest Stable Version](https://poser.pugx.org/spiral/hmvc/version)](https://packagist.org/packages/spiral/hmvc) [![Build Status](https://travis-ci.org/spiral/hmvc.svg?branch=master)](https://travis-ci.org/spiral/hmvc) [![Codecov](https://codecov.io/gh/spiral/hmvc/branch/master/graph/badge.svg)](https://codecov.io/gh/spiral/hmvc/)
spiral/router | [![Latest Stable Version](https://poser.pugx.org/spiral/router/version)](https://packagist.org/packages/spiral/router) [![Build Status](https://travis-ci.org/spiral/router.svg?branch=master)](https://travis-ci.org/spiral/router) [![Codecov](https://codecov.io/gh/spiral/router/branch/master/graph/badge.svg)](https://codecov.io/gh/spiral/router/)
spiral/session | [![Latest Stable Version](https://poser.pugx.org/spiral/session/version)](https://packagist.org/packages/spiral/session) [![Build Status](https://travis-ci.org/spiral/session.svg?branch=master)](https://travis-ci.org/spiral/session) [![Codecov](https://codecov.io/gh/spiral/session/branch/master/graph/badge.svg)](https://codecov.io/gh/spiral/session/)
spiral/nyholm-bridge | [![Latest Stable Version](https://poser.pugx.org/spiral/nyholm-bridge/version)](https://packagist.org/packages/spiral/nyholm-bridge) [![Build Status](https://travis-ci.org/spiral/nyholm-bridge.svg?branch=master)](https://travis-ci.org/spiral/nyholm-bridge) [![Codecov](https://codecov.io/gh/spiral/nyholm-bridge/branch/master/graph/badge.svg)](https://codecov.io/gh/spiral/nyholm-bridge/)
spiral/auth-http | [![Latest Stable Version](https://poser.pugx.org/spiral/auth-http/version)](https://packagist.org/packages/spiral/auth-http) [![Build Status](https://travis-ci.org/spiral/auth-http.svg?branch=master)](https://travis-ci.org/spiral/auth-http) [![Codecov](https://codecov.io/gh/spiral/auth-http/branch/master/graph/badge.svg)](https://codecov.io/gh/spiral/auth-http/)

## 安全和验证组件
| 组件 | 当前状态        
| ---       | ---
spiral/encryption | [![Latest Stable Version](https://poser.pugx.org/spiral/encrypter/version)](https://packagist.org/packages/spiral/encrypter) [![Build Status](https://travis-ci.org/spiral/encrypter.svg?branch=master)](https://travis-ci.org/spiral/encrypter) [![Codecov](https://codecov.io/gh/spiral/encrypter/branch/master/graph/badge.svg)](https://codecov.io/gh/spiral/encrypter/)
spiral/security | [![Latest Stable Version](https://poser.pugx.org/spiral/security/version)](https://packagist.org/packages/spiral/security) [![Build Status](https://travis-ci.org/spiral/security.svg?branch=master)](https://travis-ci.org/spiral/security) [![Codecov](https://codecov.io/gh/spiral/security/branch/master/graph/badge.svg)](https://codecov.io/gh/spiral/security/)
spiral/validation | [![Latest Stable Version](https://poser.pugx.org/spiral/validation/version)](https://packagist.org/packages/spiral/validation) [![Build Status](https://travis-ci.org/spiral/validation.svg?branch=master)](https://travis-ci.org/spiral/validation) [![Codecov](https://codecov.io/gh/spiral/validation/branch/master/graph/badge.svg)](https://codecov.io/gh/spiral/validation/)
spiral/filters | [![Latest Stable Version](https://poser.pugx.org/spiral/filters/version)](https://packagist.org/packages/spiral/filters) [![Build Status](https://travis-ci.org/spiral/filters.svg?branch=master)](https://travis-ci.org/spiral/filters) [![Codecov](https://codecov.io/gh/spiral/filters/branch/master/graph/badge.svg)](https://codecov.io/gh/spiral/filters/)
spiral/auth | [![Latest Stable Version](https://poser.pugx.org/spiral/auth/version)](https://packagist.org/packages/spiral/auth) [![Build Status](https://travis-ci.org/spiral/auth.svg?branch=master)](https://travis-ci.org/spiral/auth) [![Codecov](https://codecov.io/gh/spiral/auth/branch/master/graph/badge.svg)](https://codecov.io/gh/spiral/auth/)

## 数据库组件
| 组件 | 当前状态        
| ---       | ---
spiral/database | [![Latest Stable Version](https://poser.pugx.org/spiral/database/version)](https://packagist.org/packages/spiral/database) [![Build Status](https://travis-ci.org/spiral/database.svg?branch=master)](https://travis-ci.org/spiral/database) [![Codecov](https://codecov.io/gh/spiral/database/branch/master/graph/badge.svg)](https://codecov.io/gh/spiral/database/)
spiral/migrations | [![Latest Stable Version](https://poser.pugx.org/spiral/migrations/version)](https://packagist.org/packages/spiral/migrations) [![Build Status](https://travis-ci.org/spiral/migrations.svg?branch=master)](https://travis-ci.org/spiral/migrations) [![Codecov](https://codecov.io/gh/spiral/migrations/branch/master/graph/badge.svg)](https://codecov.io/gh/spiral/migrations/)

## Cycle ORM
| 组件 | 当前状态        
| ---       | ---
cycle/orm   | [![Latest Stable Version](https://poser.pugx.org/cycle/orm/version)](https://packagist.org/packages/cycle/orm) [![Build Status](https://travis-ci.org/cycle/orm.svg?branch=master)](https://travis-ci.org/cycle/orm) [![Scrutinizer Code Quality](https://scrutinizer-ci.com/g/cycle/orm/badges/quality-score.png?b=master)](https://scrutinizer-ci.com/g/cycle/orm/?branch=master) [![Codecov](https://codecov.io/gh/cycle/orm/graph/badge.svg)](https://codecov.io/gh/cycle/orm)
cycle/schema-builder | [![Latest Stable Version](https://poser.pugx.org/cycle/schema-builder/version)](https://packagist.org/packages/cycle/schema-builder) [![Build Status](https://travis-ci.org/cycle/schema-builder.svg?branch=master)](https://travis-ci.org/cycle/schema-builder) [![Scrutinizer Code Quality](https://scrutinizer-ci.com/g/cycle/schema-builder/badges/quality-score.png?b=master)](https://scrutinizer-ci.com/g/cycle/schema-builder/?branch=master) [![Codecov](https://codecov.io/gh/cycle/schema-builder/graph/badge.svg)](https://codecov.io/gh/cycle/schema-builder)
cycle/annotated | [![Latest Stable Version](https://poser.pugx.org/cycle/annotated/version)](https://packagist.org/packages/cycle/annotated) [![Build Status](https://travis-ci.org/cycle/annotated.svg?branch=master)](https://travis-ci.org/cycle/annotated) [![Scrutinizer Code Quality](https://scrutinizer-ci.com/g/cycle/annotated/badges/quality-score.png?b=master)](https://scrutinizer-ci.com/g/cycle/annotated/?branch=master) [![Codecov](https://codecov.io/gh/cycle/annotated/graph/badge.svg)](https://codecov.io/gh/cycle/annotated)
cycle/proxy-factory | [![Latest Stable Version](https://poser.pugx.org/cycle/proxy-factory/version)](https://packagist.org/packages/cycle/proxy-factory) [![Build Status](https://travis-ci.org/cycle/proxy-factory.svg?branch=master)](https://travis-ci.org/cycle/proxy-factory) [![Scrutinizer Code Quality](https://scrutinizer-ci.com/g/cycle/proxy-factory/badges/quality-score.png?b=master)](https://scrutinizer-ci.com/g/cycle/proxy-factory/?branch=master) [![Codecov](https://codecov.io/gh/cycle/proxy-factory/graph/badge.svg)](https://codecov.io/gh/cycle/proxy-factory)
cycle/migrations | [![Latest Stable Version](https://poser.pugx.org/cycle/migrations/version)](https://packagist.org/packages/cycle/migrations) [![Build Status](https://travis-ci.org/cycle/migrations.svg?branch=master)](https://travis-ci.org/cycle/migrations) [![Scrutinizer Code Quality](https://scrutinizer-ci.com/g/cycle/migrations/badges/quality-score.png?b=master)](https://scrutinizer-ci.com/g/cycle/migrations/?branch=master) [![Codecov](https://codecov.io/gh/cycle/migrations/graph/badge.svg)](https://codecov.io/gh/cycle/migrations)

## Stempler 模板引擎
| 组件 | 当前状态        
| ---       |  ---
spiral/stempler | [![Latest Stable Version](https://poser.pugx.org/spiral/stempler/version)](https://packagist.org/packages/spiral/stempler) [![Build Status](https://travis-ci.org/spiral/stempler.svg?branch=master)](https://travis-ci.org/spiral/stempler) [![Codecov](https://codecov.io/gh/spiral/stempler/branch/master/graph/badge.svg)](https://codecov.io/gh/spiral/stempler/) 
spiral/stempler-bridge | [![Latest Stable Version](https://poser.pugx.org/spiral/stempler-bridge/version)](https://packagist.org/packages/spiral/stempler-bridge) [![Build Status](https://travis-ci.org/spiral/stempler-bridge.svg?branch=master)](https://travis-ci.org/spiral/stempler-bridge) [![Codecov](https://codecov.io/gh/spiral/stempler-bridge/branch/master/graph/badge.svg)](https://codecov.io/gh/spiral/stempler-bridge/)

## 功能组件
| 组件 | 当前状态        
| ---       |  ---
spiral/exceptions | [![Latest Stable Version](https://poser.pugx.org/spiral/exceptions/version)](https://packagist.org/packages/spiral/exceptions) [![Build Status](https://travis-ci.org/spiral/exceptions.svg?branch=master)](https://travis-ci.org/spiral/exceptions) [![Codecov](https://codecov.io/gh/spiral/exceptions/branch/master/graph/badge.svg)](https://codecov.io/gh/spiral/exceptions/)
spiral/pagination | [![Latest Stable Version](https://poser.pugx.org/spiral/pagination/version)](https://packagist.org/packages/spiral/pagination) [![Build Status](https://travis-ci.org/spiral/pagination.svg?branch=master)](https://travis-ci.org/spiral/pagination) [![Codecov](https://codecov.io/gh/spiral/pagination/branch/master/graph/badge.svg)](https://codecov.io/gh/spiral/pagination/)
spiral/files | [![Latest Stable Version](https://poser.pugx.org/spiral/files/version)](https://packagist.org/packages/spiral/files) [![Build Status](https://travis-ci.org/spiral/files.svg?branch=master)](https://travis-ci.org/spiral/files) [![Codecov](https://codecov.io/gh/spiral/files/branch/master/graph/badge.svg)](https://codecov.io/gh/spiral/files/)
spiral/streams | [![Latest Stable Version](https://poser.pugx.org/spiral/streams/version)](https://packagist.org/packages/spiral/streams) [![Build Status](https://travis-ci.org/spiral/streams.svg?branch=master)](https://travis-ci.org/spiral/streams) [![Codecov](https://codecov.io/gh/spiral/streams/branch/master/graph/badge.svg)](https://codecov.io/gh/spiral/streams/)
spiral/logger | [![Latest Stable Version](https://poser.pugx.org/spiral/logger/version)](https://packagist.org/packages/spiral/logger) [![Build Status](https://travis-ci.org/spiral/logger.svg?branch=master)](https://travis-ci.org/spiral/logger) [![Codecov](https://codecov.io/gh/spiral/logger/branch/master/graph/badge.svg)](https://codecov.io/gh/spiral/logger/)
spiral/tokenizer | [![Latest Stable Version](https://poser.pugx.org/spiral/tokenizer/version)](https://packagist.org/packages/spiral/tokenizer) [![Build Status](https://travis-ci.org/spiral/tokenizer.svg?branch=master)](https://travis-ci.org/spiral/tokenizer) [![Codecov](https://codecov.io/gh/spiral/tokenizer/branch/master/graph/badge.svg)](https://codecov.io/gh/spiral/tokenizer/)
spiral/snapshots | [![Latest Stable Version](https://poser.pugx.org/spiral/snapshots/version)](https://packagist.org/packages/spiral/snapshots) [![Build Status](https://travis-ci.org/spiral/snapshots.svg?branch=master)](https://travis-ci.org/spiral/snapshots) [![Codecov](https://codecov.io/gh/spiral/snapshots/branch/master/graph/badge.svg)](https://codecov.io/gh/spiral/snapshots/)
spiral/translator | [![Latest Stable Version](https://poser.pugx.org/spiral/translator/version)](https://packagist.org/packages/spiral/translator) [![Build Status](https://travis-ci.org/spiral/translator.svg?branch=master)](https://travis-ci.org/spiral/translator) [![Codecov](https://codecov.io/gh/spiral/translator/branch/master/graph/badge.svg)](https://codecov.io/gh/spiral/translator/)
spiral/models | [![Latest Stable Version](https://poser.pugx.org/spiral/models/version)](https://packagist.org/packages/spiral/models) [![Build Status](https://travis-ci.org/spiral/models.svg?branch=master)](https://travis-ci.org/spiral/models) [![Codecov](https://codecov.io/gh/spiral/models/branch/master/graph/badge.svg)](https://codecov.io/gh/spiral/models/)
spiral/debug | [![Latest Stable Version](https://poser.pugx.org/spiral/debug/version)](https://packagist.org/packages/spiral/debug) [![Build Status](https://travis-ci.org/spiral/debug.svg?branch=master)](https://travis-ci.org/spiral/debug) [![Codecov](https://codecov.io/gh/spiral/debug/branch/master/graph/badge.svg)](https://codecov.io/gh/spiral/debug/)
spiral/dumper | [![Latest Stable Version](https://poser.pugx.org/spiral/dumper/version)](https://packagist.org/packages/spiral/dumper) [![Build Status](https://travis-ci.org/spiral/dumper.svg?branch=master)](https://travis-ci.org/spiral/dumper) [![Codecov](https://codecov.io/gh/spiral/dumper/branch/master/graph/badge.svg)](https://codecov.io/gh/spiral/dumper/)
spiral/views | [![Latest Stable Version](https://poser.pugx.org/spiral/views/version)](https://packagist.org/packages/spiral/views) [![Build Status](https://travis-ci.org/spiral/views.svg?branch=master)](https://travis-ci.org/spiral/views) [![Codecov](https://codecov.io/gh/spiral/views/branch/master/graph/badge.svg)](https://codecov.io/gh/spiral/views/)
spiral/storage | [![Latest Stable Version](https://poser.pugx.org/spiral/storage/version)](https://packagist.org/packages/spiral/storage) [![Build Status](https://travis-ci.org/spiral/storage.svg?branch=master)](https://travis-ci.org/spiral/storage) [![Codecov](https://codecov.io/gh/spiral/storage/branch/master/graph/badge.svg)](https://codecov.io/gh/spiral/storage/)
spiral/reactor | [![Latest Stable Version](https://poser.pugx.org/spiral/reactor/version)](https://packagist.org/packages/spiral/reactor) [![Build Status](https://travis-ci.org/spiral/reactor.svg?branch=master)](https://travis-ci.org/spiral/reactor) [![Codecov](https://codecov.io/gh/spiral/reactor/branch/master/graph/badge.svg)](https://codecov.io/gh/spiral/reactor/)
spiral/annotations | [![Latest Stable Version](https://poser.pugx.org/spiral/annotations/version)](https://packagist.org/packages/spiral/annotations) [![Build Status](https://travis-ci.org/spiral/annotations.svg?branch=master)](https://travis-ci.org/spiral/annotations) [![Codecov](https://codecov.io/gh/spiral/annotations/branch/master/graph/badge.svg)](https://codecov.io/gh/spiral/annotations/)
spiral/broadcast | [![Latest Stable Version](https://poser.pugx.org/spiral/broadcast/version)](https://packagist.org/packages/spiral/broadcast) [![Build Status](https://travis-ci.org/spiral/broadcast.svg?branch=master)](https://travis-ci.org/spiral/broadcast) [![Codecov](https://codecov.io/gh/spiral/broadcast/branch/master/graph/badge.svg)](https://codecov.io/gh/spiral/broadcast/)

## 协同组件
| 组件 | 当前状态        
| ---       | ---
spiral/dotenv-bridge | [![Latest Stable Version](https://poser.pugx.org/spiral/dotenv-bridge/version)](https://packagist.org/packages/spiral/dotenv-bridge) [![Build Status](https://travis-ci.org/spiral/dotenv-bridge.svg?branch=master)](https://travis-ci.org/spiral/dotenv-bridge) [![Codecov](https://codecov.io/gh/spiral/dotenv-bridge/branch/master/graph/badge.svg)](https://codecov.io/gh/spiral/dotenv-bridge/)
spiral/monolog-bridge | [![Latest Stable Version](https://poser.pugx.org/spiral/monolog-bridge/version)](https://packagist.org/packages/spiral/monolog-bridge) [![Build Status](https://travis-ci.org/spiral/monolog-bridge.svg?branch=master)](https://travis-ci.org/spiral/monolog-bridge) [![Codecov](https://codecov.io/gh/spiral/monolog-bridge/branch/master/graph/badge.svg)](https://codecov.io/gh/spiral/monolog-bridge/)
spiral/twig-bridge | [![Latest Stable Version](https://poser.pugx.org/spiral/twig-bridge/version)](https://packagist.org/packages/spiral/twig-bridge) [![Build Status](https://travis-ci.org/spiral/twig-bridge.svg?branch=master)](https://travis-ci.org/spiral/twig-bridge) [![Codecov](https://codecov.io/gh/spiral/twig-bridge/branch/master/graph/badge.svg)](https://codecov.io/gh/spiral/twig-bridge/)
spiral/sentry-bridge | [![Latest Stable Version](https://poser.pugx.org/spiral/sentry-bridge/version)](https://packagist.org/packages/spiral/sentry-bridge) [![Build Status](https://travis-ci.org/spiral/sentry-bridge.svg?branch=master)](https://travis-ci.org/spiral/sentry-bridge) [![Codecov](https://codecov.io/gh/spiral/sentry-bridge/branch/master/graph/badge.svg)](https://codecov.io/gh/spiral/sentry-bridge/)

## 开发组件
| 组件 | 状态        
| ---       | ---
spiral/scaffolder | [![Latest Stable Version](https://poser.pugx.org/spiral/scaffolder/v/stable)](https://packagist.org/packages/spiral/scaffolder) [![Build Status](https://travis-ci.org/spiral/scaffolder.svg?branch=master)](https://travis-ci.org/spiral/scaffolder) [![Codecov](https://codecov.io/gh/spiral/scaffolder/branch/master/graph/badge.svg)](https://codecov.io/gh/spiral/scaffolder/)
spiral/prototype | [![Latest Stable Version](https://poser.pugx.org/spiral/prototype/version)](https://packagist.org/packages/spiral/prototype) [![Build Status](https://travis-ci.org/spiral/prototype.svg?branch=master)](https://travis-ci.org/spiral/prototype) [![Codecov](https://codecov.io/gh/spiral/prototype/branch/master/graph/badge.svg)](https://codecov.io/gh/spiral/prototype/)
spiral/annotations | [![Latest Stable Version](https://poser.pugx.org/spiral/annotations/version)](https://packagist.org/packages/spiral/annotations) [![Build Status](https://travis-ci.org/spiral/annotations.svg?branch=master)](https://travis-ci.org/spiral/annotations) [![Codecov](https://codecov.io/gh/spiral/annotations/graph/badge.svg)](https://codecov.io/gh/spiral/annotations)
spiral/composer-publish-plugin | [![Latest Stable Version](https://poser.pugx.org/spiral/composer-publish-plugin/version)](https://packagist.org/packages/spiral/composer-publish-plugin) [![Build Status](https://travis-ci.org/spiral/composer-publish-plugin.svg?branch=master)](https://travis-ci.org/spiral/composer-publish-plugin) [![Codecov](https://codecov.io/gh/spiral/composer-publish-plugin/branch/master/graph/badge.svg)](https://codecov.io/gh/spiral/composer-publish-plugin/)
spiral/code-style | [![Latest Stable Version](https://poser.pugx.org/spiral/code-style/version)](https://packagist.org/packages/spiral/code-style) [![Build Status](https://travis-ci.org/spiral/code-style.svg?branch=master)](https://travis-ci.org/spiral/code-style) [![Codecov](https://codecov.io/gh/spiral/code-style/branch/master/graph/badge.svg)](https://codecov.io/gh/spiral/code-style/)

开源协议:
--------
MIT License (MIT). 请阅读 [`LICENSE`](../LICENSE) 了解详情. 项目由 [Spiral Scout](https://spiralscout.com) 维护。
