# maven parent pom

各个parent pom之间的依赖关系如下：

```mermaid
graph TD;
    pure-->parent;
    base-->pure;
    assembly-->base;
    mode-->assembly;
    docker-->mode;
    boot-->docker;
    obscure-->mode;
```