## 背景
基于OnlyOffice-DocumentServer-6.4.2
## 准备条件
1.代理，这个是必要条件。编译过程中需要下载许多资源，没有代理无法下载
2.操作系统：Ubuntu 18.04 arm64 OnlyOffice官方推荐14.04，但在尝试过程中发现arm版14.04会缺少部分库。
3.硬件：基于飞腾CPU，CPU及RAM配置尽可能高，硬盘空间最少为50GB
4.安装jdk
5.安装python2.7
## 前置知识
1.python基础知识
2.编译器基础知识，makefile文件结构有基础了解
3.QT体系的.pro文件结构有初步了解，qmake命令有初步了解
4.npm命令基础了解
## 总览
基本流程为：配置好运行环境，修改源码，把编译的各主要步骤分步执行，主要需要关注的为：v8的编译，核心组件的编译，编译后图片等资源的补全。
## 相关资料
DocumentServer源码：[https://github.com/ONLYOFFICE/DocumentServer](https://github.com/ONLYOFFICE/DocumentServer)  
构建工具Build-Tools源码：[https://github.com/ONLYOFFICE/build_tools](https://github.com/ONLYOFFICE/build_tools)   
OnlyOffice 编译文档：[https://helpcenter.onlyoffice.com/installation/docs-community-compile.aspx](https://helpcenter.onlyoffice.com/installation/docs-community-compile.aspx)  
OnlyOffice 安装文档：[https://helpcenter.onlyoffice.com/installation/docs-community-install-centos.aspx](https://helpcenter.onlyoffice.com/installation/docs-community-install-centos.aspx)  
V8引擎构建指南：[https://v8.dev/docs/contribute](https://v8.dev/docs/contribute)  
V8引擎ARM64构建指南：[https://v8.dev/docs/compile-arm64](https://v8.dev/docs/compile-arm64)  


## 具体步骤
​

### 执行脚本
​

#### 前期准备
下载build_tools到本机文件夹，如/data目录下  
需要主要关注的源码：  
/data/build_tools/tools/linux/automate.py 入口  
/data/build_tools/make.py 编译主流程定义在这里  
/data/build_tools/scripts/core_common/make_common.py 编译第三方组件主流程在这里  
/data/build_tools/scripts/core_common/modules/v8.py 编译v8步骤在这里  
/data/build_tools/scripts/build.py 编译核心组件  
/data/build_tools/scripts/build_server.py 编译web服务  
/data/build_tools/scripts/deploy_server.py 产出server所有执行程序  
/data/build_tools/build.pro 构建核心组件使用的pro文件 用于生成makefile  
/data/build_tools/build.makefile_linux_64 通过pro文件生成的makefile  


切换到脚本所在目录 /data/build_tools/tools/linux   
执行命令 ./automate.py server (参数server指仅编译server版本)静待脚本自动安装相关依赖及第三方组件。先忽略错误，走完完整流程，保证依赖的第三方组件都能够完整下载。  
第三方组件下载完毕后，可注释掉如下代码，防止再次下载，浪费时间：  
1.automate.py
```python
install_deps()  
install_qt()
```
2.make.py
```python
 if ("1" == config.option("update")):
     repositories = base.get_repositories()
     base.update_repositories(repositories)
```
接下来的编译，在make.py中分为5个步骤：  
make_common.make()  
build.make()  
build_js.make()  
build_server.make()  
deploy.make()  
首先执行make_common.make()暂时注释掉其他步骤。  


#### 编译v8
在执行make_common.make()时，会遇到v8编译相关的错误，其他组件暂无问题。  
主要原因是由于：1.v8编译用到的工具(gn和ninja)，默认下载的为x86版，所以需要先将工具手动编译 2.gcc 需要升级至7.5.0。v8这部分独立编译。  
1.下载工具depot_tools  
在/data/下创建文件夹，build_v8_only用于存放编译v8所需资源。按照上面提到的官方文档，下载工程depot_tools,并将其添加到环境变量。  
2.编译ninja，并将ninja添加到环境变量。  
3.编译gn，并将gn添加到环境变量。  
4.编译v8  
5.迁移文件  
将v8构建后的文件/data/build_v8_only/v8/out.gn/arm64.release/下所有问及那复制到：/data/core/Common/3dParty/v8/v8/out.gn/linux_64/使onlyoffice编译源码能正确引用v8相关文件  
至此，v8相关的编译完成。   
​

#### 解决核心组件的编译问题（build.make()）
修改pro文件/data/build_tools/build.pro
```
将
core_windows {
	CONFIG += core_and_multimedia
}
core_linux {
	CONFIG += core_and_multimedia
}
修改为
core_windows {
}
core_linux {
}
这部分组件编译会有问题，暂未找到解决方案，且该组件为视频播放器，对于文档在线预览编辑不是必须的
```
将build.py中的代码调整为如下
```python
    # # non windows platform
    # if not base.is_windows():
    #   if base.is_file(makefiles_dir + "/build.makefile_" + file_suff):
    #     base.delete_file(makefiles_dir + "/build.makefile_" + file_suff)
    #   print("+++++++++++++++++++++++++++++++++++++++++++++++++++++++make file: " + makefiles_dir + "/build.makefile_" + file_suff)
    #   print(pro_file)
    #   print(config_param)
    #   print(qmake_addon)
    #   base.cmd(qt_dir + "/bin/qmake", ["-nocache", pro_file, "CONFIG+=" + config_param] + qmake_addon)
    #   if ("1" == config.option("clean")):
    #     base.cmd_and_return_cwd(base.app_make(), ["clean", "-f", makefiles_dir + "/build.makefile_" + file_suff], True)
    #     base.cmd_and_return_cwd(base.app_make(), ["distclean", "-f", makefiles_dir + "/build.makefile_" + file_suff], True)
    #     base.cmd(qt_dir + "/bin/qmake", ["-nocache", pro_file, "CONFIG+=" + config_param] + qmake_addon)
    #   if ("0" != config.option("multiprocess")):
    #     # base.cmd_and_return_cwd(base.app_make(), ["-f", makefiles_dir + "/build.makefile_" + file_suff, "-j" + str(multiprocessing.cpu_count())])
    #      base.cmd_and_return_cwd(base.app_make(), ["-f", makefiles_dir + "/build.makefile_linux_32" , "-j" + str(multiprocessing.cpu_count())])
    #   else:
    #     base.cmd_and_return_cwd(base.app_make(), ["-f", makefiles_dir + "/build.makefile_" + file_suff])
    # else:
    #   print("is windows ignore") 
    base.cmd_and_return_cwd(base.app_make(), ["-f", makefiles_dir + "/build.makefile_" + file_suff])
```
即，通过pro文件生成makefile的步骤手动执行，仅保留执行makefile编译的过程  
通过pro文件生成makefile这一步骤使用QT的qmake工具，目前一个很奇怪的问题是，qmake始终无法正确识别系统位数，当前系统为64位，它始终生成的是32位文件，为解决这一问题，需要：  
1.执行命令，生成makefile /data/build_tools/tools/linux/qt_build/Qt-5.9.9/gcc_64/bin/qmake -nocache CONFIG+=server -spec linux-g++-64  
这里添加的参数：-spec linux-g++-64 是让为qmake指定位数  
2.修改生成的makefile文件（build.makefile_linux_64）  
生成的makefile由于上述添加了-spec参数，会在单独组件的makefile中的CFLAGS CXFLAGS添加-m64标识，这个标识编译器无法识别，需要通过下述脚本去除掉。  
通过vscode替换  
1.
-spec linux-g++-64 ) &&  
为
-spec linux-g++-64 ) && sed -i 's/-m64 //' Makefile.*linux_64 &&   
​

2.(vscode开启正则搜索)  
-spec linux-g\+\+-64 \)\n  
-spec linux-g++-64&& sed -i 's/-m64 //' Makefile.*linux_64\n  
修改过后继续执行流程。  


添加了-m64参数会遇到的错误信息：  
```
cd ../core/Common/3dParty/cryptopp/project/ && ( test -e /data/build_tools/../core/Common/3dParty/cryptopp/project/Makefile.cryptopplinux_64 || /data/build_tools/tools/linux/qt_build/Qt-5.9.9/gcc_64/bin/qmake -o /data/build_tools/../core/Common/3dParty/cryptopp/project/Makefile.cryptopplinux_64 /data/core/Common/3dParty/cryptopp/project/cryptopp.pro -nocache CONFIG+=server -spec linux-g++-64 ) && sed -i 's/-m64 //' Makefile.*linux_64 && make-f /data/build_tools/../core/Common/3dParty/cryptopp/project/Makefile.cryptopplinux_64
Project ERROR: Cannot run compiler 'g++'. Output:===================
Using built-in specs.COLLECT_GCC=g++
g++: error: unrecognized command line option '-m64'
Target: aarch64-linux-gnu
Configured with: ../src/configure -v --with-pkgversion='Ubuntu/Linaro 7.5.0-3ubuntu1~18.04' --with-bugurl=file:///usr/share/doc/gcc-7/README.Bugs --enable-languages=c,ada,c++,go,d,fortran,objc,obj-c++ --prefix=/usr --with-gcc-major-version-only --program-suffix=-7 --program-prefix=aarch64-linux-gnu- --enable-shared --enable-linker-build-id --libexecdir=/usr/lib --without-included-gettext --enable-threads=posix --libdir=/usr/lib --enable-nls --enable-bootstrap --enable-clocale=gnu --enable-libstdcxx-debug --enable-libstdcxx-time=yes --with-default-libstdcxx-abi=new --enable-gnu-unique-object --disable-libquadmath --disable-libquadmath-support --enable-plugin --enable-default-pie --with-system-zlib --enable-multiarch --enable-fix-cortex-a53-843419 --disable-werror --enable-checking=release --build=aarch64-linux-gnu --host=aarch64-linux-gnu --target=aarch64-linux-gnuThread model: posixgcc version 7.5.0 (Ubuntu/Linaro 7.5.0-3ubuntu1~18.04)
===================
Maybe you forgot to setup the environment?
makefiles/build.makefile_linux_64:71: recipe for target 'sub--data-core-Common-3dParty-cryptopp-project-cryptopp-pro-make_first-ordered' failed
make: *** [sub--data-core-Common-3dParty-cryptopp-project-cryptopp-pro-make_first-ordered] Error 3
Error (make): 2
Error (./make.py): 1
```


编译核心组件期间可能遇到的问题（XXXXX代表某核心组件）：  
```
/data/core/XXXXX/build/Qt/../../../build/lib/linux_64/libdoctrenderer.so: undefined reference to `vtable for v8::internal::SetupIsolateDelegate'
collect2: error: ld returned 1 exit status
/data/build_tools/../core/XXXXX/build/Qt/Makefile.XXXXXXlinux_64:213: recipe for target '../../../build/bin/linux_64/XXXXX' failed
```
该错误是某些核心组件没有编译完成或编译的有问题，清理重新编译即可，切换到该组件makefile所在目录，执行命令：  
make -f makefile的名称 clean  
make -f makefile的名称 distclean  
​

#### 执行后续剩余流程  
注释掉：  
make_common.make()  
build.make()  
执行剩余流程  
build_js.make()  
build_server.make()  
需要注释掉如下代码  
```python
  # example_dir = base.get_script_dir() + "/../../document-server-integration/web/documentserver-example/nodejs"
  # # /data/document-server-integration/web/documentserver-example/nodejs
  # base.delete_dir(example_dir  + "/node_modules")
  # base.cmd_in_dir(example_dir, "npm", ["install"])
  # sync_rpc_lib_dir = example_dir + "/node_modules/sync-rpc/lib"
  # patch_file = base.get_script_dir() + "/../tools/linux/sync-rpc.patch"
  # if ("linux" == base.host_platform()):  
  #   base.cmd_in_dir(sync_rpc_lib_dir, "patch", ["-N", "-i", patch_file])
  
  # base.cmd_in_dir(example_dir, "pkg", [".", "-t", pkg_target, "-o", "example"])
```
否则会报如下错误，这部分看起来是生成example相关的，可忽略掉。  
```
npm "install"
npm WARN OnlineEditorsExampleNodeJS@4.1.0 license should be a valid SPDX license expression

added 215 packages from 141 contributors and audited 215 packages in 7.431s

2 packages are looking for funding
  run `npm fund` for details

found 3 vulnerabilities (1 moderate, 1 high, 1 critical)
  run `npm audit fix` to fix them, or `npm audit` for details
Traceback (most recent call last):
  File "./make.py", line 95, in <module>
    build_server.make()
  File "scripts/build_server.py", line 62, in make
    base.cmd_in_dir(sync_rpc_lib_dir, "patch", ["-N", "-i", patch_file])
  File "scripts/base.py", line 335, in cmd_in_dir
    os.chdir(dir)
OSError: [Errno 2] No such file or directory: '/data/build_tools/scripts/../../document-server-integration/web/documentserver-example/nodejs/node_modules/sync-rpc/lib'
Error (./make.py): 1
```
deploy.make()  


#### 启动onlyoffice  
编译完成后，按照官方文档，依次执行部署流程：   
安装配置nginx  
安装配置postgresql并初始化数据库  
安装rabbitMQ  
启动服务（注，由于官方文档较旧，编译出的可执行程序并没有 SpellChecker服务）  
按照文档命令启动（前台）  
这样启动只能是前台启动，每个服务使用一个shell连接，shell断开后服务也随之停止，可采用supervisor、nuhup等方式启动。  
使用Supervisor托管启动  
为使document server相关服务后台运行，且使服务具备一定的自动恢复能能力，可使用supervisor进行管理。  
测试文档预览编辑：  
在本地创建html文档，替换js地址，检查文档是否正常打开。  
```html
<!DOCTYPE html>
<html>
    <head>
     <meta charset="utf-8">
     <script type="text/javascript" src="http://替换为onlyoffice服务地址/web-apps/apps/api/documents/api.js"></script>
    </style>
    </head>
    <body>
     <div id="placeholder" class = "nav"></div>
      <script language="javascript" type="text/javascript">
            new DocsAPI.DocEditor("placeholder", {
       "document": {
          "fileType": "docx",
          "key": "12NAFE",
          "title": "test6.docx",
          "url": "https://file-examples-com.github.io/uploads/2017/02/file-sample_100kB.doc"
      },
      "documentType": "text",
      "width": "1600px", 
            "height": "900px",
            "editorConfig": {
                "callbackUrl": "编辑word后保存时回调的地址"
            },
            "permissions": {
            "comment": true,
            "download": true,
            "edit": true, 
            "fillForms": true,
            "print": true,
            "review": true
        }
});
        </script>
    </body>
</html>
```


编译完成后缺少相关图片资源：  
启动服务后，访问页面，打开浏览器控制台，会发现许多图片缺失。可能是脚本的问题，web服务文件图片路径为/data/build_tools/out/linux_64/onlyoffice/documentserver/web-apps/apps/documenteditor/main/resources/img可以在如下目录查找缺失的图片资源，复制到上述路径中：  
/data/web-apps/apps/documenteditor/main/resources/img  


重新编译后，提示文档打开失败：  
如需重新编译某一组件，编译后启动服务，打开文档时如遇提示文档打开失败，日志无详细错误信息，可尝试drop掉onlyoffice的数据库，重新初始化数据库。  
​

​

​

