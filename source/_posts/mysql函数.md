---
title: mysql函数大全
date: 2020-04-03 17:13:26
tags:
  - mysql
  - 函数
categories: 数据库
description: 使用数据库版本：mysql 8.x
---

# 数据准备

## 表结构

### 班级 t_class

| 字段名     | 类型        | 注释     |
| ---------- | ----------- | -------- |
| class_id   | int         | 班级主键 |
| class_name | varchar(32) | 班级名称 |

### 学生表 t_student

| 字段名       | 类型        | 注释         |
| ------------ | ----------- | ------------ |
| student_id   | int         | 学生主键     |
| student_name | varchar(32) | 学生姓名     |
| class_id     | int         | 所在班级主键 |

### 科目 t_subject

| 字段名       | 类型        | 注释     |
| ------------ | ----------- | -------- |
| subject_id   | int         | 科目主键 |
| subject_name | varchar(32) | 科目姓名 |

### 成绩 t_score

| 字段名        | 类型 | 注释     |
| ------------- | ---- | -------- |
| subject_id    | int  | 科目主键 |
| student_id    | int  | 学生主键 |
| subject_score | int  | 成绩     |

## Sql

```

-- ----------------------------
-- Table structure for t_class
-- ----------------------------
DROP TABLE IF EXISTS `t_class`;
CREATE TABLE `t_class`  (
  `class_id` int(11) NOT NULL COMMENT '班级主键',
  `class_name` varchar(32) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL COMMENT '班级名称',
  PRIMARY KEY (`class_id`) USING BTREE
) ENGINE = InnoDB CHARACTER SET = utf8mb4 COLLATE = utf8mb4_general_ci ROW_FORMAT = Dynamic;

-- ----------------------------
-- Records of t_class
-- ----------------------------
INSERT INTO `t_class` VALUES (1, '1班');
INSERT INTO `t_class` VALUES (2, '2班');
INSERT INTO `t_class` VALUES (3, '3班');

-- ----------------------------
-- Table structure for t_score
-- ----------------------------
DROP TABLE IF EXISTS `t_score`;
CREATE TABLE `t_score`  (
  `subject_id` int(11) NULL DEFAULT NULL COMMENT '科目主键',
  `student_id` int(11) NULL DEFAULT NULL COMMENT '学生主键',
  `subject_score` int(11) NULL DEFAULT NULL COMMENT '成绩'
) ENGINE = InnoDB CHARACTER SET = utf8mb4 COLLATE = utf8mb4_general_ci ROW_FORMAT = Dynamic;

-- ----------------------------
-- Records of t_score
-- ----------------------------
INSERT INTO `t_score` VALUES (1, 1, 67);
INSERT INTO `t_score` VALUES (1, 2, 88);
INSERT INTO `t_score` VALUES (1, 3, 69);
INSERT INTO `t_score` VALUES (1, 4, 89);
INSERT INTO `t_score` VALUES (1, 5, 92);
INSERT INTO `t_score` VALUES (1, 6, 69);
INSERT INTO `t_score` VALUES (1, 8, 75);
INSERT INTO `t_score` VALUES (1, 9, 72);
INSERT INTO `t_score` VALUES (1, 10, 78);
INSERT INTO `t_score` VALUES (2, 1, 84);
INSERT INTO `t_score` VALUES (2, 2, 96);
INSERT INTO `t_score` VALUES (2, 3, 76);
INSERT INTO `t_score` VALUES (2, 4, 89);
INSERT INTO `t_score` VALUES (2, 5, 90);
INSERT INTO `t_score` VALUES (2, 6, 81);
INSERT INTO `t_score` VALUES (2, 8, 76);
INSERT INTO `t_score` VALUES (2, 9, 77);
INSERT INTO `t_score` VALUES (2, 10, 79);

-- ----------------------------
-- Table structure for t_student
-- ----------------------------
DROP TABLE IF EXISTS `t_student`;
CREATE TABLE `t_student`  (
  `student_id` int(11) NOT NULL COMMENT '学生主键',
  `student_name` varchar(32) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL COMMENT '学生姓名',
  `class_id` int(11) NULL DEFAULT NULL COMMENT '所在班级主键',
  PRIMARY KEY (`student_id`) USING BTREE
) ENGINE = InnoDB CHARACTER SET = utf8mb4 COLLATE = utf8mb4_general_ci ROW_FORMAT = Dynamic;

-- ----------------------------
-- Records of t_student
-- ----------------------------
INSERT INTO `t_student` VALUES (1, '张一', 1);
INSERT INTO `t_student` VALUES (2, '李二', 1);
INSERT INTO `t_student` VALUES (3, '牛三', 1);
INSERT INTO `t_student` VALUES (4, '马四', 2);
INSERT INTO `t_student` VALUES (5, '刘六', 2);
INSERT INTO `t_student` VALUES (6, '孙七', 2);
INSERT INTO `t_student` VALUES (8, '郝八', 2);
INSERT INTO `t_student` VALUES (9, '李九', 3);
INSERT INTO `t_student` VALUES (10, '郭十', 3);

-- ----------------------------
-- Table structure for t_subject
-- ----------------------------
DROP TABLE IF EXISTS `t_subject`;
CREATE TABLE `t_subject`  (
  `subject_id` int(11) NOT NULL COMMENT '科目主键',
  `subject_name` varchar(32) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT NULL COMMENT '科目姓名',
  PRIMARY KEY (`subject_id`) USING BTREE
) ENGINE = InnoDB CHARACTER SET = utf8mb4 COLLATE = utf8mb4_general_ci ROW_FORMAT = Dynamic;

-- ----------------------------
-- Records of t_subject
-- ----------------------------
INSERT INTO `t_subject` VALUES (1, '语文');
INSERT INTO `t_subject` VALUES (2, '计算机');
```
