---
layout: post
title: Golang HTTP2ʵ�����ڴ�Ĳ�����ʹ��
---
������ڵ���Ŀ��Ҫ�õ�HTTP2������golang��ߣ��һ��������һЩ���ܲ��ԣ�GET�����ܻ��ã���POST���ύ��ʱ�����ڴ������ǣ�h2load 10000������100��stream per conn�£�16G�ڴ�ûһ��ͺĸ��ˡ�

��Ϊ�պ��ṩ��HTTP��������ֱ����[net/http/pprof](https://golang.org/pkg/net/http/pprof/)ץheap��������������Դ�룬���־�Ȼ����request body������£����������ķ���64KB��buf������һ��ı���˵ʵ����̫�˷��ˡ�

�ȼ򵥰����ֵ��С��1K�������ڴ�ռ��ȷʵ�д�����½�����request body����1KB�Ϳ�ʼ��RST�ˡ�����ϸ���˵��������`Content-Length`��С��64K��ֱ��ʹ��������������ԭ�����߼���

�����С�Ķ����˸�[PR](https://github.com/golang/go/issues/18509)��������golang�Ǳߵķ�Ӧ������һ�����õ�ʵ�֣�ֻ���ٵȵȿ��ˡ�