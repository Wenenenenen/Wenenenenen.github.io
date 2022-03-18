---
title: 优雅地实现递归遍历树形结构
index_img: /img/index_post/tree.jpg
date: 2022-03-18 14:27:27
tags: [JAVA,递归遍历,stream流,lambda表达式]
categories: JAVA
excerpt: 使用java8，stream流和lambda表达式，递归遍历树形结构
---

#递归遍历树状结构优雅实现

##实体：

```java
@Data
@Builder
public class Menu {
  private Integer id;
  private String name;
  private Integer parentId;
  private List<Menu> childrenList;

  public Menu(Integer id, String name, Integer parentId) {
    this.id = id;
    this.name = name;
    this.parentId = parentId;
  }

  public Menu(Integer id, String name, Integer parentId, List<Menu> childrenList) {
    this.id = id;
    this.name = name;
    this.parentId = parentId;
    this.childrenList = childrenList;
  }
}
```

##实现：

```java
@SpringBootTest
public class RecursionTest {
  @Test
  public void recursion() {
    List<Menu> menus = Arrays.asList(
            new Menu(1, "根节点", 0),
            new Menu(2, "子节点1", 1),
            new Menu(3, "子节点1.1", 2),
            new Menu(4, "子节点1.2", 2),
            new Menu(5, "根节点1.3", 2),
            new Menu(6, "根节点2", 1),
            new Menu(7, "根节点2.1", 6),
            new Menu(8, "根节点2.2", 6),
            new Menu(9, "根节点2.2.1", 7),
            new Menu(10, "根节点2.2.2", 7),
            new Menu(11, "根节点3", 1),
            new Menu(12, "根节点3.1", 11));

    // 获取父节点
    List<Menu> collect = menus.stream()
            .filter(m -> m.getParentId() == 0)
            .peek((m) -> m.setChildrenList(getChildren(m, menus)))
            .collect(Collectors.toList());
    System.out.println(JSONUtil.toJsonStr(collect));
  }

  private List<Menu> getChildren(Menu root, List<Menu> all) {
    return all.stream()
            .filter(m -> Objects.equals(m.getParentId(), root.getId()))
            .peek(m -> m.setChildrenList(getChildren(m, all)))
            .collect(Collectors.toList());
  }

}
```
