---
title: BusTub Lab0 cpp primer
date: 2023-03-18T00:28:50Z
lastmod: 2023-03-19T15:57:50Z
categories: [Database,BusTub]
---

# BusTub Lab0 cpp primer

实现一个字典树，主要定义了三个类：

* TrieNode
* TrieNodeWithValue
* Trie

## Task1

实现一个不支持并发的字典树

### TrieNode

* 内部存储的数据为一个char字符，
* 并且有一个标识位置来表示是否到达了结束位置
* 对于字节点通过一个char-> unique_ptr的Map来进行存储

使用unique_ptr则意味着当将其分配给其他的变量时需要小心，`InsertChildNode`​ `GetChildNode`​的返回格式均为`unique_ptr`​，因此可以不进行copy的直接访问当中的数据

通过移动构造函数来进行赋值，传输数据至一个新的节点上，确保没有对unique_ptr进行拷贝。

### TrieNodeWithValue

继承自TrieNode，代表最终节点，key为字符串的最后一个字符，is_end永远为true

根据情况不同调用不同的构造函数：

* (char key_char,T value)用于根据给定的kv创建一个节点
* (TrieNode &&trieNode,T value)获取传入节点的独占指针，并且将给定值设置到自身

### Trie

定义了整个字典树，其中定义一个root node作为所有节点的根节点

**Insert**​

插入一个key时需要首先遍历整个树，如果不存在再插入节点，不允许插入重复的节点，重复则返回false

遍历完之后有三种可能：

* 节点不存在，调用TrieNodeWithValue(char key_char, T value)构造函数去插入一个新的节点，并且对TrieNode使用独占指针也可以对存储一个指向TrieNodeWithValue的独占指针？？
* 节点存在，但是不是最终节点，需要调用`TrieNodeWithValue(TrieNode &&trieNode, T value)`​构造函数去将旧的`TrieNode`​转换为新的`TrieNodeWithValue`​
* 节点存在但是已经是最终节点，return false即可

**Remove**​

* 遍历寻找给定的key，如果不存在则立即返回
* 如果找到了则将is_end_设置为false
* 如果该节点没有任何的子节点，直接将其从父节点的map当中移除
* 遍历尝试并递归地删除没有子节点的节点。遇到有子节点时停止

**GetValue**​

如果类型不匹配或者没找到key，则将success设置为false，

‍
