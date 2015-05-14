title: �����й��ߣ�find
date: 2015-05-14 14:15:48
categories: [����]
tags: [����, find]
description: �����й���, find, grep, recursive search
---

find���ߵļ��÷���

<!-- more -->

# ͳ�ƴ�������

```bash
find . -type f -name "*.java" -or -name "*.xml" | xargs cat | wc -l
# for GoW on Windows
find . -type f -name "*.java" -or -name "*.xml" | tr \r\n ' ' | xargs cat | wc -l

# ����˵����
-type f # �ļ�����ֻ��עfile
-name "*.java" -or -name "*.xml" # ֻ��עjava��xml�ļ�
xargs cat # ��ӡ���ļ�����
wc -l # ͳ������
tr \r\n ' ' # ��\r\nת��Ϊ�ո񣬷���xargs cat���޷������ָ��ļ����������Ҳ����ļ�
```

# �ݹ���������Ŀ¼

```bash
find . -type f -name "*.java" -exec grep -i "onCreate" {} \;

# ����˵��
-type f -name "*.java" # �ҵ����е�java�ļ�
-exec grep -inH "onCreate" {} \; # ÿ�ҵ�һ����ִ��grep���{}�����ҵ����ļ����ļ����� \; ��ֹ�ֺű�ת��
                               # Windows�ϲ���Ҫ�ֺ�ת��
                               # -i: ���Դ�Сд�� -n: ����кţ� -H: ����ļ���
```