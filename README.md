# 土制热加载框架

得了吧，你自己写的项目有多大你自己心里没x数么，大厂的什么乱七八糟的 hot patch 还是一边凉快去吧。

这玩意脚本加代码不到百行，不需要引入其他裤子，中小型项目可以引入来使了。

## 原理

构建时把 app 子项目 的 kotlin 代码构建出来的 classes 提早 dx 掉，然后不让他们进入 classes.dex 里，接着在 Application 里把 dex 加载进来。

弊端呢，就是 Application 类必须要用 java 写否则起不来。不过我相信对大多数项目来说没这个问题。

## 实现

build.gradle
```gradle
apply from: 'sdktools.gradle'
// 看这里 https://github.com/undownding/autodetect_android_sdk_and_buildTools

...
// 合适的地方加这个，你懂的
 i.assets.srcDirs += "src/${i.name}/assets-hotload"
...

task finalize() {
    doFirst {
        println('开始构建热加载模块')

        if (System.getProperty('os.name').toLowerCase(Locale.ROOT).contains('windows')) {
            exec {
                workingDir "${projectDir}\\build\\tmp\\kotlin-classes\\"
                commandLine "${android.getSdkDirectory().toString()}\\build-tools\\${android.buildToolsVersion}\\dx.bat"             \
            , "--dex"             \
            , "--output=..\\oldschool.dex"             \
            , "release"
            }
        } else {
            exec {
                workingDir "${projectDir}/build/tmp/kotlin-classes/"
                commandLine "${project.androidSDKDir}/build-tools/${android.buildToolsVersion}/dx"             \
            , "--dex"             \
            , "--output=../oldschool.dex"             \
            , "release"
            }
        }
        //We don't want in our APK this .class
        project.delete "build/tmp/kotlin-classes/release"
        project.mkdir "build/tmp/kotlin-classes/release"

        project.delete "src/main/assets-hotload"
        project.mkdir "src/main/assets-hotload"
        copy {
            from "${projectDir}/build/tmp/oldschool.dex"
            into "src/main/assets-hotload/"
        }
    }
}

tasks.whenTaskAdded { task ->
    if (task.name == "compileReleaseKotlin") {
        task.finalizedBy finalize
    }
}

```

最终生成出来的 oldschool.dex 已经进入到 apk 的 assets 里了，然后我们需要在 Application 里加载起来。

BaseApplication.java
```java
        // 处理动态加载
        if (!BuildConfig.DEBUG) {
            final File oldSchool = new File(getFilesDir().getAbsolutePath() + "/oldschool.dex");
            final File newSchool = new File(getFilesDir().getAbsolutePath() + "/oldschool.dex");

            // 如果 newschool.dex 存在, 则加载
            // SplashActivity 里检查并下载 newschool.dex
            if (newSchool.exists()) {
                final DexClassLoader classLoader = new DexClassLoader(
                        newSchool.getAbsolutePath(),
                        this.getDir("dexOpt", MODE_PRIVATE).getAbsolutePath(),
                        null,
                        getClassLoader()
                );
                setApkClassLoader(classLoader);
            } else { // 如果 newschool.dex 不存在，则加载 oldschool.dex
                if (!oldSchool.exists()) { // 需要加载 oldschool.dex 但是不存在的话，先从 assets 里解压出来
                    try (FileOutputStream steam = new FileOutputStream(oldSchool, false)) {
                        InputStream inputStream = getAssets().open("oldschool.dex");

                        byte[] buffer = new byte[8192];
                        int bytesRead;
                        while ((bytesRead = inputStream.read(buffer)) != -1) {
                            steam.write(buffer, 0, bytesRead);
                        }
                    } catch (IOException e) {
                        e.printStackTrace();
                    }
                    final DexClassLoader classLoader = new DexClassLoader(
                            oldSchool.getAbsolutePath(),
                            this.getDir("dexOpt", MODE_PRIVATE).getAbsolutePath(),
                            null,
                            getClassLoader()
                    );
                    setApkClassLoader(classLoader);
                }
            }
        }
```

``setApkClassLoader`` 是为了把 Application 的 classloader 替换为我们自己的，网上现在有好几种方案，不过我都不太满意。先观望，回头等有满意的再贴出来。
