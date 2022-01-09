go 错误记录

## 1. goland missing dependency :

现象: goland编译正常, 但是在goland中ctrl + 左键跳转的时候跳转不了, go.mod文件依赖飘红.

![missing_dependency](Untitled/image-20211104001631280.png)

原因: 本地存在了多个版本的第三方依赖缓存导致

解决: 执行 `go clean --modcache` 清理缓存就恢复正常了

## 2. goland中项目能够正常运行,但是import依赖飘红,并且go.mod文件里面依赖也是红的:

可能是由于goland的.idea文件有问题, 将.idea文件删除后重启项目, 项目就能正常了.

