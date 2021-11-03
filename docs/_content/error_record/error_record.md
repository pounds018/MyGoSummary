go 错误记录

## 1. goland missing dependency :

现象: goland编译正常, 但是在goland中ctrl + 左键跳转的时候跳转不了, go.mod文件依赖飘红.

![missing_dependency](Untitled/image-20211104001631280.png)

原因: 本地存在了多个版本的第三方依赖缓存导致

解决: 执行 `go clean --modcache` 清理缓存就恢复正常了