# NPM 版本冲突

npm 对于包的版本管理是根据 package.json 里指定的版本号来确定的。同时，第一次执行 npm install 后会生成 package-lock.json 文件，而后每次修改 node_modules 树或者修改 package.json 都会触发 package-lock.json 的更新。

我们知道 npm 自从 npm3 发布以来，将原来的 node_modules 目录由原来的树形结构改为扁平化结构。也就是说所有的依赖包默认都是安装在 node_modules 目录下第一层。

但是如果项目的 package.json 和项目某一个依赖 A 的 package.json 同时都依赖了另一个包 Dep，但是依赖的版本不同，那么这个时候 npm 就需要处理这种冲突关系。

npm 做法是当发现这种冲突的时候，就会把这个依赖 A 里的 Dep 安装到依赖 A 的 node_modules 目录下面。项目依赖的那个 Dep 包继续留在项目根目录的 node_modules 目录的第一层。package-lock.json 文件里也会同时出现两个不同版本的 Dep 包。

这时候如果移除了项目直接依赖的那个 Dep 包，依赖 A 里的那个版本的 Dep 就会被移动到项目的 node_modules 的第一层，此时也就恢复到默认扁平化目录结构。

通常来说 package-lock.json 是不会被发布到 npm 仓库的，如果想要确定依赖的版本号，要么是把版本号写死，要么是使用 npm-shrinkwrap.json，它和 package-lock.json 内容和作用是一样的，但是可以被发布到 npm 仓库。