Android Studio通过远程gradle文件灵活控制依赖包的版本

## 前言

对于同时开发多个应用的团队来讲，总会希望在各个方面做得规范和统一（比如编码规范、项目框架、开发流程等），但开发者的素质肯定是参差不齐，所以依赖人为地约束效果肯定不理想，需要共同遵守的规范最好是能够通过工具来做自动化检测，这样就不会因为人的原因导致规范执行得不理想，或者是频繁出错。
比如：

1、对于编码规范的约束可以在代码提交前通过脚本对代码做静态检测（githook或者gerrit），只要不符合规范就无法提交代码（除对代码做静态检测外，还可以通过自定义提交前检查的脚本做很多事情，比如对项目目录结构的检测、版本号和版本名的检查、是否有通用基础功能的检查，反正都是对文件做分析，只要每一项规则有规律可循，都可以通过脚本来做匹配检测）；

2、通过jenkins上的脚本来判断每次发布版本时版本号和版本名是否对应，是否都做了改动；

3、在项目交付的每一个流程通过CheckList检查项来避免问题流向下一个环节，通过人工/脚本检测项目在各个交付阶段是否符合CheckList检查项的要求，不符合就退回（比如项目发布前的检查，版本号和版本名是否OK、是否有自升级功能、是否有采集功能等）；

4、通过APM检测每个应用的性能是否达标。

## 现实问题

除了上面所说的这些外，还存在一个让团队开发特别烦恼的问题，那就是对APP公共配置的控制，特别是同时开发多个应用的团队，具体问题有：

1、如何保证各个应用对每个第三方开源库的版本依赖统一？

2、如何保证团队内部开发的公共库能够及时更新到每个应用，并且保证每个应用都是使用的最新版本？

3、如何保证各个应用的签名文件统一？

4、如何保证各个应用的版本号和版本名命名规则统一？

## 解决方案

在网上找了一圈都没有符合要求的解决方案，有必要分享一下自己的实际项目开发经验，可以通过在build.gradle中apply远程的gradle文件，所有的公共配置都定义在远程配置中，每个应用只需要apply这个gradle文件，就能够实现公共配置的统一，实例如下：

Step 1：定义公共的config.gradle文件并上传到服务器：

	ext.dependenceConfig = [
            compileSdkVersion          : 23,
            buildToolsVersion          : "24.0.0",
            minSdkVersion              : 15,
            targetSdkVersion           : 23,
            deps             : [
                    "3rd-fastjson"                  : 'com.alibaba:fastjson:1.1.56.android',
                    "3rd-leakcanary-android"        : 'com.squareup.leakcanary:leakcanary-android:1.5.1',
                    "3rd-leakcanary-android-no-op"  : 'com.squareup.leakcanary:leakcanary-android-no-op:1.5.1',
                    "3rd-fresco"                    : 'com.facebook.fresco:fresco:1.3.0',
                    "3rd-okhttp"                    : 'com.squareup.okhttp3:okhttp:3.7.0',
            ],
    
            signing                    : [
                    "keyAlias"     : 'zhangmy',
                    "keyPassword"  : 'zhangmy',
                    "storePassword": 'zhangmy',
            ],
    ]
    

Step 2：在项目工程根目录下的build.gradle中apply Step 1中的config.gradle文件：

	apply from: "https://gist.githubusercontent.com/zmywly8866/94c126dd3ee16532bb1049fdc8e49233/raw/6ad45f85bd881c2bcf159c04ed7e2564b32a2efd/config.gradle"
    
    buildscript {
        repositories {
            jcenter()
        }
        dependencies {
            classpath 'com.android.tools.build:gradle:2.0.0'
        }
    }
    
    allprojects {
        repositories {
            jcenter()
        }
    }
    
    task clean(type: Delete) {
        delete rootProject.buildDir
    }

Step 3：在需要依赖公共配置的module中，修改build.gradle，实现对公共配置的依赖：

	apply plugin: 'com.android.application'
    
    android {
        compileSdkVersion dependenceConfig.compileSdkVersion
        buildToolsVersion dependenceConfig.buildToolsVersion
        defaultConfig {
            applicationId "com.zhangmy.applyremotegradlesample"
            minSdkVersion dependenceConfig.minSdkVersion
            targetSdkVersion dependenceConfig.targetSdkVersion
            versionCode 1
            versionName "1.0"
        }
        buildTypes {
            release {
                minifyEnabled false
                proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
            }
        }
    }
    
    dependencies {
        compile fileTree(include: ['*.jar'], dir: 'libs')
        compile dependenceConfig.deps.'3rd-fastjson'
        compile dependenceConfig.deps.'3rd-fresco'
        compile dependenceConfig.deps.'3rd-okhttp'
        debugCompile dependenceConfig.deps["3rd-leakcanary-android"]
        releaseCompile dependenceConfig.deps["3rd-leakcanary-android-no-op"]
    }

**说明：**实例参见[ApplyRemoteGradleSample](https://github.com/zmywly8866/ApplyRemoteGradleSample)，本例中将公共配置的gradle文件上传到了gist上面。

## 后记

apply远程的config.gradle文件有如下优点：

1、可以通过远程gradle文件控制各依赖库的版本，当某个依赖库的版本有改动时，通过修改这个config.gradle实现在应用无需做任何改动的情况下保证了所有应用的依赖版本变更；

2、此种方式与通过“+”的方式自动获取依赖库的最新版本相比，控制版本更为灵活，且不会导致编译时每次都需要检测依赖包是否有最新版本；

3、如果你们团队内部自己开发了很多公共的库，在发布库时可以通过远程gradle文件灵活控制各个库的版本，并且可以避免因为多个库依赖同一个库的版本不一致导致的编译冲突问题；

4、远程gradle文件还可以定义方法、配置APK的签名文件等，实现对所有应用的统一配置。