# 组件化整体方案
## 目录结构
```
app
- app_* (业务模块文件夹,包含业务模块&业务对外api模块。并且每个业务模块的base层)
    - app_base_*
    - app_base_*_api
    - app_host1_*
    - app_host1_*_api
    - app_host2_*
    - app_host2_*_api

- vendor_* (组件模块)
```

![](./img/frame.png)

![](./img/api-frame.png)

## project source引入与maven aar引入
备注：
    1. 原则上业务模块都需要支持动态切换俩种引入方式， 组件模块可以都使用maven aar引入   
    2. 引用其他模块api需要用aar方式（implementation(depsApp.app.app_host1_login_api)）

如何使用，app_login为例 (详见：./setting_template.gradle)
1. 添加模块git到工程 git submodule add ../app_login.git
2. settings.gradle文件moduleInfoList添加对应配置
    ```
    List moduleInfoList = [
        [sourceInclude: Boolean.valueOf(findProperty("app_host1_login.sourceInclude", false)),
                 hasApiModule : true, //为true则会自动依赖app_host1_login_api
                 artifactId   : "app_host1_login",
                 group        : "com.github.hcanyz",
                 path         : "app_login/app_host1_login"],
    ]
    includeDynamic(gradle, settings, moduleInfoList)
    ```
    备注：sourceInclude是否project方式引入，可以在local.properties配置,默认格式app_host1_login.sourceInclude=true   
    artifactId项目id   
    group项目组id   
    path项目在当前工程的相对目录
3. 其他模块具体依赖方法
    例如：app_host1_im模块需要依赖app_host1_login_api模块，在app_host1_im的build.gradle中

    ```
    implementation(depsApp.app.app_host1_login_api) {
        changing = true
    }
    ```
    
    这样就可以根据app_host1_login_api的sourceInclude配置动态引入了
    
    备注：changing = true 这个可以加上，保证aar引入时一定会取服务器最新的aar包
4. 多arr maven上传脚本
    找一个build.gradle文件,加入下面方法，运行
    ```
    task uploadArchivesAll(dependsOn: [':app_base_login:uploadArchives',
                                       ':app_base_login_api:uploadArchives',
                                       ':app_host1_login:uploadArchives',
                                       ':app_host1_login_api:uploadArchives',
                                       ':app_host2_login:uploadArchives',
                                       ':app_host2_login_api:uploadArchives',]) {
        doLast {
            println("success")
        }
    }
    ```

## 各个组件独立初始化
#### 需要初始化的模块
1. build.gradle 添加 implementation deps.vendor.vendor_moduleinit 依赖

2. 建立初始化文件，其中target = module_init_async_arrow_key表示异步执行 = module_init_sync_arrow_key表示同步执行
    ```
    @AutoBowArrow(target = module_init_sync_arrow_key, context = true)
    class ModuleInit(private val application: Application) : IAutoBowArrow {
        override fun shoot() {
            HelloWorld().test(application)
        }
    }
    ```

    注: 有些模块初始化需要依赖host工程的数据，可以放在host工程的vendor_linker项目中统一初始化

#### host 执行初始化
1. 根目录的的build dependencies 添加 classpath deps.build.autoInjectPlugin

2. host的build.gradle 添加 implementation deps.vendor.vendor_moduleinit 依赖

3. 在host的build.gradle  android deps.gradleConfig.simpleAndroid的下面添加includeAutoInject(project),如下
    ```
    android deps.gradleConfig.simpleAndroid
    includeAutoInject(project)
    ```

4. 指定项目Application 为 com.github.hcanyz.moduleinit.ModuleInitApplication
   或者在自己的Application写一份ModuleInitApplication中的代码
       
#### ！启动时间关键指标
接入ModuleInitApplication后可以在logcat中查看 "StartAnalysis" 数据，观察启动时消耗时间 