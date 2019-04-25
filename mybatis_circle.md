<!-- TOC -->

- [分享简介](#分享简介)
    - [1 问题描述](#1-问题描述)
        - [1.1 问题代码](#11-问题代码)
        - [1.2 问题描述](#12-问题描述)
    - [3 问题概览](#3-问题概览)
        - [3.1 引用的问题](#31-引用的问题)
        - [3.2 数据变更](#32-数据变更)
    - [4 问题分析](#4-问题分析)
        - [4.1 引用的问题](#41-引用的问题)
        - [4.2 数据变更的问题](#42-数据变更的问题)
    - [5 解决方案](#5-解决方案)
        - [5.1 引用的问题](#51-引用的问题)
        - [5.1 缓存的问题](#51-缓存的问题)
    - [6 扩展阅读](#6-扩展阅读)
        - [6.1 Mybatis的缓存机制](#61-mybatis的缓存机制)

<!-- /TOC -->

# 分享简介

当前的分享是对于在权限模块出现生成的权限结果树数据循环引用的情况

## 1 问题描述

### 1.1 问题代码

<pre>
    private Object getMenuAuth(Set<Long> userAuthSet) {
        List<MenuTreeNode> authMenuTree = null;
        if (userAuthSet != null && userAuthSet.size() > 0) {
            List<MenuTreeNode> authMenu = menuDao.getMenuSubset(userAuthSet);
            authMenuTree = TreeNodeUtils.createMenuTree(authMenu);
        }
        return authMenuTree;
    }

    public static List<MenuTreeNode> createMenuTree(List<MenuTreeNode> nodeList) {
        //无父子关系的节点整理出父子关系
        List<MenuTreeNode> resultList = new ArrayList<MenuTreeNode>();
        for (MenuTreeNode node1 : nodeList) {
            boolean rootMark = false;//根节点标志
            //根节点放到类目下
            if (StringUtils.isEmpty(node1.getParentCode()) && !node1.getCode().equals("!")) {
                node1.setParentCode("!");
            }
            for (MenuTreeNode node2 : nodeList) {
                if (node1.getParentCode() != null && node1.getParentCode().equals(node2.getCode())) {
                    rootMark = true;
                    if (node2.getChildren() == null) {
                        node2.setChildren(new ArrayList<MenuTreeNode>());
                    }
                    node2.getChildren().add(node1);
                    break;
                }
            }
            if (!rootMark) {
                resultList.add(node1);
            }
        }
        return resultList;
    }
</pre>

### 1.2 问题描述

* 第一次输入

<pre>
[
    {
        "code":"1",
        "label":"1"
    },
    {
        "code":"2",
        "label":"1-2",
        "parentCode":"1"
    },
    {
        "code":"3",
        "label":"1-3",
        "parentCode":"1"
    },
    {
        "code":"4",
        "label":"1",
        "parentCode":"1"
    },
    {
        "code":"5",
        "label":"1",
        "parentCode":"4"
    }
]
</pre>

* 第一次输出

<pre>
[
    {
        "children":[
            {
                "code":"2",
                "label":"1-2",
                "parentCode":"1"
            },
            {
                "code":"3",
                "label":"1-3",
                "parentCode":"1"
            },
            {
                "children":[
                    {
                        "code":"5",
                        "label":"1",
                        "parentCode":"4"
                    }
                ],
                "code":"4",
                "label":"1",
                "parentCode":"1"
            }
        ],
        "code":"1",
        "label":"1",
        "parentCode":"!"
    }
]
</pre>

* 第二次输入

<pre>
[
    {
        "children":[
            {
                "code":"2",
                "label":"1-2",
                "parentCode":"1"
            },
            {
                "code":"3",
                "label":"1-3",
                "parentCode":"1"
            },
            {
                "children":[
                    {
                        "code":"5",
                        "label":"1",
                        "parentCode":"4"
                    }
                ],
                "code":"4",
                "label":"1",
                "parentCode":"1"
            }
        ],
        "code":"1",
        "label":"1",
        "parentCode":"!"
    },
    {
        "$ref":"$[0].children[0]"
    },
    {
        "$ref":"$[0].children[1]"
    },
    {
        "$ref":"$[0].children[2]"
    },
    {
        "$ref":"$[0].children[2].children[0]"
    }
]
</pre>

* 第二次输出

<pre>
[
    {
        "children":[
            {
                "code":"2",
                "label":"1-2",
                "parentCode":"1"
            },
            {
                "code":"3",
                "label":"1-3",
                "parentCode":"1"
            },
            {
                "children":[
                    {
                        "code":"5",
                        "label":"1",
                        "parentCode":"4"
                    },
                    {
                        "$ref":"$[0].children[2].children[0]"
                    }
                ],
                "code":"4",
                "label":"1",
                "parentCode":"1"
            },
            {
                "$ref":"$[0].children[0]"
            },
            {
                "$ref":"$[0].children[1]"
            },
            {
                "$ref":"$[0].children[2]"
            }
        ],
        "code":"1",
        "label":"1",
        "parentCode":"!"
    }
]
</pre>

* ……

## 3 问题概览

### 3.1 引用的问题

### 3.2 数据变更

在代码中的List每次都是New出来的，但是数据却变更了

## 4 问题分析

### 4.1 引用的问题

* 症结所在

<pre>
for (MenuTreeNode node1 : nodeList) {
    boolean rootMark = false;//根节点标志
    //根节点放到类目下
    if (StringUtils.isEmpty(node1.getParentCode()) && !node1.getCode().equals("!")) {
        node1.setParentCode("!");
    }
    for (MenuTreeNode node2 : nodeList) {
        if (node1.getParentCode() != null && node1.getParentCode().equals(node2.getCode())) {
            rootMark = true;
            if (node2.getChildren() == null) {
                node2.setChildren(new ArrayList<MenuTreeNode>());
            }
            node2.getChildren().add(node1);
            break;
        }
    }
    if (!rootMark) {
        resultList.add(node1);
    }
}
</pre>

* 讲解

> 这里的代码是两层循环，并且两层循环的主体都是nodeList，在代码中我们可以发现以下语句
<pre>
if (node2.getChildren() == null) {
    node2.setChildren(new ArrayList<MenuTreeNode>());
}
node2.getChildren().add(node1);
break;
</pre>
> 这段代码在符合条件(children为空)的情况下，新增了一个节点进去，这个节点是node1,同时node1其实也是nodeList中的元素，于是导致了对象引用的情况，而且如果对于同一个List来说，循环的次数越多，那么引用的情况越严重(最后的结果是890条数据生成数据树，占用的空间能够达到20M+)

### 4.2 数据变更的问题

* 症结所在

<pre>
private Object getMenuAuth(Set<Long> userAuthSet) {
    List<MenuTreeNode> authMenuTree = null;
    if (userAuthSet != null && userAuthSet.size() > 0) {
        List<MenuTreeNode> authMenu = menuDao.getMenuSubset(userAuthSet);
        authMenuTree = TreeNodeUtils.createMenuTree(authMenu);
    }
    return authMenuTree;
}
</pre>

* 讲解

<pre>
传入createMenuTree的参数为authMenu，这个参数每次都是由数据库返回的，那么照理说应该是每次都是一个新的列表，而且每次都是重新调用了 getMenuAuth这个方法， 不应该出现这个问题

百思不得骑姐

通过对处理的线程以及对象的hashCode分析
处理线程为单线程
虽然每次都重新调用了，但是每次处理的对象的hashCode是一样的

那么我们可以肯定List中的对象引用是没有变更的，但是为什么数据却变更了呢?

有没有想起来上一个问题，没错，就是上一个问题导致的引用指向的堆中的数据变更掉了

现在就有另外一个问题，引用为什么没有变更呢？

幕后大Boss终于出现了，Mybatis的缓存导致的
</pre>

## 5 解决方案

### 5.1 引用的问题

* 将List修改为Tree结构的算法重写

<pre>
/**
*功能描述 将非树形结构的菜单整理成树形结构的菜单
* 实现说明:
* 1.首先对传入的参数进行判空的逻辑
* 2.如果不为空，则遍历传入的List
* 3.使用map存储已经加入到树形结构的引用
* 4.对于新循环的数据，如果已经加入到树形结构，那么跳过不处理
* 5.对于新循环的数据，如果没有加入到树形结果，如果其parentCode已经加入到树形结构，那么新建一条数据放入到其parent的child List中，同时记录其引用
* 6.对于新循环的数据，如果没有加入到树形结果，如果其parentCode没有加入到树形结构，那么新建一条数据放入到结果集中，同时记录其引用
* 7.循环执行完成，则树形结构生成
* <p> author liyong </p>
* <p> date 2019-04-18 </p>
* <p> param   </p>
* <p> return java.util.List<com.bwton.dist.util.MenuTreeNode> </p>
*/
public static List<MenuTreeNode> createMenuTree(List<MenuTreeNode> list) {
ArrayList<MenuTreeNode> result = new ArrayList<>();
//无父子关系的节点整理出父子关系
if (!CollectionUtils.isEmpty(list)) {
    List<String> alreadyInIds = new ArrayList<>();
    Map<String, MenuTreeNode> alreadyInMap = new HashMap<>();
    for (MenuTreeNode node : list) {
        if (!alreadyInIds.contains(node.getCode())) {
            //生成一个MenuTreeNode，放入到Result对应的对象的List中
            MenuTreeNode newNode = new MenuTreeNode(node.getCode(), node.getLabel(), node.getParentCode(), new ArrayList<MenuTreeNode>(), node.getSortNo(), node.getIcon(), node.getUrl());
            if (alreadyInMap.containsKey(newNode.getParentCode())) {
                //如果当前存在父节点则添加到父节点的List中
                if (alreadyInMap.get(newNode.getParentCode()).getChildren() == null) {
                    alreadyInMap.get(newNode.getParentCode()).setChildren(new ArrayList<>());
                }
                alreadyInMap.get(newNode.getParentCode()).getChildren().add(newNode);
            } else {
                //创建新节点，写入到顶级节点
                result.add(newNode);
            }
            alreadyInIds.add(node.getCode());
            alreadyInMap.put(node.getCode(), newNode);
        }
    }
}
return result;
}
</pre>

### 5.1 缓存的问题

* 缓存本来是没有问题的，而且有利于提高性能，问题都出在开发人员身上

## 6 扩展阅读

### 6.1 Mybatis的缓存机制

* mybatis的缓存分为一级缓存和二级缓存，一级缓存是SqlSession级别的，二级缓存是Mapper级别的

* 一级缓存

<pre>
每次对数据库的操作都是一个一个新的SqlSession，Session缓存对象是存储在Map中的
执行流程:
1、第一次查询数据之后，会将数据缓存起来，[namespace:sql:参数]作为key
2、第二次查询如果参数一样，那么直接从缓存中获取数据
</pre>

* 二级缓存

<pre>
二级缓存是mapper级别的缓存，也就是同一个namespace的mappe.xml，当多个SqlSession使用同一个Mapper操作数据库的时候，得到的数据会缓存在同一个二级缓存区域
二级缓存默认是没有开启的。需要在setting全局参数中配置开启二级缓存

执行流程:
1. 第一次执行缓存数据，[namespace:sql:参数]作为key
2. 第二次执行先到一级缓存查询，然后去二级缓存查询，如果查询到则直接返回，否则查数据库
</pre>