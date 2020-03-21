
## 关于docsify的配置

author: [Myriad-Dreamin](https://github.com/Myriad-Dreamin)

---

#### 在本地运行docsify

先安装`docsify`:

```bash
yarn global add docsify-cli
# or
npm install -g docsify-cli
```

在`qemu-book`下运行`docsify serve .`

#### 在Sidebar中添加一个项目

在根目录下编辑`_sidevar.md`

#### TeX公式支持

```markdown
CPU由$ \text{TB}_i $ 块转向 $ \text{TB}_{i+1} $
```

CPU由$ \text{TB}_i $ 块转向 $ \text{TB}_{i+1} $

#### 内嵌高亮语言支持

`docsify`的prism渲染模块是按需加载的，例如我们需要渲染c++代码，那么在`index.html`中添加一行
```html
<!-- prism.js  -->
已添加的模块...
<!-- c++是类C语言，所以依赖c.min.js -->
<script src="//unpkg.com/prismjs/components/prism-c.min.js"></script>
<script src="//unpkg.com/prismjs/components/prism-cpp.min.js"></script>
```

然后在正文中这样书写
````markdown
​```cpp
#include <cstdio>

int test() {

}

int main() {
    test();
}
```
````

​```cpp
#include <cstdio>

int test() {

}

int main() {
    test();
}
```

#### 流程图支持

````markdown
​```mermaid
sequenceDiagram
    张三->>李四: 吃了吗？
    李四->>张三: 吃了
```
````

​```mermaid
sequenceDiagram
        张三->>李四: 吃了吗？
        李四->>张三: 吃了
```