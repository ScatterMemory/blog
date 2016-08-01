# 关于使用cocos2d编译android项目
 编译环境win10，编辑相关工具如下：
 android-studio 2.1.2 下载地址:
  https://developer.android.com/studio/index.html(官网下载,国内需翻墙)
  http://www.androiddevtools.cn/(无需翻墙，但不一定是最新的)
 cocos2d-x 3.12 下载地址：
  http://www.cocos.com/download/#
 JDK 下载地址：
  http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html
 NDK ：
  这个可以在android-studio里面下载(我是用的是最新的r12b的，目前官方推荐使用的是r10e的版本，这个可以自行下载)
 ANT 下载地址：
  https://ant.apache.org/bindownload.cgi
 python 下载地址：
   https://www.python.org/downloads/
  
 下载安装完后需要配置一些环境变量：
 ANDROID_SDK_ROOT你的sdk根目录
 ANT_HOME 你的ant根目录
 NDK_ROOT 你的ndk根目录
 path变量里面添加值%ANT_HOME%/bin;%ANT_HOME%/lib
 可以运行cocos2d根目录下的setup.py来确保你没有遗漏
 
 上述步骤完成后运行一个cmd命令
  运行cocos命令：
   cocos new -p com.test.game -d E:\test\code -l cpp testCocos2d
  cocos new 表示新建一个项目 -p 指定你的包名，-d 是你新建项目的存放路径 -l指定项目使用语言，
  可以选择lua,js,cpp 最后则是你的项目的名称，这里没有指定模板，cocos会自动拷贝默认模板，
  可以通过-t来制定模板
 
 执行命令完成后就会看到E:\test\code目录下有一个testCocos2d的项目，里面有各种版本，
 cd命令进入你创建的项目的根目录
 使用命令：
 cocos compile -p android --android-studio --ap android-23
 发现错误信息
 make: Entering directory `E:/code/MyCppGame/proj.android-studio/app'
E:\code\MyCppGame\proj.android-studio\../cocos2d/cocos/./Android.mk:250: *** commands commence before first target.  Stop.
make: Leaving directory `E:/code/MyCppGame/proj.android-studio/app'
执行命令出错，返回值：2。
发现是由于E:\code\MyCppGame\proj.android-studio\../cocos2d/cocos/./Android.mk:249行末尾少了 " \"导致的（目测是官方3.12版本的BUG）
加上后再执行一遍发现有出现新的错误
Android NDK: ERROR:E:\code\MyCppGame\proj.android-studio\../cocos2d/external/freetype2/prebuilt/android/Android.mk:cocos_freetype2_static: LOCAL_SRC_FILES points to a missing file
Android NDK: Check that E:/code/MyCppGame/proj.android-studio/../cocos2d/external/freetype2/prebuilt/android/arm64-v8a/libfreetype.a exists  or that its path is correct
 是缺少arm64-v8a这个库导致的，百思不得其解。
 
 后来发现因为ndk换成了r12版本里面只有arm-linux-androideabi-4.9然后发现\plugin\tools下的android_build.py里面有这么一段代码
 
 def select_toolchain_version():
    '''Because ndk-r8e uses gcc4.6 as default. gcc4.6 doesn't support c++11. So we should select gcc4.7 when
    using ndk-r8e. But gcc4.7 is removed in ndk-r9, so we should determine whether gcc4.7 exist.
    Conclution:
    ndk-r8e  -> use gcc4.7
    ndk-r9   -> use gcc4.8
    '''

    ndk_root = check_environment_variables()
    if os.path.isdir(os.path.join(ndk_root,"toolchains/arm-linux-androideabi-4.8")):
        os.environ['NDK_TOOLCHAIN_VERSION'] = '4.8'
        print "The Selected NDK toolchain version was 4.8 !"
    elif os.path.isdir(os.path.join(ndk_root,"toolchains/arm-linux-androideabi-4.7")):
        os.environ['NDK_TOOLCHAIN_VERSION'] = '4.7'
        print "The Selected NDK toolchain version was 4.7 !"
    else:
        print "Couldn't find the gcc toolchain."
        exit(1)
 于是我手动加上
elif os.path.isdir(os.path.join(ndk_root,"toolchains/arm-linux-androideabi-4.9")):
        os.environ['NDK_TOOLCHAIN_VERSION'] = '4.9'
        print "The Selected NDK toolchain version was 4.9 !"
不知道这个对编辑结果是否存在影响，没验证（真心好累啊）。
然后又发现tools\cocos2d-console\plugins\plugin_generate目录下的gen_libs.py里面有这么一段代码

  if not self.disable_strip:
            # strip the android libs
            ndk_root = os.environ["NDK_ROOT"]
            if cocos.os_is_win32():
                if cocos.os_is_32bit_windows():
                    check_bits = [ "", "-x86_64" ]
                else:
                    check_bits = [ "-x86_64", "" ]

                sys_folder_name = "windows"
                for bit_str in check_bits:
                    check_folder_name = "windows%s" % bit_str
                    check_path = os.path.join(ndk_root, "toolchains/arm-linux-androideabi-4.8/prebuilt/%s" % check_folder_name)
                    if os.path.isdir(check_path):
                        sys_folder_name = check_folder_name
                        break
            elif cocos.os_is_mac():
                sys_folder_name = "darwin-x86_64"
            else:
                sys_folder_name = "linux-x86_64"

            # set strip execute file name
            if cocos.os_is_win32():
                strip_execute_name = "strip.exe"
            else:
                strip_execute_name = "strip"

            # strip arm libs
            strip_cmd_path = os.path.join(ndk_root, "toolchains/arm-linux-androideabi-4.8/prebuilt/%s/arm-linux-androideabi/bin/%s"
                % (sys_folder_name, strip_execute_name))
            if os.path.exists(strip_cmd_path):
                armlibs = ["armeabi", "armeabi-v7a"]
                for fold in armlibs:
                    self.trip_libs(strip_cmd_path, os.path.join(android_out_dir, fold))

            # strip x86 libs
            strip_cmd_path = os.path.join(ndk_root, "toolchains/x86-4.8/prebuilt/%s/i686-linux-android/bin/%s" % (sys_folder_name, strip_execute_name))
            if os.path.exists(strip_cmd_path) and os.path.exists(os.path.join(android_out_dir, "x86")):
                self.trip_libs(strip_cmd_path, os.path.join(android_out_dir, 'x86'))

可以确定它的确默认是没有arm-v8a的，网上说是通过增加ABI来解决，但是我这里是直接尝试在cocos命令里面指定它编译的armeabi来偷巧
最终编辑命令如下：
cocos compile -p --android-studio --target android-23 --ap android-23 --ndk-mode debug --ndk-toolchain arm-linux-androideabi-4.9 --platform android --app-abi armeabi --src E:\code\testCocos
终于编辑成功
BUILD SUCCESSFUL

Total time: 3 mins 35.176 secs
PREDEX CACHE HITS:   0
PREDEX CACHE MISSES: 3
Stopped 0 compiler daemon(s).
正在移动 apk 文件 E:\code\testCocos\bin\debug\android
编译成功。(对比一下unity3d直接可以运行到手机上看调试效果，真是累觉不爱了)。
