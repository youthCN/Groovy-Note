#Groovy笔记

### groovy常用解析工具
1. json工具
	(1). Json转换成实体对象:JsonSlurper
	(2). 实体对象转换成Json: JsonOutput
2. xml工具
	(1).解析xml工具:XmlSlurper 
	(2).xml数据生成:MarkupBuilder
```
def xmlSlurper =  new XmlSlurper()
def response = xmlSlurper.parseText(xmlStr)
```
3. 文件处理
	(1).文本文件处理
	(2).对象处理
	
 
# Gradle #

### 一.gradle执行流程

![](/pic/process.png)

在settings.gradle文件中,初始化阶段开始
```
println("初始化阶段开始执行")
```
在app.gradle文件中,调用监听gradle执行的生命周期回调:
```
//配置阶段开始前的监听回调
this.beforeEvaluate {
    println "配置阶段开始前..."
}

//配置阶段完成以后的回调
this.afterEvaluate {
    println "配置阶段执行完毕..."
}

//gradle执行完毕以后的回调监听
this.gradle.buildFinished {
    println "执行完毕..."
}
```

其他监听方法:

```
this.gradle.beforeProject {
    println "配置阶段开始前..."
}
该方法等同于 beforeEvaluate
```

```
this.gradle.afterProject {}
等同于 afterEvaluate
```

添加Listener
```
this.gradle.addListener(){}
this.gradle.addBuildListener(){}
this.gradle.addProjectEvaluationListener(){}
this.gradle.addShutdownHook {}(){}
```

###二.Project核心api
**1. project概念**
	包含一个build.gradle的，一般认为是一个project；所以在一个Android项目中，project是一个gradle中的project，一个model也是一个project。

**2. api组成**
	![](/pic/projectApi.png)

在Project中的build.gradle中执行：

**2.1 Project(工程)相关Api**
```
printProjects()

def printProjects(){
    println "打印项目里面所有的Project。。。。。。。。"
    getAllprojects().eachWithIndex {Project project,int index ->
        println "name = ${project.name},index = ${index}"
    }
    println "打印该项目里面所有的子Project》》》》》》》》"
    this.getChildProjects().eachWithIndex {Project project,int index ->
        println "name = ${project.name},index = ${index}"
    };
//    this.getParent() 获取父Project
//    this.getRootProject() 获取根Project
}
```
父工程中配置子过程
```
/**
 * 该Api直接拿到一个过程Project对象,直接在父工程中对子工程进行配置,
 * 但是一般不推荐这么做,保证各个模块配置的单一性,公共配置,可以抽取到这里
* Project project(String path, Closure configureClosure);
*/
project('app'){ Project project->
    apply plugin:'com.android.application'
    group 'com.gcy.test'
    version '1.0.0-release'
    dependencies{
        
    }
    android{
        
    }
}
```

```
/**
 * <p>Configures this project and each of its sub-projects.</p>
 * 配置当前Project和其子Project
 */
allprojects {
    group 'com.gcy.test'
    version '1.0.0-release'
}
println project('app').group
```

```
//不包括当前结点工程,只包括它的子工程
subprojects { Project project ->
    if (project.plugins.hasPlugin('com.android.library')){
        //通过apply引入其他的gradle配置,相当于导包.
        apply from:'../publishToMaven.gradle'
    }
}
```

**2.2 Properties(属性)相关Api**

属性是指Project类中的字段, 自带属性有如下字段:
```
public interface Project extends Comparable<Project>, ExtensionAware, PluginAware {
    /**
     * The default project build file name.
     */
    String DEFAULT_BUILD_FILE = "build.gradle";

    /**
     * The hierarchy separator for project and task path names.
     */
    String PATH_SEPARATOR = ":";

    /**
     * The default build directory name.
     */
    String DEFAULT_BUILD_DIR_NAME = "build";

    String GRADLE_PROPERTIES = "gradle.properties";

    String SYSTEM_PROP_PREFIX = "systemProp";

    String DEFAULT_VERSION = "unspecified";

    String DEFAULT_STATUS = "release";
```

自带属性往往不能满足需求,我们可以扩展属性,扩展属性的方式有如下3种.

**2.2.1 自定义变量**
```
def mCompileSdkVersion = 27
android {
    compileSdkVersion mCompileSdkVersion//替换了27
    defaultConfig {
        applicationId "gcytest.apt_test"
        minSdkVersion 21
        targetSdkVersion 27
        versionCode 1
        versionName "1.0"
        testInstrumentationRunner "android.support.test.runner.AndroidJUnitRunner"
    }
    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        }
    }
}
```

**2.2.2 通过扩展块定义扩展属性**
```
ext{
    mCompileSdkVersion = 27
}
android {
    compileSdkVersion this.mCompileSdkVersion
    defaultConfig {
        applicationId "gcytest.apt_test"
        minSdkVersion 21
        targetSdkVersion 27
        versionCode 1
        versionName "1.0"
        testInstrumentationRunner "android.support.test.runner.AndroidJUnitRunner"
    }
    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        }
    }
}
```
使用这种方式,可以把扩展属性块放在父Project中,这样定义在父Project中的变量,可以在子工程中使用.
父工程:
```
ext{
    mCompileSdkVersion = 27
}
```
子工程
```
android {
//实际上,直接使用 this.mCompileSdkVersion 也是可以的,这个会被子工程继承
    compileSdkVersion this.rootProject.mCompileSdkVersion
    defaultConfig {
        applicationId "gcytest.apt_test"
        minSdkVersion 21
        targetSdkVersion 27
        versionCode 1
        versionName "1.0"
        testInstrumentationRunner "android.support.test.runner.AndroidJUnitRunner"
    }
    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        }
    }
}
```
* 通用方式
  定义一个配置文件config.gradle
```
ext{
    androidConfig = [mCompileSdkVersion:27]//定义一个map映射
//    mCompileSdkVersion = 27
}
```
  在父工程中引入配置
```
apply from:file('config.gradle')
```
 在子工程中可以直接使用这种配置
```
android {
    compileSdkVersion this.androidConfig.mCompileSdkVersion
    ...
}
```
**2.2.3 在gradle.properties配置文件中自定义属性**
例如动态依赖一个工程:
```
isLoadTest = false
```
在setting.gradle中
```
if(hasProperty('isLoadTest')?isLoadTest.toBoolean():false){
    include ':Test'
}
```

**3.文件属性Api**

**3.1 文件目录api**
在父工程中的gradle中执行:
```
println 'the root file path is:'+getRootDir().absolutePath
println 'the build file path is:'+getBuildDir()
println 'the project file path is:'+getProjectDir()
```
结果:
```
the root file path is:E:\MyCode\androidcode\APT_Test
the build file path is:E:\MyCode\androidcode\APT_Test\build
the project file path is:E:\MyCode\androidcode\APT_Test

```
3.2文件定位api
 File file(Object path) 方法
该方法和Groovy中 new File 不同,这个是gradle中的一个方法,这个方法相对于当前工程目录
```
println getContent('app/build.gradle')
def getContent(String path){
    try {
        def file = file(path)
        return file.text
    }catch (GradleException e){
        println 'file is not find'
    }
    return null
}
```
其他方法
ConfigurableFileCollection files(Object... paths);
3.3文件拷贝


### 三. Task
